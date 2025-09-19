# TiDB Executor Engine

## Overview

TiDB implements a sophisticated executor engine based on the Volcano model, designed for high-performance query execution with support for parallel processing, memory management, and spill-to-disk capabilities. This document provides a comprehensive analysis of the executor architecture and implementation.

## 1. Executor Framework Architecture

### 1.1 Core Interface Design

The executor framework is built around the `Executor` interface defined in `pkg/executor/internal/exec/executor.go:41-77`:

```go
type Executor interface {
    // Chunk management
    NewChunk() *chunk.Chunk
    NewChunkWithCapacity(fields []*types.FieldType, capacity int, maxCachesize int) *chunk.Chunk
    
    // Runtime and monitoring
    RuntimeStats() *execdetails.BasicRuntimeStats
    HandleSQLKillerSignal() error
    RegisterSQLAndPlanInExecForTopSQL()
    
    // Tree structure
    AllChildren() []Executor
    SetAllChildren([]Executor)
    
    // Volcano model operations
    Open(context.Context) error
    Next(ctx context.Context, req *chunk.Chunk) error
    Close() error
    
    // Schema and type information
    Schema() *expression.Schema
    RetFieldTypes() []*types.FieldType
    InitCap() int
    MaxChunkSize() int
    
    // Session management
    Detach() (Executor, bool)
}
```

### 1.2 Base Executor Implementation

TiDB provides two base executor implementations:

#### BaseExecutorV2 (`pkg/executor/internal/exec/executor.go:278-330`)
- Simplified version without full session context
- Composed of helper structs for modularity:
  - `executorMeta`: Schema and children management
  - `executorChunkAllocator`: Chunk allocation and memory management
  - `executorStats`: Runtime statistics and TopSQL integration
  - `executorKillerHandler`: SQL cancellation support

#### BaseExecutor (`pkg/executor/internal/exec/executor.go:356-409`)
- Full-featured executor with session context
- Extends `BaseExecutorV2` with additional capabilities:
  - Session context access
  - System session management
  - Transaction delta tracking

### 1.3 Executor Composition Pattern

The framework uses composition over inheritance:

```go
type BaseExecutorV2 struct {
    executorMeta           // Schema, children, return types
    executorKillerHandler  // SQL cancellation
    executorStats          // Runtime statistics
    executorChunkAllocator // Memory management
}
```

This design provides:
- **Modularity**: Each component handles specific responsibilities
- **Reusability**: Components can be reused across different executor types
- **Testability**: Individual components can be tested in isolation

## 2. Volcano Model Implementation

### 2.1 Iterator Pattern

TiDB implements the Volcano execution model with batch processing optimizations:

```
"Volcano-An Extensible and Parallel Query Evaluation System"
```

Key characteristics:
- **Open-Next-Close Protocol**: Standard iterator interface
- **Batch Processing**: Returns chunks of rows instead of single rows
- **Lazy Evaluation**: Data flows through the execution tree on-demand

### 2.2 Execution Flow

1. **Open Phase** (`pkg/executor/internal/exec/executor.go:426-438`):
   ```go
   func Open(ctx context.Context, e Executor) error {
       // Runtime stats tracking
       if e.RuntimeStats() != nil {
           start := time.Now()
           defer func() { e.RuntimeStats().RecordOpen(time.Since(start)) }()
       }
       return e.Open(ctx)
   }
   ```

2. **Next Phase** (`pkg/executor/internal/exec/executor.go:440-467`):
   ```go
   func Next(ctx context.Context, e Executor, req *chunk.Chunk) error {
       // Performance monitoring
       // SQL killer signal handling
       // TopSQL registration
       // Tracing integration
       return e.Next(ctx, req)
   }
   ```

3. **Close Phase** (`pkg/executor/internal/exec/executor.go:469-481`):
   - Resource cleanup
   - Statistics finalization
   - Error handling with recovery

### 2.3 Chunk-Based Processing

TiDB uses vectorized execution with chunks:
- **Chunk Size**: Configurable via session variables
- **Memory Pool**: Reuses chunk allocations
- **Columnar Layout**: Optimized for SIMD operations

## 3. Memory Management

### 3.1 Memory Tracking Architecture

Each executor implements memory tracking through multiple layers:

#### Chunk Allocator (`pkg/executor/internal/exec/executor.go:81-133`)
```go
type executorChunkAllocator struct {
    AllocPool     chunk.Allocator
    retFieldTypes []*types.FieldType
    initCap       int
    maxChunkSize  int
}
```

#### Memory Tracker Integration
- **Hierarchical Tracking**: Session → Statement → Executor
- **OOM Protection**: Automatic spill-to-disk when memory limits exceeded
- **Memory Pool**: Reuses allocated chunks to reduce GC pressure

### 3.2 Spill-to-Disk Strategy

Critical executors implement spill capabilities:

