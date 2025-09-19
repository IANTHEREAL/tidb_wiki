# Distributed Execution System

TiDB's distributed execution system enables parallel query processing across multiple storage nodes, providing horizontal scalability and high performance for both OLTP and OLAP workloads. This document explores how TiDB coordinates distributed query execution.

## 🏗️ Distributed Execution Architecture

```
┌─────────────────────────────────────────────────────┐
│                 TiDB Coordinator                    │
│  ┌─────────────────────────────────────────────────┐ │
│  │           Query Planner                         │ │
│  │   ┌─────────────┐  ┌─────────────────────────┐  │ │
│  │   │ Logical Plan│  │   Physical Plan         │  │ │
│  │   │ Generation  │  │   Distribution          │  │ │
│  │   └─────────────┘  └─────────────────────────┘  │ │
│  └─────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────┐ │
│  │         Execution Coordinator                   │ │
│  │   ┌─────────────┐  ┌─────────────────────────┐  │ │
│  │   │Task Dispatch│  │   Result Aggregation    │  │ │
│  │   └─────────────┘  └─────────────────────────┘  │ │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────┘
                      │ gRPC/Coprocessor Protocol
            ┌─────────▼─────────┐
            │  Region Mapping   │
            │   & Load Balance  │
            └─────────┬─────────┘
      ┌───────────────┼───────────────┐
      ▼               ▼               ▼
┌──────────┐    ┌──────────┐    ┌──────────┐
│TiKV Node │    │TiKV Node │    │TiFlash   │
│Region 1-5│    │Region 6-10│   │MPP Engine│
└──────────┘    └──────────┘    └──────────┘
```

## 🔧 Core Components

### 1. Distributed Query Coordinator (`pkg/executor/`)

The coordinator manages distributed query execution across storage nodes:

```go
// Core coordinator interface
type DistSQLExecutor interface {
    executor.Executor
    BuildResp(ctx context.Context) error
}

// Base implementation for distributed execution
type baseDistSQLExecutor struct {
    executor.BaseExecutor
    
    startTS         uint64              // Transaction start timestamp
    kvRanges        []kv.KeyRange       // Key ranges to process
    dagPB           *tipb.DAGRequest    // Execution plan
    ctx             sessionctx.Context  // Session context
    concurrency     int                 // Parallel execution degree
    
    // Results handling
    resultChan      chan *distsql.SelectResult
    partialCount    int64
    memTracker      *memory.Tracker
}
```

#### Table Reader Execution

```go
type TableReaderExecutor struct {
    baseDistSQLExecutor
    
    table           table.Table         // Table metadata
    physicalTableID int64               // Physical table ID
    keepOrder       bool                // Maintain result order
    desc            bool                // Descending order
    streaming       bool                // Streaming mode
    feedback        *statistics.QueryFeedback
}

func (e *TableReaderExecutor) Open(ctx context.Context) error {
    // 1. Build distributed SQL request
    kvReq, err := e.buildKVReq(ctx)
    if err != nil {
        return err
    }
    
    // 2. Execute across storage nodes
    result, err := e.SelectResult(ctx, e.ctx, kvReq, e.retFieldTypes)
    if err != nil {
        return err
    }
    
    // 3. Set up result streaming
    e.resultChan = make(chan *distsql.SelectResult, 1)
    e.resultChan <- result
    
    return nil
}
```

### 2. Region-Based Execution (`pkg/distsql/`)

TiDB partitions queries across data regions for parallel processing:

```go
// Request distribution across regions
type RequestBuilder struct {
    kv.Request
    is          infoschema.MetaOnlyInfoSchema
    err         error
    dag         *tipb.DAGRequest
    executorID  string
}

func (builder *RequestBuilder) SetKeyRanges(keyRanges []kv.KeyRange) *RequestBuilder {
    // Partition key ranges across regions
    builder.KeyRanges = keyRanges
    
    // Determine optimal concurrency
    expectedConcurrency := len(keyRanges)
    if expectedConcurrency > builder.concurrency {
        builder.concurrency = expectedConcurrency
    }
    
    return builder
}
```

