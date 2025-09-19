# TiDB Distributed SQL Framework

## Overview

TiDB's Distributed SQL (DistSQL) framework enables the execution of SQL queries across multiple storage nodes (TiKV/TiFlash) in a distributed manner. This document analyzes the key components of TiDB's distributed SQL execution system.

## 1. DistSQL Framework Architecture

### Core Components
- **DistSQL Package** (`pkg/distsql`): Central coordination layer
- **Coprocessor Client** (`pkg/store/copr`): Handles communication with storage nodes
- **Request Builder**: Constructs distributed execution requests
- **Select Result**: Manages result streaming and aggregation

### Key Files
- `pkg/distsql/distsql.go`: Main DistSQL interface and execution logic
- `pkg/distsql/request_builder.go`: Request construction and key range management
- `pkg/distsql/select_result.go`: Result processing and streaming
- `pkg/distsql/context/context.go`: DistSQL execution context

### Framework Flow
```
SQL Query → Planner → DistSQL Request → Coprocessor → Storage Nodes
```

## 2. Coprocessor Requests

### Coprocessor Client (`pkg/store/copr/coprocessor.go`)

The coprocessor client manages communication with TiKV/TiFlash nodes:

```go
type CopClient struct {
    kv.RequestTypeSupportedChecker
    store           *Store
    replicaReadSeed uint32
}
```

### Request Types
- **DAG Requests**: Directed Acyclic Graph execution plans
- **Analyze Requests**: Statistics collection
- **Checksum Requests**: Data integrity verification
- **Batch Requests**: Bulk operations for TiFlash

### Key Features
- **Streaming Results**: Supports both row-by-row and chunk-based streaming
- **Retry Logic**: Automatic retry with backoff for failed requests
- **Resource Management**: Memory tracking and quota enforcement
- **Load Balancing**: Distributes requests across replicas

### Request Lifecycle
1. **Request Building**: Convert logical plan to physical request
2. **Key Range Splitting**: Split query ranges across regions
3. **Task Distribution**: Send tasks to appropriate storage nodes
4. **Result Aggregation**: Collect and merge results from multiple nodes
5. **Error Handling**: Retry failed requests and handle region splits

## 3. Region Cache and Routing

### Region Cache (`pkg/store/copr/region_cache.go`)

Wraps TiKV's region cache for efficient key-to-region mapping:

```go
type RegionCache struct {
    *tikv.RegionCache
}
```

### Key Range Management (`pkg/store/copr/key_ranges.go`)

Manages key range distribution and optimization:

- **Range Splitting**: Divides key ranges across region boundaries
- **Range Merging**: Combines adjacent ranges for efficiency
- **Int64 Boundary Handling**: Special handling for unsigned integer overflow

### Routing Logic
1. **Key Range Analysis**: Parse table/index key ranges
2. **Region Lookup**: Map key ranges to specific regions
3. **Store Location**: Determine which TiKV/TiFlash stores serve each region
4. **Request Distribution**: Send coprocessor requests to appropriate stores

### Range Conversion Functions
- `TableRangesToKVRanges()`: Convert table scan ranges
- `IndexRangesToKVRanges()`: Convert index scan ranges
- `CommonHandleRangesToKVRanges()`: Handle clustered index ranges

## 4. Distributed Execution

### Execution Patterns

#### Table Reader Execution (`pkg/executor/distsql.go`)
- **TableReaderExecutor**: Distributed table scans
- **IndexReaderExecutor**: Distributed index scans
- **IndexLookUpExecutor**: Index + table lookup operations

#### MPP Execution (`pkg/executor/mpp_gather.go`)
- **MPP Query Coordination**: Manages Massively Parallel Processing queries
- **TiFlash Integration**: Coordinates analytical workloads
- **Exchange Operators**: Handle data shuffling between nodes

### Request Builder (`pkg/distsql/request_builder.go`)

Constructs distributed execution requests:

```go
type RequestBuilder struct {
    kv.Request
    is   infoschema.MetaOnlyInfoSchema
    err  error
    used bool
    dag  *tipb.DAGRequest
}
```

#### Key Methods
- `SetDAGRequest()`: Sets up DAG execution plan
- `SetTableRanges()`: Configures table scan ranges
- `SetIndexRanges()`: Configures index scan ranges
- `SetFromSessionVars()`: Applies session-level settings

### Optimization Strategies
- **Concurrency Control**: Adaptive concurrency based on query complexity
- **Paging**: Reduces memory usage for large result sets
- **Chunk Encoding**: Efficient data serialization format
- **Push-down Predicates**: Filters applied at storage level

## 5. PD Client Integration

### PD Communication
TiDB integrates with Placement Driver (PD) for:
- **Metadata Retrieval**: Table/region metadata
- **Timestamp Oracle**: Global timestamp allocation
- **Region Information**: Current region topology
- **Store Health**: Monitor storage node status

### Mock PD Client (`pkg/store/mockstore/unistore/pd.go`)

For testing and development:

```go
type pdClient struct {
    *us.MockPD
    pd.ResourceManagerClient
    globalConfig      map[string]string
    externalTimestamp atomic.Uint64
    addrs             []string
    keyspaceMeta      *keyspacepb.KeyspaceMeta
}
```

### Integration Points
- **Transaction Management**: TSO allocation for MVCC
- **Region Management**: Region split/merge notifications
- **Load Balancing**: Store capacity and performance metrics
- **Configuration**: Global configuration management

## 6. Performance Optimizations

### Chunk-based Execution
- **Memory Layout**: Optimized for SIMD operations
- **Batch Processing**: Reduces function call overhead
- **Zero-copy Operations**: Minimizes data copying

### Adaptive Concurrency
- **Dynamic Scaling**: Adjusts concurrency based on workload
- **Resource Limits**: Respects memory and CPU constraints
- **Queue Management**: Prevents resource exhaustion

### Caching Strategies
- **Coprocessor Cache**: Results caching for repeated queries
- **Region Cache**: Reduces PD communication overhead
- **Plan Cache**: Reuses execution plans

## 7. Error Handling and Resilience

### Retry Mechanisms
- **Exponential Backoff**: Progressive retry delays
- **Region Refresh**: Handle stale region information
- **Store Failover**: Switch to alternative replicas

### Resource Management
- **Memory Tracking**: Monitor and limit memory usage
- **Timeout Handling**: Prevent hanging operations
- **Circuit Breakers**: Protect against cascading failures

## 8. Integration with Storage Engines

### TiKV Integration
- **Row Store**: OLTP workloads
- **Raft Consensus**: Strong consistency guarantees
- **Snapshot Isolation**: MVCC implementation

### TiFlash Integration
- **Column Store**: OLAP workloads
- **MPP Framework**: Massively parallel processing
- **Real-time Replication**: Delta sync from TiKV

## Summary

TiDB's DistSQL framework provides a sophisticated distributed SQL execution engine that:

1. **Abstracts Complexity**: Hides distributed execution details from upper layers
2. **Optimizes Performance**: Adaptive concurrency, caching, and push-down operations
3. **Ensures Reliability**: Comprehensive error handling and retry mechanisms
4. **Scales Horizontally**: Efficient distribution across multiple storage nodes
5. **Supports Hybrid Workloads**: Both OLTP (TiKV) and OLAP (TiFlash) operations

The framework demonstrates advanced distributed systems principles including consistent hashing, leader election, consensus protocols, and adaptive resource management.