#### Hash Join Spill (`pkg/executor/join/hash_join_spill.go`)
- **Partition-based Spilling**: Spills hash table partitions
- **Grace Hash Join**: Falls back to disk-based join algorithm
- **Memory Threshold**: Configurable memory limits

#### Sort Spill (`pkg/executor/sortexec/sort_spill.go`)
- **External Merge Sort**: Multi-way merge from disk
- **Partition Sorting**: Sorts in-memory partitions before spill
- **Compression**: Optional compression for spilled data

### 3.3 Memory Control Mechanisms

```go
// Example from ParallelNestedLoopApplyExec
e.memTracker = memory.NewTracker(e.ID(), -1)
e.memTracker.AttachTo(e.Ctx().GetSessionVars().StmtCtx.MemTracker)
```

Key features:
- **Limit Enforcement**: Hard and soft memory limits
- **Spill Triggers**: Automatic spill when approaching limits
- **Resource Accounting**: Tracks all memory allocations

## 4. Parallel Execution Architecture

### 4.1 Worker-Based Parallelism

TiDB implements sophisticated parallel execution patterns:

#### Hash Aggregation (`pkg/executor/aggregate/agg_hash_executor.go:94`)
```
                        +-------------+
                        | Main Thread |
                        +------+------+
                               ^
                               |
                          finalOutputCh
                               |
                 +--------------+-------------+
                 | final worker |   | final worker |
                 +------------+-+   +-+------------+
                              ^       ^
                              |       |
                      partialOutputChs
                              |       |
                 +------------+-+   +-+------------+
                 | partial worker |  | partial worker |
                 +--------------+-+  +-+--------------+
                                ^      ^
                                |      |
                        partialInputChs
                                |      |
                      +----v---------+
                      | data fetcher |
                      +--------------+
```

#### Parallel Sort (`pkg/executor/sortexec/sort.go:83-97`)
- **Worker Pool**: Multiple sort workers process chunks concurrently
- **Multi-way Merge**: Combines sorted results from parallel workers
- **Spill Coordination**: Coordinated spilling across workers

### 4.2 Concurrency Control

#### Channel-Based Communication
```go
type HashAggExec struct {
    inputCh         chan *HashAggInput
    partialOutputChs []chan *chunk.Chunk
    finalOutputCh   chan *chunk.Chunk
}
```

#### Synchronization Patterns
- **WaitGroups**: Coordinate worker lifecycle
- **Channels**: Buffered channels for data flow
- **Atomic Operations**: Lock-free coordination where possible

### 4.3 Resource Management

#### Worker Lifecycle
1. **Initialization**: Workers created during Open()
2. **Execution**: Data flows through worker pipelines
3. **Cleanup**: Coordinated shutdown in Close()

#### Load Balancing
- **Round-Robin**: Distributes input chunks evenly
- **Work Stealing**: Advanced algorithms balance load dynamically
- **Adaptive Parallelism**: Adjusts worker count based on data characteristics

## 5. Advanced Features

### 5.1 Apply Cache (`pkg/executor/internal/applycache/`)
- **Correlation Caching**: Caches results for correlated subqueries
- **Cache Invalidation**: Intelligent cache management
- **Hit Rate Optimization**: Adaptive caching strategies

### 5.2 Runtime Statistics Integration
- **Execution Time**: Per-operator timing
- **Memory Usage**: Peak and average memory consumption
- **Row Counts**: Input/output row statistics
- **Spill Events**: Disk I/O tracking

### 5.3 Fault Tolerance
- **SQL Killer**: Graceful query cancellation
- **Resource Cleanup**: Proper cleanup on errors
- **Partial Results**: Returns partial results when possible

## 6. Performance Optimizations

### 6.1 Vectorized Processing
- **SIMD Operations**: Utilizes CPU SIMD instructions
- **Batch Processing**: Reduces function call overhead
- **Cache Locality**: Optimized memory access patterns

### 6.2 Memory Optimizations
- **Pool Allocation**: Reuses memory allocations
- **Lazy Materialization**: Delays expensive operations
- **Compression**: Reduces memory footprint

### 6.3 I/O Optimizations
- **Asynchronous I/O**: Non-blocking disk operations
- **Prefetching**: Anticipates data access patterns
- **Compression**: Reduces I/O bandwidth requirements

## Architecture Summary

The TiDB executor engine represents a sophisticated implementation of the Volcano model with modern optimizations:

1. **Modular Design**: Composition-based architecture for flexibility
2. **Memory Safety**: Comprehensive memory tracking and spill capabilities
3. **Parallel Processing**: Advanced worker-based parallelism
4. **Performance**: Vectorized execution with optimized memory access
5. **Reliability**: Robust error handling and resource management

This architecture enables TiDB to handle complex analytical workloads while maintaining ACID properties and providing predictable performance characteristics.