#### Coprocessor Request Distribution

```go
// Distribute coprocessor requests to storage nodes
func (w *copIteratorWorker) run(ctx context.Context) {
    defer func() {
        close(w.respChan)
    }()
    
    for task := range w.taskChan {
        // Build coprocessor request for specific region
        req := &kv.Request{
            Tp:            task.requestType,
            KeyRanges:     task.ranges,
            Data:          task.plan,
            Concurrency:   w.concurrency,
            StoreType:     task.storeType,
        }
        
        // Send to storage node
        resp := w.sendReq(ctx, req, task.region)
        
        // Forward response
        select {
        case w.respChan <- resp:
        case <-ctx.Done():
            return
        }
    }
}
```

### 3. MPP (Massively Parallel Processing) Engine

For analytical workloads, TiDB leverages TiFlash's MPP capabilities:

```go
// MPP task coordination
type MPPGather struct {
    executor.BaseExecutor
    
    tasks           []kv.MPPTaskMeta    // Distributed tasks
    mppReqs         []*kv.MPPDispatchRequest
    respIter        MPPResponseIterator
    memTracker      *memory.Tracker
    
    // Execution control
    canceled        uint32
    enableFastScan  bool
    needTriggerApi  bool
}

func (e *MPPGather) Open(ctx context.Context) error {
    // 1. Build MPP execution plan
    mppReqs, err := e.constructMPPRequests(ctx)
    if err != nil {
        return err
    }
    
    // 2. Dispatch tasks to TiFlash nodes
    respChan := make(chan *mpp.MPPDataPacket, 10)
    for _, req := range mppReqs {
        go e.dispatchMPPTask(ctx, req, respChan)
    }
    
    // 3. Set up result iterator
    e.respIter = newMPPResponseIterator(respChan)
    
    return nil
}
```

#### MPP Task Construction

```go
func (e *MPPGather) constructMPPRequests(ctx context.Context) ([]*kv.MPPDispatchRequest, error) {
    // 1. Get TiFlash node topology
    tiFlashStores, err := e.getTiFlashStores(ctx)
    if err != nil {
        return nil, err
    }
    
    // 2. Partition execution plan across nodes
    var mppReqs []*kv.MPPDispatchRequest
    for _, store := range tiFlashStores {
        req := &kv.MPPDispatchRequest{
            Data:      e.dagReq.Data,
            Meta:      e.buildTaskMeta(store),
            Timeout:   e.timeout,
            SchemaVar: e.mppQueryID.QueryID,
        }
        mppReqs = append(mppReqs, req)
    }
    
    return mppReqs, nil
}
```

### 4. Index Execution Strategies

TiDB implements various index access patterns for distributed execution:

#### Index Reader

```go
type IndexReaderExecutor struct {
    baseDistSQLExecutor
    
    index           *model.IndexInfo    // Index metadata
    physicalTableID int64               // Physical table ID
    outputColumns   []*model.ColumnInfo // Output columns
    ranges          []*ranger.Range     // Index scan ranges
    
    // Execution properties
    keepOrder       bool                // Maintain sort order
    desc           bool                 // Descending scan
    corColInFilter bool                 // Correlated columns in filter
    corColInAccess bool                 // Correlated columns in access
}

func (e *IndexReaderExecutor) buildKVReq(ctx context.Context) (*kv.Request, error) {
    // 1. Convert ranges to key ranges
    keyRanges, err := e.buildKeyRanges(e.ranges)
    if err != nil {
        return nil, err
    }
    
    // 2. Build index scan DAG
    dagReq := &tipb.DAGRequest{
        Executors: []*tipb.Executor{
            e.buildIndexScanExecutor(),
        },
        Flags: e.getDAGFlag(),
    }
    
    // 3. Create distributed request
    return &kv.Request{
        Tp:          kv.ReqTypeIndex,
        KeyRanges:   keyRanges,
        Data:        dagReq.Marshal(),
        Concurrency: e.concurrency,
        StoreType:   kv.TiKV,
    }, nil
}
```

