# TiDB Storage Layer Architecture

## Overview

TiDB's storage layer provides a clean abstraction over distributed key-value operations, enabling ACID transactions and efficient data access. The storage layer consists of several key components that work together to provide a unified interface to the distributed TiKV storage engine.

## 1. KV Interface Abstraction (pkg/kv)

### Core Interfaces

The KV package defines fundamental interfaces that abstract storage operations:

#### Basic Access Patterns

```go
// Getter - Basic key-value retrieval
type Getter interface {
    Get(ctx context.Context, k Key) ([]byte, error)
}

// Mutator - Basic key-value modification
type Mutator interface {
    Set(k Key, v []byte) error
    Delete(k Key) error
}

// Retriever - Enhanced retrieval with iteration
type Retriever interface {
    Getter
    Iter(k Key, upperBound Key) (Iterator, error)
    IterReverse(k, lowerBound Key) (Iterator, error)
}
```

#### Storage Interface Hierarchy

**Storage** - Main storage interface providing transaction management:
- `Begin()` - Start new transaction
- `BeginWithStartTS()` - Begin with specific timestamp
- `GetSnapshot()` - Create read-only snapshot
- `Close()` - Cleanup resources

**Transaction** - ACID transaction interface:
- Inherits `RetrieverMutator`, `AssertionProto`, `FairLockingController`
- `Commit(context.Context) error` - Commit changes
- `Rollback() error` - Abort transaction
- `LockKeys()` - Pessimistic locking
- `GetMemBuffer()` - Access transaction buffer
- `GetSnapshot()` - Access transaction snapshot

**Snapshot** - Point-in-time read interface:
- Inherits `Retriever`
- `BatchGet()` - Batch key retrieval
- `SetOption()` - Configure snapshot behavior

#### Memory Buffer Architecture

**MemBuffer** - In-memory transaction staging:
```go
type MemBuffer interface {
    RetrieverMutator
    
    // Staging buffer management
    Staging() StagingHandle
    Release(StagingHandle)
    Cleanup(StagingHandle)
    
    // Concurrent access protection
    RLock()
    RUnlock()
    
    // Flag-based operations
    SetWithFlags(Key, []byte, ...FlagsOp) error
    GetFlags(Key) (KeyFlags, error)
    UpdateFlags(Key, ...FlagsOp)
}
```

### Key Features

- **Staging Buffers**: Hierarchical transaction buffers for atomic modifications
- **Key Flags**: Metadata tracking for optimizations (e.g., `UnCommitIndexKVFlag`)
- **Concurrent Safety**: Reader-writer locks for safe concurrent access
- **Snapshot Isolation**: Point-in-time consistent views

## 2. TiKV Client Implementation

### Driver Architecture

The TiKV driver (`pkg/store/driver`) provides the concrete implementation of KV interfaces:

#### TiKV Store Driver

**tikvStore** - Main storage implementation:
- Wraps `tikv.KVStore` from client-go
- Manages PD client connections
- Handles cluster metadata
- Provides transaction factory methods

```go
type tikvStore struct {
    tikv.Storage
    pdClient   pd.Client
    pdHTTPClient pdhttp.Client
    etcdAddrs    []string
    tlsConfig    *tls.Config
    tokenManager *tokenManager
}
```

#### Transaction Implementation

**tikvTxn** - Transaction wrapper:
```go
type tikvTxn struct {
    *tikv.KVTxn
    idxNameCache        map[int64]*model.TableInfo
    snapshotInterceptor kv.SnapshotInterceptor
    columnMapsCache     any
    isCommitterWorking  atomic.Bool
    memBuffer          *memBuffer
}
```

Key capabilities:
- **KV Filtering**: `TiDBKVFilter` for key validation
- **Index Caching**: Table schema caching for error reporting
- **Snapshot Interception**: Middleware for snapshot operations
- **Commit Coordination**: Atomic commit state management

### Client-Go Integration

TiDB leverages `github.com/tikv/client-go/v2` for:
- **Region Routing**: Automatic request routing to appropriate TiKV nodes
- **Load Balancing**: Round-robin and leader-follower strategies
- **Retry Logic**: Exponential backoff with jitter
- **Connection Pooling**: gRPC connection management

## 3. Transaction Buffer and Mutations

### Memory Buffer Implementation

