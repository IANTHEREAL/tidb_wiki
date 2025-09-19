# TiKV Integration in TiDB

## Overview

TiDB's integration with TiKV provides a distributed transactional storage foundation. This document details how TiDB connects to, communicates with, and coordinates operations across the TiKV cluster through sophisticated client libraries and protocols.

## Integration Architecture

### Component Stack

```
┌─────────────────────────────────────────┐
│              TiDB SQL Layer             │
├─────────────────────────────────────────┤
│         Storage Interface (pkg/kv)      │
├─────────────────────────────────────────┤
│        TiKV Driver (pkg/store/driver)   │
├─────────────────────────────────────────┤
│       TiKV Client-Go (tikv/client-go)   │
├─────────────────────────────────────────┤
│              gRPC Network               │
├─────────────────────────────────────────┤
│            TiKV Cluster                 │
└─────────────────────────────────────────┘
```

## 1. Client Library Dependencies

### Core Dependencies

From TiDB's `go.mod`:
```
github.com/tikv/client-go/v2 v2.0.8-0.20250905073636-469a7adf7ae8
github.com/tikv/pd/client v0.0.0-20250703091733-dfd345b89500
```

### Key Client Components

**tikv/client-go/v2** provides:
- **KV Operations**: Basic get/set/delete primitives
- **Transaction Manager**: 2PC transaction coordination
- **Region Cache**: Automatic region routing and caching
- **Retry Logic**: Resilient request handling
- **Load Balancing**: Request distribution across replicas

**tikv/pd/client** provides:
- **Cluster Metadata**: TSO, region info, store status
- **Leader Discovery**: PD leader election handling
- **Configuration**: Cluster-wide settings management

## 2. TiKV Driver Implementation

### Driver Registration

**TiKVDriver** (`pkg/store/driver/tikv_driver.go`):
```go
type TiKVDriver struct {
    pdConfig        config.PDClient
    security        config.Security
    tikvConfig      config.TiKVClient
    txnLocalLatches config.TxnLocalLatches
}
```

### Store Connection Management

**Connection Pooling**:
```go
type storeCache struct {
    sync.Mutex
    cache map[string]*tikvStore
}
```

**Store Creation Process**:
1. **Parse Connection String**: Extract PD endpoints and options
2. **Create PD Client**: Establish PD connection for metadata
3. **Initialize TiKV Store**: Create KVStore with region cache
4. **Setup SafePoint**: Configure GC coordination
5. **Cache Store Instance**: Reuse connections across sessions

### Configuration Integration

**Resource Control Hooks**:
```go
func init() {
    variable.EnableGlobalResourceControlFunc = tikv.EnableResourceControl
    variable.DisableGlobalResourceControlFunc = tikv.DisableResourceControl
}
```

## 3. Region Cache and Routing

### Region Cache Architecture

The region cache maintains TiKV cluster topology:

**RegionCache** features:
- **Key Range Mapping**: Key → Region → Store resolution
- **Leader Tracking**: Primary replica identification
- **Follower Management**: Read replica load balancing
- **Invalidation**: Automatic cache refresh on topology changes

### Request Routing

**Key → Region Resolution**:
1. **Range Lookup**: Binary search in region cache
2. **Store Selection**: Choose appropriate TiKV node
3. **Connection Reuse**: gRPC connection pooling
4. **Failure Handling**: Automatic failover and retry

**RegionRequestSender** workflow:
```go
// Pseudo-code for request routing
func (s *RegionRequestSender) SendReq(key []byte, req *Request) {
    region := s.regionCache.LocateKey(key)
    store := s.selectStore(region, req.ReadType)
    
    resp, err := s.sendToStore(store, req)
    if err != nil {
        s.handleError(err, region)
        // Retry with updated cache
    }
}
```

## 4. Transaction Coordination

### Two-Phase Commit (2PC)

**Transaction Flow**:
1. **Begin**: Acquire start timestamp from PD
2. **Execution Phase**: Buffer mutations in memory
3. **Prewrite Phase**: Lock all keys across regions
4. **Commit Phase**: Atomic commit with commit timestamp

**Distributed Coordination**:
```go
type tikvTxn struct {
    *tikv.KVTxn                    // Underlying TiKV transaction
    idxNameCache map[int64]*model.TableInfo  // Schema cache
    snapshotInterceptor kv.SnapshotInterceptor
    isCommitterWorking atomic.Bool  // Commit state tracking
}
```

### Pessimistic vs Optimistic

**Optimistic Transactions**:
- Fast execution with conflict detection at commit
- Suitable for low-contention workloads
- Uses 2PC with prewrite validation

**Pessimistic Transactions**:
- Acquire locks during execution
- Better for high-contention scenarios
- Immediate conflict detection

## 5. Connection and Network Management

### gRPC Configuration

**Connection Settings**:
```go
func createGRPCConn(addr string, opts *options) (*grpc.ClientConn, error) {
    dialOpts := []grpc.DialOption{
        grpc.WithKeepaliveParams(keepalive.ClientParameters{
            Time:                time.Duration(pingTime) * time.Second,
            Timeout:             time.Duration(pingTimeout) * time.Second,
            PermitWithoutStream: true,
        }),
        grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(maxMsgSize)),
    }
    
    return grpc.Dial(addr, dialOpts...)
}
```

