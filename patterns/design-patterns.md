# TiDB Design Patterns and Architectural Principles

This document outlines the key design patterns and architectural principles used throughout the TiDB codebase. Understanding these patterns will help you write code that fits well with TiDB's architecture and maintains consistency with existing implementations.

## 🏗️ Core Architectural Patterns

### 1. Layered Architecture Pattern

TiDB follows a strict layered architecture with clear separation of concerns:

```
┌─────────────────────┐
│   Presentation      │ ← Server/Protocol Layer (pkg/server)
├─────────────────────┤
│   Application       │ ← Session/SQL Layer (pkg/session)  
├─────────────────────┤
│   Business Logic    │ ← Parser/Planner/Executor (pkg/parser, pkg/planner, pkg/executor)
├─────────────────────┤
│   Data Access       │ ← Storage Layer (pkg/store, pkg/kv)
├─────────────────────┤
│   Storage           │ ← TiKV/TiFlash (External components)
└─────────────────────┘
```

**Implementation Example**:
```go
// Each layer only depends on layers below it
package server // Presentation layer

import (
    "github.com/pingcap/tidb/pkg/session"  // ✓ Can depend on application layer
    // "github.com/pingcap/tidb/pkg/store"   // ✗ Cannot skip layers
)

package session // Application layer

import (
    "github.com/pingcap/tidb/pkg/executor"  // ✓ Can depend on business logic
    "github.com/pingcap/tidb/pkg/store"     // ✓ Can depend on data access
)
```

### 2. Interface Segregation Pattern

TiDB uses small, focused interfaces rather than large monolithic ones:

```go
// Small, focused interfaces
type Executor interface {
    Open(ctx context.Context) error
    Next(ctx context.Context, req *chunk.Chunk) error
    Close() error
}

type Retriever interface {
    Get(ctx context.Context, k Key) ([]byte, error)
}

type Mutator interface {
    Set(k Key, v []byte) error
    Delete(k Key) error
}

// Composed interface
type Transaction interface {
    Retriever
    Mutator
    Commit(ctx context.Context) error
    Rollback() error
}
```

### 3. Strategy Pattern for Algorithms

Different algorithms are implemented as strategies that can be swapped at runtime:

```go
// Join strategy interface
type JoinAlgorithm interface {
    Execute(ctx context.Context, leftChild, rightChild Executor) (*chunk.Chunk, error)
    EstimateCost(leftStats, rightStats *StatsInfo) float64
}

// Concrete strategies
type HashJoinAlgorithm struct {
    buildKeys  []expression.Expression
    probeKeys  []expression.Expression
}

type MergeJoinAlgorithm struct {
    leftKeys   []expression.Expression
    rightKeys  []expression.Expression
    ascending  []bool
}

type IndexJoinAlgorithm struct {
    outerChild Executor
    innerPlan  PhysicalPlan
}

// Context uses strategies
type JoinExecutor struct {
    algorithm JoinAlgorithm
}

func (e *JoinExecutor) Open(ctx context.Context) error {
    return e.algorithm.Execute(ctx, e.leftChild, e.rightChild)
}
```

### 4. Builder Pattern for Complex Objects

Complex objects like execution plans and requests are built using the builder pattern:

```go
// Plan builder
type PlanBuilder struct {
    ctx           sessionctx.Context
    is            infoschema.InfoSchema
    colMapper     map[*ast.ColumnNameExpr]int
    err           error
}

func (b *PlanBuilder) buildSelect(ctx context.Context, sel *ast.SelectStmt) (LogicalPlan, error) {
    // Build step by step
    p, err := b.buildResultSetNode(ctx, sel.From)
    if err != nil {
        return nil, err
    }
    
    if sel.Where != nil {
        p, err = b.buildSelection(ctx, p, sel.Where, nil)
        if err != nil {
            return nil, err
        }
    }
    
    if sel.GroupBy != nil {
        p, err = b.buildAggregation(ctx, p, sel.GroupBy.Items, sel.Having)
        if err != nil {
            return nil, err
        }
    }
    
    return b.buildProjection(ctx, p, sel.Fields.Fields, nil)
}

// Request builder
type RequestBuilder struct {
    kv.Request
    err error
}

func (b *RequestBuilder) SetKeyRanges(keyRanges []kv.KeyRange) *RequestBuilder {
    if b.err != nil {
        return b
    }
    b.KeyRanges = keyRanges
    return b
}

func (b *RequestBuilder) SetDAGRequest(dag *tipb.DAGRequest) *RequestBuilder {
    if b.err != nil {
        return b
    }
    data, err := dag.Marshal()
    if err != nil {
        b.err = err
        return b
    }
    b.Data = data
    return b
}

func (b *RequestBuilder) Build() (*kv.Request, error) {
    if b.err != nil {
        return nil, b.err
    }
    return &b.Request, nil
}
```

