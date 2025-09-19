# TiDB Storage Engine Integration

TiDB's storage layer provides a unified interface to heterogeneous storage engines, enabling Hybrid Transactional/Analytical Processing (HTAP) capabilities. This document explains how TiDB integrates with TiKV (row-based OLTP storage) and TiFlash (columnar OLAP storage).

## 🏗️ Storage Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                TiDB SQL Layer                       │
└─────────────────────┬───────────────────────────────┘
                      │ Unified Storage Interface
┌─────────────────────▼───────────────────────────────┐
│              Storage Abstraction Layer              │
│  ┌─────────────────────────────────────────────────┐ │
│  │          Client Interface (pkg/kv/)            │ │
│  └─────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────┐ │
│  │      Request Builder (pkg/distsql/)            │ │
│  └─────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────┐ │
│  │     Table Codec (pkg/tablecodec/)              │ │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────┘
        ┌─────────────▼─────────────┐
        │    Storage Engines        │
        │  ┌─────────┐ ┌──────────┐ │
        │  │  TiKV   │ │ TiFlash  │ │
        │  │(Row-based│ │(Columnar)│ │
        │  │  OLTP)  │ │  OLAP)   │ │
        │  └─────────┘ └──────────┘ │
        └───────────────────────────┘
```

## 🔧 Storage Interface Layer

### Core Storage Interface (`pkg/kv/`)

The foundation of TiDB's storage abstraction is the `Storage` interface:

```go
type Storage interface {
    // Transaction management
    Begin(opts ...TxnOption) (Transaction, error)
    BeginWithStartTS(startTS uint64) (Transaction, error)
    GetSnapshot(ver Version) Snapshot
    
    // Client access
    GetClient() Client
    GetMPPClient() MPPClient
    
    // Resource management
    GetMemCache() MemManager
    Close() error
    
    // Configuration
    Name() string
    Describe() string
}
```

### Store Types and Routing

TiDB differentiates between storage engines using store types:

```go
type StoreType uint8

const (
    TiKV StoreType = iota    // Row-based transactional storage
    TiFlash                  // Columnar analytical storage  
    TiDB                     // Local memory storage
)

// Request routing based on store type
type Request struct {
    Tp          int64
    StartKey    Key
    EndKey      Key
    Data        []byte
    StoreType   StoreType     // Determines target storage engine
    Concurrency int
}
```

### Client Interface Abstraction

The unified client interface enables storage-agnostic operations:

```go
type Client interface {
    // Send request to appropriate storage engine
    Send(ctx context.Context, req *Request, vars any, option *ClientSendOption) Response
    
    // Capability checking
    IsRequestTypeSupported(reqType, subType int64) bool
}

// Storage-specific clients
type TiKVClient interface {
    Client
    SendReqCtx(ctx context.Context, req *Request, regionID RegionID) (*Response, error)
}

type MPPClient interface {
    // TiFlash MPP (Massively Parallel Processing) operations
    ConstructMPPTasks(context.Context, *MPPBuildTasksRequest, time.Duration) ([]MPPTaskMeta, error)
    DispatchMPPTask(DispatchMPPTaskParam) (*mpp.DispatchTaskResponse, bool, error)
    EstablishMPPConns(EstablishMPPConnsParam) (*tikvrpc.MPPStreamResponse, bool, error)
}
```

## 🗃️ Data Encoding and Organization

### Key-Value Mapping (`pkg/tablecodec/`)

TiDB maps relational data to key-value pairs using a structured encoding scheme:

#### Key Encoding Structure

```go
// Key prefixes for different data types
var (
    tablePrefix     = []byte{'t'}      // Table data: t{tableID}_r{handle}
    recordPrefixSep = []byte("_r")     // Record separator
    indexPrefixSep  = []byte("_i")     // Index separator: t{tableID}_i{indexID}
    metaPrefix      = []byte{'m'}      // Metadata: m{metaKey}
)
```

#### Row Data Encoding

```go
// Encode table row key
func EncodeRowKeyWithHandle(tableID int64, handle kv.Handle) kv.Key {
    buf := make([]byte, 0, prefixLen+len(handle.Encoded()))
    buf = appendTableRecordPrefix(buf, tableID)  // t{tableID}_r
    buf = append(buf, handle.Encoded()...)       // append handle
    return buf
}

