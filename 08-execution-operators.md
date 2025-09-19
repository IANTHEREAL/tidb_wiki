# TiDB Execution Operators

## Overview

This document provides detailed analysis of TiDB's execution operators, covering their implementation strategies, optimization techniques, and performance characteristics. TiDB implements a comprehensive set of operators following the Volcano model with modern optimizations for parallel execution and memory management.

## 1. Hash Join Operators

### 1.1 Hash Join Architecture

TiDB implements multiple hash join variants located in `pkg/executor/join/`:

#### Core Components (`pkg/executor/join/hash_join_base.go`)

```go
type hashJoinCtxBase struct {
    SessCtx        sessionctx.Context
    ChunkAllocPool chunk.Allocator
    Concurrency    uint
    joinResultCh   chan *hashjoinWorkerResult
    closeCh        chan struct{}
    finished       atomic.Bool
    IsNullEQ       []bool
    buildFinished  chan error
    JoinType       base.JoinType
    IsNullAware    bool
    memTracker     *memory.Tracker
    diskTracker    *disk.Tracker
}
```

Key features:
- **Worker-based parallelism**: Configurable concurrency
- **Memory management**: Integrated memory and disk tracking
- **Null-aware joins**: Support for complex null semantics
- **Channel-based coordination**: Lock-free communication

### 1.2 Hash Join V1 Implementation (`pkg/executor/join/hash_join_v1.go`)

#### Build Phase Architecture
```go
type BuildWorkerV1 struct {
    buildWorkerBase
    HashJoinCtx *HashJoinCtxV1
    buildKeyColIdx []int
    buildNAKeyColIdx []int
    // Hash table construction components
    builded bool
    hashTable hashRowContainer
}
```

#### Probe Phase Architecture
```go
type ProbeWorkerV1 struct {
    probeWorkerBase
    HashJoinCtx      *HashJoinCtxV1
    ProbeKeyColIdx   []int
    ProbeNAKeyColIdx []int
    // Pre-allocated buffers for performance
    buildSideRows    []chunk.Row
    buildSideRowPtrs []chunk.RowPtr
    // Individual joiner per worker
    Joiner               Joiner
    rowIters             *chunk.Iterator4Slice
    rowContainerForProbe *hashRowContainer
}
```

#### Execution Flow
1. **Build Phase**: 
   - Build workers read from build-side child
   - Construct hash table partitions
   - Signal build completion via `buildFinished` channel

2. **Probe Phase**:
   - Probe workers read from probe-side child
   - Lookup matching rows in hash table
   - Join and output result chunks

3. **Coordination**:
   ```go
   func wait4BuildSide(isBuildEmpty, checkSpill, canSkipIfBuildEmpty, needScanAfterProbeDone, hashJoinCtx) (skipProbe, buildSuccess) {
       select {
       case <-hashJoinCtx.closeCh:
           skipProbe = true
       case err = <-hashJoinCtx.buildFinished:
           buildSuccess = (err == nil)
       }
       // Optimization: skip probe for empty build side in inner joins
       if buildSuccess && isBuildEmpty() && canSkipIfBuildEmpty {
           skipProbe = true
       }
   }
   ```

### 1.3 Spill-to-Disk Strategy (`pkg/executor/join/hash_join_spill.go`)

#### Grace Hash Join Algorithm
When memory pressure is detected:
1. **Partition Spilling**: Hash table partitions spilled to disk
2. **Probe Spilling**: Corresponding probe data spilled 
3. **Recursive Processing**: Process spilled partitions recursively
4. **Memory Reclamation**: Free memory for current partition

#### Memory Management
- **Threshold-based**: Configurable memory limits trigger spilling
- **Partition-wise**: Individual partitions can be spilled independently
- **Compression**: Optional compression for spilled data

### 1.4 Join Types and Optimizations

#### Supported Join Types
- **Inner Join**: Standard hash join with early termination optimizations
- **Left/Right Outer Join**: Null-padding for unmatched rows
- **Semi Join**: Existence checks with early exit
- **Anti Semi Join**: Non-existence checks with specialized algorithms
- **Null-Aware Anti Join**: Complex null semantics handling

#### Performance Optimizations
- **Bloom Filters**: Reduce probe-side scanning
- **Runtime Filter Pushdown**: Dynamic filtering of probe side
- **Vectorized Hashing**: SIMD-optimized hash computation
- **Cache-Conscious Layout**: Optimized memory access patterns

## 2. Sort Operators

### 2.1 Sort Executor Architecture (`pkg/executor/sortexec/sort.go`)