#### Index Merge

```go
type IndexMergeReaderExecutor struct {
    baseDistSQLExecutor
    
    partialPlans    [][]executor.Executor // Partial execution plans
    tblPlans        []executor.Executor   // Table lookup plans
    indexes         []*model.IndexInfo    // Multiple indexes
    
    // Merge control
    pushedDownCond  expression.CNFExprs   // Pushed down conditions
    tblPushedDownCond expression.CNFExprs // Table pushed down conditions
    
    // Runtime state
    partialWorkerWg sync.WaitGroup
    tblWorkerWg     sync.WaitGroup
    processWorkerWg sync.WaitGroup
}

func (e *IndexMergeReaderExecutor) startPartialWorker(
    ctx context.Context,
    workCh chan<- *indexMergeTableTask,
    partialPlan []executor.Executor,
) {
    defer e.partialWorkerWg.Done()
    
    for {
        // Execute partial index scan
        handles, err := e.fetchHandlesFromIndex(ctx, partialPlan)
        if err != nil {
            return
        }
        
        if len(handles) == 0 {
            return
        }
        
        // Send handles for table lookup
        task := &indexMergeTableTask{
            handles: handles,
        }
        
        select {
        case workCh <- task:
        case <-ctx.Done():
            return
        }
    }
}
```

## 🚀 Execution Models

### 1. Volcano Iterator Model

TiDB uses the Volcano iterator model for pipeline execution:

```go
// Base executor interface
type Executor interface {
    Open(ctx context.Context) error
    Next(ctx context.Context, req *chunk.Chunk) error
    Close() error
    Schema() *expression.Schema
}

// Iterator-based execution
func (e *ProjectionExec) Next(ctx context.Context, req *chunk.Chunk) error {
    // 1. Get input from child
    err := Next(ctx, e.children[0], e.childResult)
    if err != nil {
        return err
    }
    
    // 2. Apply projection
    err = e.evaluateExpression(e.childResult, req)
    if err != nil {
        return err
    }
    
    return nil
}
```

### 2. Streaming Execution

For large result sets, TiDB implements streaming execution:

```go
type StreamAggExec struct {
    baseExecutor
    
    aggFuncs    []aggfuncs.AggFunc      // Aggregation functions
    groupByItems []expression.Expression // Group by expressions
    curGroupKey []types.Datum           // Current group key
    
    // Streaming state
    streamingProcessorWg sync.WaitGroup
    inputCh              chan *chunk.Chunk
    outputCh             chan *chunk.Chunk
    errCh                chan error
}

func (e *StreamAggExec) Next(ctx context.Context, req *chunk.Chunk) error {
    // Stream processing with minimal memory footprint
    select {
    case chunk := <-e.outputCh:
        req.SwapColumns(chunk)
        return nil
    case err := <-e.errCh:
        return err
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

### 3. Batch Processing

Optimized batch processing for analytical queries:

```go
type HashAggExec struct {
    baseExecutor
    
    aggFuncs         []aggfuncs.AggFunc
    groupByItems     []expression.Expression
    partialWorkers   []*HashAggPartialWorker
    finalWorker      *HashAggFinalWorker
    
    // Batch configuration
    defaultChunkSize int
    maxBatchSize     int
    spillDisk        bool
}