### TLS and Security

**Security Configuration**:
- **mTLS**: Mutual authentication between TiDB and TiKV
- **Certificate Management**: Automatic cert rotation
- **Encryption**: In-transit data protection

### Load Balancing Strategies

**Follower Read**:
- **Learner Replicas**: Read from closest replica
- **Stale Read**: Accept slightly stale data for performance
- **Leader Fallback**: Automatic fallback for strong consistency

**Request Distribution**:
- **Round Robin**: Even distribution across healthy nodes
- **Locality Awareness**: Prefer local replicas
- **Health Monitoring**: Exclude unhealthy nodes

## 6. Error Handling and Resilience

### Retry Mechanisms

**Backoff Strategy**:
```go
type BackoffConfig struct {
    BaseDelay  time.Duration
    Multiplier float64
    Jitter     float64
    MaxDelay   time.Duration
}
```

**Error Classification**:
- **Retryable**: Network timeouts, region splits, leader changes
- **Non-retryable**: Write conflicts, data corruption
- **Circuit Breaker**: Temporary failure isolation

### Region Management

**Dynamic Region Splitting**:
- **Automatic Splits**: Handle region size limits
- **Cache Invalidation**: Update routing on splits
- **Key Range Updates**: Maintain accurate mappings

**Store Failure Handling**:
- **Health Checks**: Continuous store monitoring
- **Failover**: Automatic replica promotion
- **Recovery**: Graceful store re-integration

## 7. Performance Optimizations

### Batch Operations

**BatchGet Optimization**:
```go
func (s *Snapshot) BatchGet(keys []Key) (map[string][]byte, error) {
    // Group keys by region
    regionTasks := s.groupKeysByRegion(keys)
    
    // Parallel execution across regions
    results := make(chan batchResult, len(regionTasks))
    for _, task := range regionTasks {
        go s.executeBatchTask(task, results)
    }
    
    return s.mergeBatchResults(results)
}
```

### Coprocessor Integration

**Pushdown Coordination**:
- **Task Distribution**: Split computation across regions
- **Result Streaming**: Incremental result processing
- **Resource Management**: CPU and memory limits

### Connection Pooling

**Pool Management**:
- **Per-Store Pools**: Dedicated connections per TiKV node
- **Connection Reuse**: Long-lived connection sharing
- **Pool Sizing**: Dynamic scaling based on load

## 8. Monitoring and Observability

### Metrics Integration

**Client Metrics**:
- **Request Latency**: P99, P95 latencies per operation type
- **Error Rates**: Retry counts and failure classifications
- **Connection Health**: Active connections and pool utilization

**Integration Points**:
```go
// Prometheus metrics example
var (
    tikvRequestDuration = prometheus.NewHistogramVec(...)
    tikvRequestCounter = prometheus.NewCounterVec(...)
    regionCacheHitRatio = prometheus.NewGaugeVec(...)
)
```

### Tracing Support

**Distributed Tracing**:
- **Span Creation**: Automatic request tracing
- **Context Propagation**: Trace context across services
- **Baggage Support**: Custom metadata propagation

## 9. Configuration Management

### Client Configuration

**Key Settings**:
```yaml
tikv_client:
  grpc_connection_count: 4
  grpc_keepalive_time: 10s
  grpc_keepalive_timeout: 3s
  region_cache_ttl: 600s
  store_limit: 0
  copr_cache:
    capacity_mb: 1000
```

### Dynamic Configuration

**Runtime Updates**:
- **Feature Flags**: Enable/disable optimizations
- **Resource Limits**: Adjust memory and CPU usage
- **Retry Parameters**: Tune backoff and timeout values

## 10. Deployment Considerations

### Network Requirements

**Connectivity**:
- **TiDB ↔ PD**: Cluster metadata and TSO
- **TiDB ↔ TiKV**: Data operations and coprocessor
- **Latency Sensitivity**: Sub-millisecond for local operations

### Resource Planning

**Memory Usage**:
- **Region Cache**: ~1MB per 10K regions
- **Connection Pools**: ~1MB per 100 connections
- **Transaction Buffers**: Variable based on transaction size

**CPU Utilization**:
- **Network I/O**: gRPC serialization/deserialization
- **Cache Management**: Region cache maintenance
- **Retry Logic**: Exponential backoff calculations

## Key Integration Benefits

1. **Scalability**: Automatic sharding across TiKV nodes
2. **Consistency**: ACID transactions with snapshot isolation
3. **Availability**: Automatic failover and replica management
4. **Performance**: Coprocessor pushdown and batch operations
5. **Observability**: Comprehensive metrics and tracing
6. **Security**: mTLS and certificate-based authentication

This tight integration enables TiDB to provide SQL semantics over a distributed, consistent, and highly available key-value store, making it suitable for demanding OLTP and OLAP workloads.