// Example: Table 100, Handle 1 → t100_r1
```

#### Index Data Encoding

```go
// Encode index key
func EncodeIndexSeekKey(tableID int64, idxID int64, encodedValue []byte) kv.Key {
    key := make([]byte, 0, RecordRowKeyLen+len(encodedValue))
    key = appendTableIndexPrefix(key, tableID)    // t{tableID}_i
    key = codec.EncodeInt(key, idxID)            // append index ID
    key = append(key, encodedValue...)           // append index values
    return key
}

// Example: Table 100, Index 2, Value "John" → t100_i2"John"
```

#### Value Encoding

```go
// Row value encoding with column families
func EncodeRow(row []types.Datum, colIDs []int64, loc *time.Location) ([]byte, error) {
    if len(row) != len(colIDs) {
        return nil, errors.Errorf("EncodeRow error: data and columnID count not match %d vs %d", len(row), len(colIDs))
    }
    
    values := make([]byte, 0, defaultValuesLen)
    for i, col := range row {
        value, err := tablecodec.EncodeValue(loc, col)
        if err != nil {
            return nil, err
        }
        values = append(values, value...)
    }
    return values, nil
}
```

### Schema Metadata Storage (`pkg/meta/`)

Schema information is stored using a hierarchical key structure:

```go
// Metadata key structure:
// NextGlobalID -> int64                    Global ID allocation
// SchemaVersion -> int64                   Schema version for DDL
// DBs -> database list                     Database catalog
// DB:{dbID} -> database metadata           Per-database info  
// DB:{dbID}:Table:{tableID} -> table meta  Table definitions

func encodeDBKey(dbID int64) []byte {
    return []byte(fmt.Sprintf("%s:%d", mDBPrefix, dbID))
}

func encodeTableKey(dbID, tableID int64) []byte {
    return []byte(fmt.Sprintf("%s:%d:%s:%d", mDBPrefix, dbID, mTablePrefix, tableID))
}
```

## 🚀 TiKV Integration (OLTP Storage)

### Connection Management (`pkg/store/driver/`)

```go
type tikvStore struct {
    *tikv.KVStore                    // TiKV client-go integration
    etcdAddrs     []string           // PD endpoints
    tlsConfig     *tls.Config        // TLS configuration
    memCache      kv.MemManager      // Memory cache
    gcWorker      *gcworker.GCWorker // Garbage collection
    coprStore     *copr.Store        // Coprocessor operations
    codec         tikv.Codec         // Key-value codec
}
```

### Transaction Management

TiDB coordinates ACID transactions across TiKV using Two-Phase Commit:

```go
type Transaction interface {
    // CRUD operations
    Get(ctx context.Context, k Key) ([]byte, error)
    Set(k Key, v []byte) error
    Delete(k Key) error
    
    // Batch operations
    Iter(k Key, upperBound Key) (Iterator, error)
    BatchGet(ctx context.Context, keys []Key) (map[string][]byte, error)
    
    // Transaction lifecycle  
    Commit(ctx context.Context) error
    Rollback() error
    
    // Properties
    StartTS() uint64
    GetCommitTS() uint64
    IsReadOnly() bool
}
```

### Coprocessor Integration

TiKV coprocessors enable computation pushdown:

```go
type CopRequest struct {
    Tp          int64              // Request type (selection, aggregation, etc.)
    Data        []byte             // Encoded execution plan
    Ranges      []KeyRange         // Key ranges to process
    StartTs     uint64             // MVCC timestamp
    IsolationLevel IsoLevel        // Isolation level
}