```go
type SortExec struct {
    exec.BaseExecutor
    ByItems         []*plannerutil.ByItems
    fetched         *atomic.Bool
    ExecSchema      *expression.Schema
    keyColumns      []int
    keyCmpFuncs     []chunk.CompareFunc
    curPartition    *sortPartition
    spillLimit      int64
    memTracker      *memory.Tracker
    diskTracker     *disk.Tracker
    IsUnparallel    bool
    finishCh        chan struct{}
    multiWayMerge   *multiWayMerger
}
```

### 2.2 Parallel Sort Implementation

#### Worker-Based Architecture
```go
type parallelSortWorker struct {
    workerID       int
    chunkChannel   chan *chunkWithMemoryUsage
    sortedRowsIter *chunk.Iterator4Slice
    memTracker     *memory.Tracker
    diskTracker    *disk.Tracker
    spillAction    *parallelSortSpillAction
}
```

#### Execution Flow
1. **Data Distribution**: Input chunks distributed among workers
2. **Parallel Sorting**: Each worker sorts assigned chunks
3. **Spill Management**: Individual workers spill when memory constrained
4. **Multi-way Merge**: Results merged using k-way merge algorithm

#### Spill Strategy (`pkg/executor/sortexec/sort_spill.go`)
```go
type sortPartitionSpillDiskAction struct {
    spilledBytes     *atomic.Int64
    spilledRows      *atomic.Int64
    spilledPartitions []*sortPartition
    multiWayMerger   *multiWayMerger
}
```

### 2.3 Sort Optimizations

#### Memory-Aware Sorting
- **Adaptive Chunking**: Adjusts chunk sizes based on available memory
- **Incremental Spilling**: Spills data incrementally to maintain working set
- **Memory Pool**: Reuses sorted chunks to reduce allocation overhead

#### Algorithm Optimizations
- **Quicksort Hybrid**: Uses quicksort for small datasets, merge sort for larger
- **Vectorized Comparisons**: SIMD-optimized comparison functions
- **Cache-Friendly Layout**: Optimizes memory access patterns for sorting

## 3. Aggregation Operators

### 3.1 Hash Aggregation Architecture (`pkg/executor/aggregate/agg_hash_executor.go`)

#### Parallel Execution Pipeline
```
┌─────────────┐
│ Main Thread │
└──────┬──────┘
       │ finalOutputCh
       ▼
┌─────────────┬─────────────┐
│final worker │final worker │
└──────┬──────┴──────┬──────┘
       │             │ partialOutputChs
       ▼             ▼
┌──────────────┬──────────────┐
│partial worker│partial worker│
└──────┬───────┴──────┬───────┘
       │              │ partialInputChs
       ▼              ▼
┌─────────────────────────────┐
│      data fetcher           │
└─────────────────────────────┘
```

#### Core Components
```go
type HashAggExec struct {
    exec.BaseExecutor
    Sc               *stmtctx.StatementContext
    PartialAggFuncs  []aggfuncs.AggFunc
    FinalAggFuncs    []aggfuncs.AggFunc
    partialResultMap aggfuncs.AggPartialResultMapper
    groupSet         set.StringSet
    inputCh          chan *HashAggInput
    partialOutputChs []chan *chunk.Chunk
    finalOutputCh    chan *chunk.Chunk
    partialWorkers   []HashAggPartialWorker
    finalWorkers     []HashAggFinalWorker
}
```

### 3.2 Partial Workers (`pkg/executor/aggregate/agg_hash_partial_worker.go`)

#### Worker Architecture
```go
type HashAggPartialWorker struct {
    baseHashAggWorker
    ctx                   sessionctx.Context
    inputCh              chan *chunk.Chunk
    outputChs            []chan *aggfuncs.AggPartialResultMapper
    globalOutputCh       chan *AfFinalResult
    giveBackCh           chan<- *HashAggInput
    BInMaps              []int
    partialResultsBuffer [][]aggfuncs.PartialResult
    partialResultsMap    []aggfuncs.AggPartialResultMapper
    groupByItems         []expression.Expression
    groupKeyBuf          [][]byte
    spillHelper          *parallelHashAggSpillHelper
}
```

#### Processing Pipeline
1. **Input Processing**: Receives chunks from data fetcher
2. **Group Key Generation**: Computes hash keys for grouping
3. **Partial Aggregation**: Updates partial aggregate states
4. **Result Distribution**: Sends results to appropriate final workers
5. **Memory Management**: Tracks memory usage and triggers spilling

### 3.3 Final Workers (`pkg/executor/aggregate/agg_hash_final_worker.go`)

#### Responsibilities
- **Partial Result Merging**: Combines partial results from multiple workers
- **Final Computation**: Computes final aggregate values
- **Result Output**: Sends completed results to main thread