## 🔄 Behavioral Patterns

### 1. Iterator Pattern (Volcano Model)

TiDB uses the Volcano iterator model for query execution:

```go
// Iterator interface
type Executor interface {
    Open(ctx context.Context) error
    Next(ctx context.Context, req *chunk.Chunk) error
    Close() error
}

// Concrete iterator
type ProjectionExec struct {
    baseExecutor
    evaluators []expression.Evaluator
}

func (e *ProjectionExec) Next(ctx context.Context, req *chunk.Chunk) error {
    // Get input from child iterator
    err := Next(ctx, e.children[0], e.inputChunk)
    if err != nil {
        return err
    }
    
    // Process current batch
    err = e.evaluateProjection(e.inputChunk, req)
    return err
}

// Usage pattern
func executeQuery(executor Executor) error {
    if err := executor.Open(ctx); err != nil {
        return err
    }
    defer executor.Close()
    
    chunk := chunk.NewChunkWithCapacity(executor.Schema().Columns, 1024)
    for {
        err := executor.Next(ctx, chunk)
        if err != nil {
            return err
        }
        if chunk.NumRows() == 0 {
            break // End of data
        }
        
        // Process chunk
        processChunk(chunk)
        chunk.Reset()
    }
    
    return nil
}
```

### 2. Visitor Pattern for AST Processing

The visitor pattern is extensively used for AST traversal and transformation:

```go
// Visitor interface
type Visitor interface {
    Enter(n Node) (node Node, skipChildren bool)
    Leave(n Node) (node Node, ok bool)
}

// AST node interface
type Node interface {
    Accept(v Visitor) (node Node, ok bool)
    // ... other methods
}

// Concrete visitor for column name resolution
type columnNameResolver struct {
    schemas []*expression.Schema
    names   [][]*types.FieldName
    err     error
}

func (c *columnNameResolver) Enter(n Node) (Node, bool) {
    switch x := n.(type) {
    case *ast.ColumnNameExpr:
        // Resolve column name
        column, err := c.resolveColumnName(x)
        if err != nil {
            c.err = err
            return n, true
        }
        return column, true
    }
    return n, false
}

func (c *columnNameResolver) Leave(n Node) (Node, bool) {
    return n, c.err == nil
}

// Usage
func resolveColumns(expr ast.ExprNode, schemas []*expression.Schema) (ast.ExprNode, error) {
    resolver := &columnNameResolver{schemas: schemas}
    node, ok := expr.Accept(resolver)
    if !ok {
        return nil, resolver.err
    }
    return node.(ast.ExprNode), nil
}
```

### 3. Chain of Responsibility for Optimization Rules

Optimization rules are applied in chains:

```go
// Rule interface
type logicalOptRule interface {
    optimize(context.Context, LogicalPlan) (LogicalPlan, error)
    name() string
}

// Rule chain
var optRuleList = []logicalOptRule{
    &columnPruner{},
    &buildKeySolver{},
    &decorrelateSolver{},
    &aggregationEliminator{},
    &projectionEliminator{},
    &maxMinEliminator{},
    &ppdSolver{},
    &outerJoinEliminator{},
    &partitionProcessor{},
    &aggregationPushDownSolver{},
    &pushDownTopNOptimizer{},
    &joinReOrderSolver{},
}

// Apply rules in sequence
func logicalOptimize(ctx context.Context, flag uint64, logic LogicalPlan) (LogicalPlan, error) {
    for i, rule := range optRuleList {
        if flag&(1<<uint(i)) == 0 {
            continue
        }
        
        logic, err := rule.optimize(ctx, logic)
        if err != nil {
            return nil, err
        }
    }
    return logic, nil
}
```