func (e *HashAggExec) parallelExec(ctx context.Context, req *chunk.Chunk) error {
    // 1. Distribute work across partial workers
    for i, worker := range e.partialWorkers {
        if err := worker.run(ctx, e.children[0]); err != nil {
            return err
        }
    }
    
    // 2. Final aggregation
    if err := e.finalWorker.run(ctx, req); err != nil {
        return err
    }
    
    return nil
}
```

## 🔄 Join Execution Strategies

### 1. Hash Join

Distributed hash join implementation:

```go
type HashJoinExec struct {
    baseExecutor
    
    probeSideTupleFetcher *probeSideTupleFetcher
    buildSideTupleFetcher *buildSideTupleFetcher
    
    // Hash join specific
    buildKeys    []*expression.Column
    probeKeys    []*expression.Column
    buildSideKey []expression.Expression
    probeSideKey []expression.Expression
    
    // Concurrency control
    concurrency     uint
    buildFinished   chan error
    joinChkResourceCh chan *chunk.Chunk
    
    // Memory management
    memTracker      *memory.Tracker
    diskTracker     *disk.Tracker
    spillAction     *memory.ActionSpill
}

func (e *HashJoinExec) fetchAndBuildHashTable(ctx context.Context) error {
    // 1. Fetch build side data
    buildSideResultCh := make(chan *chunk.Chunk, 1)
    doneCh := make(chan struct{})
    
    go e.fetchBuildSideRows(ctx, buildSideResultCh, doneCh)
    
    // 2. Build hash table
    for chk := range buildSideResultCh {
        if err := e.buildHashTable(chk); err != nil {
            return err
        }
    }
    
    return nil
}
```

### 2. Merge Join

Sorted merge join for ordered data:

```go
type MergeJoinExec struct {
    baseExecutor
    
    // Join conditions
    leftKeys     []*expression.Column
    rightKeys    []*expression.Column
    otherFilter  expression.CNFExprs
    
    // Merge state
    leftRows     []chunk.Row
    rightRows    []chunk.Row
    leftPtr      int
    rightPtr     int
    
    // Comparison function
    compareFuncs []chunk.CompareFunc
    ascending    []bool
}

func (e *MergeJoinExec) Next(ctx context.Context, req *chunk.Chunk) error {
    // Merge join algorithm
    for req.NumRows() < e.maxChunkSize {
        // Compare current rows
        cmp := e.compare(e.leftRows[e.leftPtr], e.rightRows[e.rightPtr])
        
        switch {
        case cmp < 0:
            // Advance left side
            e.leftPtr++
            if e.leftPtr >= len(e.leftRows) {
                if err := e.fetchLeftRows(ctx); err != nil {
                    return err
                }
            }
        case cmp > 0:
            // Advance right side
            e.rightPtr++
            if e.rightPtr >= len(e.rightRows) {
                if err := e.fetchRightRows(ctx); err != nil {
                    return err
                }
            }
        default:
            // Match found, output join result
            e.addJoinResult(req, e.leftRows[e.leftPtr], e.rightRows[e.rightPtr])
            e.leftPtr++
            e.rightPtr++
        }
    }
    
    return nil
}
```

## 📊 Performance Optimizations

### 1. Parallel Execution

TiDB maximizes parallelism across multiple dimensions:

```go
// Parallel table scan
type TableScanExec struct {
    baseExecutor
    
    ranges           []kv.KeyRange        // Scan ranges
    concurrency      int                  // Parallel degree
    workers          []*tableScanWorker   // Worker goroutines
    
    // Load balancing
    taskChan         chan *scanTask
    resultChan       chan *scanResult
    workerWg         sync.WaitGroup
}

func (e *TableScanExec) startWorkers(ctx context.Context) {
    for i := 0; i < e.concurrency; i++ {
        worker := &tableScanWorker{
            id:         i,
            executor:   e,
            taskChan:   e.taskChan,
            resultChan: e.resultChan,
        }
        
        e.workerWg.Add(1)
        go worker.run(ctx)
    }
}
```

### 2. Memory Management

Efficient memory usage for large distributed queries:

```go
type MemoryTracker struct {
    mu          sync.RWMutex
    label       int64                    // Tracker label
    consumed    int64                    // Memory consumed
    maxConsumed int64                    // Peak memory usage
    parent      *MemoryTracker           // Parent tracker
    children    map[int64]*MemoryTracker // Child trackers
    
    // Spill control
    actionOnExceed *ActionOnExceed
    spillDisk      bool
    spillRatio     float64
}