#### Memory Management
```go
func (w *HashAggFinalWorker) processPartialResults() {
    for {
        select {
        case partialResult := <-w.inputCh:
            // Merge partial results
            w.mergePartialResults(partialResult)
            // Check memory pressure
            if w.memTracker.BytesConsumed() > w.memLimit {
                w.triggerSpill()
            }
        case <-w.finishCh:
            return
        }
    }
}
```

### 3.4 Aggregation Function Framework (`pkg/executor/aggfuncs/`)

#### Function Interface
```go
type AggFunc interface {
    AllocPartialResult() PartialResult
    ResetPartialResult(pr PartialResult)
    UpdatePartialResult(sctx sessionctx.Context, rowsInGroup []chunk.Row, pr PartialResult) error
    MergePartialResult(sctx sessionctx.Context, src, dst PartialResult) error
    AppendFinalResult2Chunk(sctx sessionctx.Context, pr PartialResult, chk *chunk.Chunk) error
}
```

#### Built-in Functions
- **Statistical**: COUNT, SUM, AVG, MIN, MAX, VARIANCE, STDDEV
- **Advanced**: GROUP_CONCAT, JSON_ARRAYAGG, JSON_OBJECTAGG
- **Window**: ROW_NUMBER, RANK, DENSE_RANK, LEAD, LAG
- **Bitwise**: BIT_AND, BIT_OR, BIT_XOR

## 4. Window Function Operators

### 4.1 Window Executor (`pkg/executor/window.go`)

```go
type WindowExec struct {
    exec.BaseExecutor
    groupChecker      *vecgroupchecker.VecGroupChecker
    childResult       *chunk.Chunk
    executed          bool
    resultChunks      []*chunk.Chunk
    remainingRowsInChunk []int
    numWindowFuncs    int
    processor         windowProcessor
}
```

#### Execution Strategy
1. **Group Processing**: Process rows in window frame groups
2. **Frame Management**: Maintains current window frame boundaries
3. **Function Evaluation**: Evaluates window functions for each row
4. **Result Buffering**: Buffers results for batch output

### 4.2 Pipelined Window Processing (`pkg/executor/pipelined_window.go`)

#### Streaming Window Algorithm
- **Frame Sliding**: Efficiently slides window frames
- **Incremental Computation**: Updates aggregates incrementally
- **Memory Efficiency**: Processes large datasets with bounded memory

## 5. Specialized Operators

### 5.1 Index Merge Reader (`pkg/executor/index_merge_reader.go`)

#### Multi-Index Access
- **Parallel Index Scans**: Scans multiple indexes concurrently
- **Result Merging**: Merges results while eliminating duplicates
- **Pushdown Optimization**: Pushes predicates to storage layer

### 5.2 Table Reader (`pkg/executor/table_reader.go`)

#### Distributed Table Access
- **Range Partitioning**: Divides table ranges among workers
- **Coprocessor Integration**: Leverages TiKV coprocessor pushdown
- **Result Streaming**: Streams results for memory efficiency

### 5.3 Union Executor (`pkg/executor/unionexec/`)

#### Set Operations
- **UNION**: Combines results with duplicate elimination
- **UNION ALL**: Simple concatenation of results
- **Parallel Processing**: Processes branches concurrently

## 6. Performance Optimization Techniques

### 6.1 Vectorization
- **Batch Processing**: Processes chunks of rows together
- **SIMD Instructions**: Utilizes CPU vector instructions
- **Column-wise Operations**: Optimizes memory access patterns

### 6.2 Memory Management
- **Memory Pools**: Reuses allocated memory structures
- **Spill-to-Disk**: Graceful degradation under memory pressure
- **Garbage Collection**: Minimizes GC overhead through careful allocation

### 6.3 Parallelization Strategies
- **Data Parallelism**: Distributes data across workers
- **Pipeline Parallelism**: Overlaps computation stages
- **Adaptive Concurrency**: Adjusts parallelism based on workload

### 6.4 Cache Optimizations
- **Locality of Reference**: Optimizes memory access patterns
- **Prefetching**: Anticipates data access requirements
- **Cache-Conscious Algorithms**: Designs algorithms for cache efficiency

## Architecture Summary

TiDB's execution operators showcase several key design principles:

1. **Modularity**: Each operator is self-contained with clear interfaces
2. **Scalability**: Worker-based parallelism scales with available resources
3. **Robustness**: Comprehensive error handling and resource management
4. **Performance**: Vectorized execution with memory-conscious algorithms
5. **Adaptability**: Graceful degradation under resource constraints

The implementation demonstrates sophisticated understanding of modern database execution techniques, combining theoretical foundations with practical optimizations for real-world workloads.