// Send coprocessor request to TiKV
func (c *CopClient) Send(ctx context.Context, req *Request) *CopResponse {
    // 1. Build coprocessor request
    copReq := buildCopRequest(req)
    
    // 2. Send to appropriate TiKV regions
    resp := c.sendToRegions(ctx, copReq, req.KeyRanges)
    
    // 3. Collect and merge results
    return c.handleResponse(resp)
}
```

### Key Features

- **Region-based sharding**: Automatic data partitioning
- **Raft replication**: Strong consistency with 3-way replication
- **Distributed transactions**: ACID properties across multiple nodes
- **Coprocessor pushdown**: Reduce network traffic via computation pushdown

## 📊 TiFlash Integration (OLAP Storage)

### MPP (Massively Parallel Processing)

TiFlash provides columnar storage with MPP capabilities:

```go
type MPPTaskMeta struct {
    StartTS    uint64              // Transaction start timestamp
    TaskID     int64               // Task identifier
    Address    string              // TiFlash node address
    Root       bool                // Root task flag
}

// Construct MPP execution plan
func (c *mppClient) ConstructMPPTasks(
    ctx context.Context, 
    req *kv.MPPBuildTasksRequest,
    ttl time.Duration,
) ([]kv.MPPTaskMeta, error) {
    // 1. Query TiFlash topology
    stores := c.getStores(ctx, req.KeyRanges)
    
    // 2. Build execution tasks
    tasks := c.buildMPPTasks(req.PlanTree, stores)
    
    // 3. Establish task dependencies
    return c.linkTaskDependencies(tasks), nil
}
```

### Data Replication

TiFlash receives real-time data from TiKV via Raft learner protocol:

```go
// TiFlash replica configuration
type TiFlashReplicaSpec struct {
    Count             uint64    // Number of replicas
    LocationLabels    []string  // Placement constraints
    AvailableZones    []string  // Available zones
}

// Query routing to TiFlash
func (c *copClient) SendToTiFlash(ctx context.Context, req *Request) Response {
    // Check TiFlash replica availability
    if !c.checkTiFlashReplica(req.KeyRanges) {
        return c.fallbackToTiKV(ctx, req)
    }
    
    // Route to TiFlash for analytical processing
    return c.sendMPPRequest(ctx, req)
}
```

### Query Execution Flow

```go
// TiFlash MPP execution pipeline
1. SQL → TiDB Planner → MPP Plan
2. MPP Plan → Task Distribution → TiFlash Nodes
3. TiFlash Nodes → Parallel Execution → Partial Results  
4. Partial Results → Root Task → Final Result
5. Final Result → TiDB → Client
```

## 🔄 Distributed SQL Processing (`pkg/distsql/`)

### Request Building

The `RequestBuilder` creates storage requests optimized for different engines:

```go
type RequestBuilder struct {
    kv.Request
    is   infoschema.MetaOnlyInfoSchema  // Schema information
    err  error                          // Build errors
    dag  *tipb.DAGRequest              // Execution plan
}

func (builder *RequestBuilder) Build() (*kv.Request, error) {
    // 1. Determine target storage engine
    storeType := builder.determineStoreType()
    
    // 2. Build appropriate request format
    switch storeType {
    case kv.TiKV:
        return builder.buildTiKVRequest()
    case kv.TiFlash:
        return builder.buildTiFlashRequest()
    }
    
    return &builder.Request, builder.err
}
```

### Smart Routing

TiDB automatically routes queries to optimal storage engines:

```go
func (builder *RequestBuilder) SetFromSessionVars(sv *variable.SessionVars) *RequestBuilder {
    // Route to TiFlash for analytical queries
    if sv.AllowMPPExecution && builder.isAnalyticalQuery() {
        builder.Request.StoreType = kv.TiFlash
        builder.Request.MPPTask = true
    } else {
        // Route to TiKV for transactional queries
        builder.Request.StoreType = kv.TiKV
    }
    return builder
}
```

### Result Handling

Unified result processing for both storage engines:

```go
type SelectResult interface {
    // Iterator interface for row-based results
    Next(ctx context.Context) ([][]byte, error)
    Close() error
    
    // Streaming interface for large results  
    Chunk() *chunk.Chunk
    GetPartialResult() *chunk.Chunk
}