func (t *MemoryTracker) Consume(bytes int64) {
    t.mu.Lock()
    defer t.mu.Unlock()
    
    t.consumed += bytes
    if t.consumed > t.maxConsumed {
        t.maxConsumed = t.consumed
    }
    
    // Check if spill action needed
    if t.actionOnExceed != nil && t.consumed > t.actionOnExceed.GetThreshold() {
        t.actionOnExceed.Action(t)
    }
    
    // Propagate to parent
    if t.parent != nil {
        t.parent.Consume(bytes)
    }
}
```

### 3. Result Caching

Cache coprocessor results for repeated queries:

```go
type CopCache struct {
    mu     sync.RWMutex
    cache  map[string]*CachedResult     // Key -> Result
    lru    *list.List                   // LRU list
    
    // Cache configuration
    capacity   int64                    // Max cache size
    admPolicy  string                   // Admission policy
    ttl        time.Duration            // Time to live
}

func (c *CopCache) Get(key string) (*CachedResult, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    if result, exists := c.cache[key]; exists {
        // Move to front (LRU)
        c.lru.MoveToFront(result.element)
        return result, true
    }
    
    return nil, false
}
```

## 🔧 Configuration and Tuning

### Execution Parameters

Key parameters for distributed execution optimization:

```go
// TiDB session variables for distributed execution
SET SESSION tidb_distsql_scan_concurrency = 15;        // Scan concurrency
SET SESSION tidb_index_lookup_concurrency = 4;         // Index lookup concurrency  
SET SESSION tidb_index_lookup_join_concurrency = 4;    // Index join concurrency
SET SESSION tidb_hash_join_concurrency = 5;            // Hash join concurrency
SET SESSION tidb_projection_concurrency = 4;           // Projection concurrency

// Memory management
SET SESSION tidb_mem_quota_query = 1073741824;         // Per-query memory limit
SET SESSION tidb_mem_quota_hash_join = 34359738368;    // Hash join memory limit
SET SESSION tidb_mem_quota_merge_join = 34359738368;   // Merge join memory limit

// MPP configuration
SET SESSION tidb_allow_mpp = ON;                       // Enable MPP
SET SESSION tidb_enforce_mpp = ON;                     // Force MPP
SET SESSION tidb_mpp_store_fail_ttl = '60s';          // MPP store fail TTL
```

### Monitoring and Observability

Key metrics for distributed execution monitoring:

```go
// Execution metrics
var (
    DistSQLQueryCounter = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "tidb_distsql_query_total",
            Help: "Total number of distributed SQL queries",
        }, []string{"type", "result"})
    
    DistSQLDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "tidb_distsql_duration_seconds",
            Help: "Duration of distributed SQL operations",
        }, []string{"type"})
    
    MPPQueryDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "tidb_mpp_query_duration_seconds", 
            Help: "Duration of MPP queries",
        }, []string{"store_type"})
)
```

## 🚀 Best Practices

### 1. Query Design

- **Leverage predicate pushdown** to reduce data transfer
- **Use appropriate join types** based on data distribution
- **Consider index design** for efficient range scans
- **Partition large tables** for better parallelism

### 2. Resource Management

- **Set appropriate concurrency** based on hardware resources
- **Monitor memory usage** to avoid OOM conditions
- **Use streaming execution** for large result sets
- **Configure spill-to-disk** for memory-intensive operations

### 3. Performance Optimization

- **Analyze query execution plans** to identify bottlenecks
- **Monitor region distribution** for load balancing
- **Use TiFlash for analytical workloads** when appropriate
- **Cache frequently accessed data** in memory

## 📚 Related Documentation

- [SQL Layer Deep Dive](sql-layer.md)
- [Storage Engine Integration](storage-engine.md)
- [Query Optimizer](optimizer.md)
- [Performance Guidelines](../patterns/performance.md)