### 4. Observer Pattern for Event Handling

TiDB uses the observer pattern for handling events like DDL completion:

```go
// Event interface
type Event interface {
    Type() EventType
    String() string
}

// Observer interface
type Observer interface {
    OnEvent(event Event) error
}

// Event dispatcher
type EventDispatcher struct {
    observers map[EventType][]Observer
    mu        sync.RWMutex
}

func (d *EventDispatcher) Register(eventType EventType, observer Observer) {
    d.mu.Lock()
    defer d.mu.Unlock()
    
    d.observers[eventType] = append(d.observers[eventType], observer)
}

func (d *EventDispatcher) Dispatch(event Event) error {
    d.mu.RLock()
    observers := d.observers[event.Type()]
    d.mu.RUnlock()
    
    for _, observer := range observers {
        if err := observer.OnEvent(event); err != nil {
            return err
        }
    }
    return nil
}

// Usage in DDL
type DDLWorker struct {
    dispatcher *EventDispatcher
}

func (w *DDLWorker) finishDDLJob(job *model.Job) error {
    // Complete DDL operation
    err := w.completeDDL(job)
    if err != nil {
        return err
    }
    
    // Notify observers
    event := &DDLCompleteEvent{Job: job}
    return w.dispatcher.Dispatch(event)
}
```

## 🔧 Creational Patterns

### 1. Factory Pattern for Executors

Executors are created using factory patterns:

```go
// Executor factory
type ExecutorBuilder struct {
    ctx           sessionctx.Context
    is            infoschema.InfoSchema
    snapshotTS    uint64
    err           error
}

func (b *ExecutorBuilder) build(p plannercore.Plan) Executor {
    switch v := p.(type) {
    case *plannercore.PhysicalProjection:
        return b.buildProjection(v)
    case *plannercore.PhysicalSelection:
        return b.buildSelection(v)
    case *plannercore.PhysicalTableScan:
        return b.buildTableReader(v)
    case *plannercore.PhysicalIndexScan:
        return b.buildIndexReader(v)
    case *plannercore.PhysicalHashJoin:
        return b.buildHashJoin(v)
    case *plannercore.PhysicalMergeJoin:
        return b.buildMergeJoin(v)
    // ... many other types
    default:
        b.err = ErrUnknownPlan.GenWithStack("Unknown Plan %T", p)
        return nil
    }
}

// Specific factory methods
func (b *ExecutorBuilder) buildProjection(v *plannercore.PhysicalProjection) Executor {
    childExec := b.build(v.Children()[0])
    if b.err != nil {
        return nil
    }
    
    return &ProjectionExec{
        baseExecutor: newBaseExecutor(b.ctx, v.Schema(), v.ID(), childExec),
        numWorkers:   v.GetConcurrency(),
        evaluators:   v.Exprs,
    }
}
```

### 2. Singleton Pattern for Global Resources

Some global resources use singleton pattern:

```go
// Statistics cache singleton
type statsCache struct {
    mu    sync.RWMutex
    cache map[int64]*statistics.Table
}

var (
    statsCacheInstance *statsCache
    statsCacheOnce     sync.Once
)

func GetStatsCache() *statsCache {
    statsCacheOnce.Do(func() {
        statsCacheInstance = &statsCache{
            cache: make(map[int64]*statistics.Table),
        }
    })
    return statsCacheInstance
}

// Domain singleton (one per cluster)
var (
    domainInstance *Domain
    domainMu       sync.Mutex
)

func GetDomain(store kv.Storage) *Domain {
    domainMu.Lock()
    defer domainMu.Unlock()
    
    if domainInstance == nil {
        domainInstance = NewDomain(store)
    }
    return domainInstance
}
```

### 3. Object Pool Pattern for Memory Management

TiDB uses object pools to reduce GC pressure:

```go
// Chunk pool for reusing memory
type ChunkPool struct {
    pools []*sync.Pool
}

func NewChunkPool() *ChunkPool {
    pools := make([]*sync.Pool, 6) // Different sizes
    for i := range pools {
        size := 1 << (i + 10) // 1KB, 2KB, 4KB, etc.
        pools[i] = &sync.Pool{
            New: func() interface{} {
                return chunk.NewChunkWithCapacity(nil, size)
            },
        }
    }
    return &ChunkPool{pools: pools}
}

func (p *ChunkPool) Get(size int) *chunk.Chunk {
    idx := p.sizeToIndex(size)
    return p.pools[idx].Get().(*chunk.Chunk)
}

func (p *ChunkPool) Put(c *chunk.Chunk) {
    c.Reset()
    idx := p.sizeToIndex(c.Capacity())
    p.pools[idx].Put(c)
}

// Usage
var chunkPool = NewChunkPool()

func (e *SomeExecutor) Next(ctx context.Context, req *chunk.Chunk) error {
    // Get chunk from pool
    tempChunk := chunkPool.Get(1024)
    defer chunkPool.Put(tempChunk)
    
    // Use chunk
    return e.processChunk(tempChunk, req)
}
```

## 📊 Data Patterns

### 1. MVCC Pattern for Concurrency Control

TiDB implements Multi-Version Concurrency Control:

```go
// Version interface
type Version interface {
    Cmp(Version) int
    String() string
}

// Versioned value
type VersionedValue struct {
    Value     []byte
    Version   Version
    ValueType ValueType
}

// MVCC key-value store
type MVCCStore interface {
    Get(key []byte, version Version) (*VersionedValue, error)
    Put(key []byte, value []byte, version Version) error
    Delete(key []byte, version Version) error
    Scan(start, end []byte, version Version) (Iterator, error)
}

// Transaction with snapshot isolation
type Transaction struct {
    startTS   Version
    commitTS  Version
    mutations map[string]*VersionedValue
}

func (txn *Transaction) Get(key []byte) ([]byte, error) {
    // Check local mutations first
    if mutation, exists := txn.mutations[string(key)]; exists {
        return mutation.Value, nil
    }
    
    // Read from snapshot at startTS
    vv, err := txn.store.Get(key, txn.startTS)
    if err != nil {
        return nil, err
    }
    
    return vv.Value, nil
}
```

### 2. Copy-on-Write Pattern for Schema Management

Schema objects use copy-on-write for thread safety:

```go
// Immutable schema
type Schema struct {
    columns    []*Column
    uniqueKeys [][]int
    keys       [][]int
    version    int64
}

// Schema builder for modifications
type SchemaBuilder struct {
    base    *Schema
    columns []*Column
    keys    [][]int
}

func (s *Schema) Copy() *SchemaBuilder {
    return &SchemaBuilder{
        base:    s,
        columns: append([]*Column(nil), s.columns...),
        keys:    append([][]int(nil), s.keys...),
    }
}

func (b *SchemaBuilder) AddColumn(col *Column) *SchemaBuilder {
    b.columns = append(b.columns, col)
    return b
}

func (b *SchemaBuilder) Build() *Schema {
    return &Schema{
        columns:    b.columns,
        keys:       b.keys,
        version:    b.base.version + 1,
    }
}

// Usage
func addColumnToSchema(schema *Schema, newCol *Column) *Schema {
    return schema.Copy().AddColumn(newCol).Build()
}
```

### 3. Lazy Loading Pattern for Statistics

Statistics are loaded lazily to improve performance:

```go
// Lazy-loaded statistics
type TableStats struct {
    tableMeta   *model.TableInfo
    histColl    *statistics.HistColl
    loaded      int32 // atomic flag
    loadChan    chan struct{}
}

func (ts *TableStats) GetHistogram(colID int64) (*statistics.Histogram, error) {
    // Check if already loaded
    if atomic.LoadInt32(&ts.loaded) == 1 {
        return ts.histColl.Columns[colID], nil
    }
    
    // Load statistics if not loaded
    ts.loadOnce()
    
    return ts.histColl.Columns[colID], nil
}

func (ts *TableStats) loadOnce() {
    if atomic.CompareAndSwapInt32(&ts.loaded, 0, 1) {
        // Load statistics from storage
        histColl, err := loadStatsFromStorage(ts.tableMeta.ID)
        if err != nil {
            atomic.StoreInt32(&ts.loaded, 0)
            return
        }
        
        ts.histColl = histColl
        close(ts.loadChan)
    } else {
        // Wait for other goroutine to finish loading
        <-ts.loadChan
    }
}
```

## 🚦 Error Handling Patterns

### 1. Error Wrapping Pattern

TiDB consistently wraps errors with context:

```go
import "github.com/pingcap/errors"

func (e *TableReaderExecutor) Open(ctx context.Context) error {
    kvReq, err := e.buildKVReq(ctx)
    if err != nil {
        return errors.Trace(err) // Add stack trace
    }
    
    result, err := e.SelectResult(ctx, e.ctx, kvReq, retTypes(e))
    if err != nil {
        return errors.Annotate(err, "failed to execute table reader") // Add context
    }
    
    e.resultHandler.open(result)
    return nil
}

// Custom error types
type PlannerError struct {
    Code    int
    Message string
    Cause   error
}

func (e *PlannerError) Error() string {
    if e.Cause != nil {
        return fmt.Sprintf("planner error %d: %s, caused by: %v", e.Code, e.Message, e.Cause)
    }
    return fmt.Sprintf("planner error %d: %s", e.Code, e.Message)
}

func NewPlannerError(code int, message string, cause error) *PlannerError {
    return &PlannerError{
        Code:    code,
        Message: message,
        Cause:   cause,
    }
}
```

### 2. Circuit Breaker Pattern

For handling failures in distributed operations:

```go
// Circuit breaker for TiKV connections
type CircuitBreaker struct {
    maxFailures   int
    resetTimeout  time.Duration
    state         int32 // 0: closed, 1: open, 2: half-open
    failures      int32
    lastFailTime  time.Time
    mu            sync.RWMutex
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    if !cb.allowRequest() {
        return ErrCircuitBreakerOpen
    }
    
    err := fn()
    cb.recordResult(err)
    return err
}

func (cb *CircuitBreaker) allowRequest() bool {
    cb.mu.RLock()
    defer cb.mu.RUnlock()
    
    state := atomic.LoadInt32(&cb.state)
    if state == 0 { // closed
        return true
    }
    
    if state == 1 { // open
        if time.Since(cb.lastFailTime) > cb.resetTimeout {
            atomic.CompareAndSwapInt32(&cb.state, 1, 2) // open -> half-open
            return true
        }
        return false
    }
    
    return true // half-open
}

func (cb *CircuitBreaker) recordResult(err error) {
    if err != nil {
        failures := atomic.AddInt32(&cb.failures, 1)
        if failures >= int32(cb.maxFailures) {
            cb.mu.Lock()
            cb.lastFailTime = time.Now()
            atomic.StoreInt32(&cb.state, 1) // -> open
            cb.mu.Unlock()
        }
    } else {
        atomic.StoreInt32(&cb.failures, 0)
        atomic.StoreInt32(&cb.state, 0) // -> closed
    }
}
```

## 🔄 Concurrency Patterns

### 1. Worker Pool Pattern

Used for parallel execution:

```go
// Worker pool for parallel table scanning
type TableScanWorkerPool struct {
    workers    []*tableScanWorker
    taskCh     chan *scanTask
    resultCh   chan *scanResult
    workerWg   sync.WaitGroup
    ctx        context.Context
    cancel     context.CancelFunc
}

type scanTask struct {
    ranges   []*ranger.Range
    executor *TableReaderExecutor
}

type scanResult struct {
    chunks []*chunk.Chunk
    err    error
}

func NewTableScanWorkerPool(concurrency int) *TableScanWorkerPool {
    ctx, cancel := context.WithCancel(context.Background())
    pool := &TableScanWorkerPool{
        workers:  make([]*tableScanWorker, concurrency),
        taskCh:   make(chan *scanTask, concurrency),
        resultCh: make(chan *scanResult, concurrency),
        ctx:      ctx,
        cancel:   cancel,
    }
    
    // Start workers
    for i := 0; i < concurrency; i++ {
        worker := &tableScanWorker{
            id:       i,
            taskCh:   pool.taskCh,
            resultCh: pool.resultCh,
        }
        pool.workers[i] = worker
        pool.workerWg.Add(1)
        go worker.run(ctx, &pool.workerWg)
    }
    
    return pool
}

func (pool *TableScanWorkerPool) Submit(task *scanTask) {
    select {
    case pool.taskCh <- task:
    case <-pool.ctx.Done():
    }
}

func (pool *TableScanWorkerPool) Results() <-chan *scanResult {
    return pool.resultCh
}

func (pool *TableScanWorkerPool) Close() {
    pool.cancel()
    close(pool.taskCh)
    pool.workerWg.Wait()
    close(pool.resultCh)
}
```

### 2. Pipeline Pattern

For streaming data processing:

```go
// Pipeline stages
type PipelineStage func(<-chan interface{}) <-chan interface{}

// Pipeline runner
type Pipeline struct {
    stages []PipelineStage
}

func NewPipeline(stages ...PipelineStage) *Pipeline {
    return &Pipeline{stages: stages}
}

func (p *Pipeline) Run(input <-chan interface{}) <-chan interface{} {
    output := input
    for _, stage := range p.stages {
        output = stage(output)
    }
    return output
}

// Example: Query execution pipeline
func createQueryPipeline() *Pipeline {
    return NewPipeline(
        parseStage,     // SQL -> AST
        planStage,      // AST -> LogicalPlan  
        optimizeStage,  // LogicalPlan -> PhysicalPlan
        executeStage,   // PhysicalPlan -> Results
    )
}

func parseStage(input <-chan interface{}) <-chan interface{} {
    output := make(chan interface{})
    go func() {
        defer close(output)
        for item := range input {
            if sql, ok := item.(string); ok {
                ast, err := parser.Parse(sql)
                if err != nil {
                    output <- err
                } else {
                    output <- ast
                }
            }
        }
    }()
    return output
}
```

## 🎯 Best Practices

### 1. Interface Design

```go
// ✓ Good: Small, focused interfaces
type Reader interface {
    Read() ([]byte, error)
}

type Writer interface {
    Write([]byte) error
}

type ReadWriter interface {
    Reader
    Writer
}

// ✗ Bad: Large, monolithic interface
type FileHandler interface {
    Read() ([]byte, error)
    Write([]byte) error
    Seek(int64) error
    Close() error
    Sync() error
    Truncate(int64) error
    // ... many more methods
}
```

### 2. Error Handling

```go
// ✓ Good: Wrap errors with context
func buildExecutor(plan PhysicalPlan) (Executor, error) {
    executor, err := createExecutor(plan)
    if err != nil {
        return nil, errors.Annotate(err, "failed to build executor for plan %s", plan.String())
    }
    return executor, nil
}

// ✗ Bad: Lose error context
func buildExecutor(plan PhysicalPlan) (Executor, error) {
    executor, err := createExecutor(plan)
    if err != nil {
        return nil, err // Lost context about what failed
    }
    return executor, nil
}
```

### 3. Resource Management

```go
// ✓ Good: Use defer for cleanup
func (e *Executor) processQuery(ctx context.Context) error {
    if err := e.Open(ctx); err != nil {
        return err
    }
    defer e.Close() // Always cleanup
    
    // Process query
    return e.execute(ctx)
}

// ✓ Good: Context-aware operations
func (e *Executor) execute(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err() // Respect cancellation
        default:
            if err := e.processNext(); err != nil {
                return err
            }
        }
    }
}
```

## 📚 Related Documentation

- [Code Walkthrough](../development/walkthrough.md) - See patterns in action
- [Performance Guidelines](performance.md) - Optimization patterns
- [Architecture Overview](../architecture/overview.md) - High-level design patterns
- [Contributing Guidelines](../guides/contributing.md) - Code style guidelines