// Process results from either TiKV or TiFlash
func (sr *selectResult) Next(ctx context.Context) ([][]byte, error) {
    switch sr.storeType {
    case kv.TiKV:
        return sr.processTiKVResult(ctx)
    case kv.TiFlash:
        return sr.processTiFlashResult(ctx)
    }
}
```

## 🎯 Performance Optimizations

### Predicate Pushdown

Both TiKV and TiFlash support computation pushdown:

```go
// Push selection conditions to storage layer
type Selection struct {
    Conditions []expression.Expression  // Filter conditions
    Children   []PhysicalPlan          // Child operators
}

func (s *Selection) ToPB() *tipb.Selection {
    return &tipb.Selection{
        Conditions: expression.ExpressionsToPBList(s.Conditions),
    }
}
```

### Batch Operations

Efficient bulk processing reduces network overhead:

```go
// Batch key-value operations
func (txn *tikvTxn) BatchGet(ctx context.Context, keys []kv.Key) (map[string][]byte, error) {
    // 1. Group keys by region
    regionsToKeys := txn.groupKeysByRegion(keys)
    
    // 2. Send parallel requests
    results := make(chan batchResult, len(regionsToKeys))
    for region, regionKeys := range regionsToKeys {
        go txn.batchGetFromRegion(ctx, region, regionKeys, results)
    }
    
    // 3. Collect and merge results
    return txn.mergeBatchResults(results)
}
```

### Connection Pooling

Efficient connection management for high concurrency:

```go
type connPool struct {
    mu    sync.RWMutex
    conns map[string]*grpc.ClientConn  // Address -> Connection
    opts  []grpc.DialOption            // Connection options
}

func (p *connPool) getConn(addr string) (*grpc.ClientConn, error) {
    p.mu.RLock()
    if conn, ok := p.conns[addr]; ok {
        p.mu.RUnlock()
        return conn, nil
    }
    p.mu.RUnlock()
    
    return p.createConn(addr)
}
```

## 🔧 Configuration and Tuning

### Storage Engine Selection

Configure which storage engine to use for different workloads:

```sql
-- Use TiFlash for analytical queries
SET SESSION tidb_allow_mpp = ON;
SET SESSION tidb_enforce_mpp = ON;

-- Use TiKV for transactional queries
SET SESSION tidb_allow_mpp = OFF;
```

### Performance Parameters

Key configuration parameters for storage optimization:

```go
// TiKV configuration
TiKVClient: {
    GrpcConnectionCount:    4,      // Connections per TiKV
    GrpcKeepAliveTime:      10,     // Keep-alive interval
    GrpcKeepAliveTimeout:   3,      // Keep-alive timeout
    CommitTimeout:          "41s",  // Transaction commit timeout
}

// TiFlash configuration  
TiFlashCompute: {
    MaxMemoryUsage:         0.8,    // Memory usage limit
    MaxThreads:            -1,      // Thread pool size
    MaxQueryMemoryUsage:    0,      // Per-query memory limit
}
```

## 🚀 Best Practices

### Query Optimization

1. **Use appropriate storage engine** for workload characteristics
2. **Leverage predicate pushdown** to reduce data transfer
3. **Design efficient indexes** for TiKV access patterns
4. **Consider TiFlash replicas** for frequently accessed analytical data

### Data Modeling

1. **Partition large tables** for better parallel processing
2. **Use clustered indexes** for range scan optimization
3. **Avoid hotspots** in key design
4. **Consider column families** for wide tables

### Monitoring and Debugging

1. **Monitor storage engine selection** via query plans
2. **Track coprocessor performance** metrics
3. **Analyze region distribution** for load balancing
4. **Monitor TiFlash replication lag** for consistency

## 📚 Related Documentation

- [Distributed Execution System](distributed-execution.md)
- [Transaction Management](transactions.md)
- [Performance Guidelines](../patterns/performance.md)
- [TiKV Documentation](https://tikv.org/docs/)
- [TiFlash Documentation](https://docs.pingcap.com/tidb/stable/tiflash-overview)