**memBuffer** (`pkg/store/driver/txn/unionstore_driver.go`):
```go
type memBuffer struct {
    tikv.MemBuffer
    isPipelinedDML bool
}
```

#### Staging Buffer System

The staging buffer system enables hierarchical transaction modifications:

1. **Primary Buffer**: Main transaction state
2. **Staging Buffers**: Temporary modification areas
3. **Handle Management**: Reference counting for buffer lifecycle

**Workflow**:
```go
// Create staging area
handle := memBuf.Staging()

// Make modifications
memBuf.Set(key, value)
memBuf.Delete(key2)

// Commit or discard
memBuf.Release(handle)  // Commit to parent
// OR
memBuf.Cleanup(handle)  // Discard changes
```

### Mutation Tracking

**KeyFlags** - Per-key metadata:
- `UnCommitIndexKVFlag`: Skip unchanged index entries
- `PresumeKeyNotExists`: Optimization for new keys
- `IsRowLock`: Row-level locking information

**Flag Operations**:
```go
type FlagsOp interface {
    ApplyFlagsOps(v *KeyFlags)
}
```

### Pipelined DML

For bulk operations, TiDB supports pipelined DML:
- **Batched Mutations**: Group related changes
- **Async Commit**: Non-blocking commit pipeline
- **Memory Management**: Automatic buffer flushing

## 4. Range Scans and Iterators

### Iterator Interface

**Iterator** - Core iteration abstraction:
```go
type Iterator interface {
    Valid() bool
    Key() Key
    Value() []byte
    Next() error
    Close()
}
```

### Scan Implementations

#### Forward Iteration
```go
iter, err := snapshot.Iter(startKey, endKey)
defer iter.Close()

for iter.Valid() {
    key := iter.Key()
    value := iter.Value()
    // Process key-value pair
    err := iter.Next()
    if err != nil {
        break
    }
}
```

#### Reverse Iteration
```go
iter, err := snapshot.IterReverse(startKey, lowerBound)
defer iter.Close()

for iter.Valid() {
    // Process in reverse order
    err := iter.Next()
    if err != nil {
        break
    }
}
```

### Range Optimization

**NextUntil** utility:
```go
func NextUntil(it Iterator, fn FnKeyCmp) error {
    var err error
    for it.Valid() && !fn(it.Key()) {
        err = it.Next()
        if err != nil {
            return err
        }
    }
    return nil
}
```

## 5. Coprocessor Pushdown

### Architecture Overview

Coprocessor pushdown enables computation delegation to TiKV nodes, reducing network overhead and improving performance.

### Request Structure

**copTask** - Coprocessor task representation:
```go
type copTask struct {
    taskID     uint64
    region     tikv.RegionVerID
    bucketsVer uint64
    ranges     *KeyRanges
    
    respChan  chan *copResponse
    storeAddr string
    cmdType   tikvrpc.CmdType
    storeType kv.StoreType
    
    // Paging support
    paging        bool
    pagingSize    uint64
    pagingTaskIdx uint32
}
```

### Task Distribution

**RegionBatchRequestSender** - Manages coprocessor request distribution:
1. **Task Building**: Split requests by region boundaries
2. **Parallel Execution**: Concurrent task processing
3. **Result Aggregation**: Merge partial results
4. **Error Handling**: Retry failed tasks with backoff

### Response Processing

**copResponse** - Coprocessor result container:
```go
type copResponse struct {
    pbResp        *coprocessor.Response
    startKey      kv.Key
    execDetails   *execdetails.ExecDetails
    detail        *CopRuntimeStats
    err           error
}
```

### Pushdown Types

1. **Selection**: Filter pushdown to reduce data transfer
2. **Projection**: Column selection at storage layer
3. **Aggregation**: Group-by and aggregate functions
4. **Join**: Hash joins between co-located data
5. **TopN**: Limit and order operations

### Performance Optimizations

- **Paging**: Large result set streaming
- **Caching**: Response caching for repeated queries
- **Batching**: Multiple regions per request
- **Store Selection**: TiKV vs TiFlash routing based on query type

## Key Design Principles

1. **Abstraction**: Clean interface separation between logical and physical layers
2. **Consistency**: ACID guarantees through staging buffers and two-phase commit
3. **Performance**: Parallel execution, caching, and pushdown optimizations
4. **Reliability**: Comprehensive error handling and retry mechanisms
5. **Scalability**: Distributed architecture with automatic sharding