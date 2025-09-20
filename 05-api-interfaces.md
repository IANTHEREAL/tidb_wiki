# TiDB API and Interfaces Documentation

## Overview

This document provides comprehensive documentation of TiDB's internal APIs and interfaces. Understanding these interfaces is crucial for developers working on TiDB core components, extensions, or debugging complex issues.

## Table of Contents

1. [Core Interfaces](#core-interfaces)
2. [Executor Interfaces](#executor-interfaces)
3. [Storage Interfaces](#storage-interfaces)
4. [Session and Context Interfaces](#session-and-context-interfaces)
5. [Expression System Interfaces](#expression-system-interfaces)
6. [Schema and Metadata Interfaces](#schema-and-metadata-interfaces)
7. [Plugin System Interfaces](#plugin-system-interfaces)
8. [Testing and Mock Interfaces](#testing-and-mock-interfaces)

## Core Interfaces

### Plan Interface Hierarchy

The planning system uses a hierarchical interface design to represent both logical and physical execution plans.

```go
// Base plan interface - foundation for all plan types
type Plan interface {
    // Identification and metadata
    ID() int
    TP() string
    String() string
    Schema() *expression.Schema

    // Tree operations
    Children() []Plan
    SetChildren(...Plan)

    // Context and statistics
    StatsInfo() *property.StatsInfo
    SetStatsInfo(*property.StatsInfo)

    // Type checking and validation
    SCtx() sessionctx.Context
}

// Logical plan interface for optimization phase
type LogicalPlan interface {
    Plan

    // Optimization operations
    PredicatePushDown([]expression.Expression) ([]expression.Expression, LogicalPlan)
    PruneColumns([]*expression.Column) error
    FindBestTask(prop *property.PhysicalProperty) (task, error)
    DeriveStats(childStats []*property.StatsInfo) (*property.StatsInfo, error)

    // Join ordering and rewriting
    ExtractEQAndInCondition() []*expression.ScalarFunction
    MaxOneRow() bool
    OutputNames() types.NameSlice
    SetOutputNames(names types.NameSlice)
}

// Physical plan interface for execution phase
type PhysicalPlan interface {
    Plan

    // Cost and properties
    Cost() float64
    SetCost(float64)
    Clone() PhysicalPlan

    // Memory and resource estimation
    MemoryUsage() (int64, int64) // (min, max) memory usage

    // Execution properties
    GetEstRowCountForDisplay() float64
    ToPB(ctx sessionctx.Context) (*tipb.Executor, error)

    // Attach and detach children
    Attach2Task(...task) task
}
```

#### Plan Implementation Examples

```go
// Logical join plan
type LogicalJoin struct {
    LogicalSchemaProducer

    JoinType JoinType
    OnConditions []expression.Expression
    LeftConditions []expression.Expression
    RightConditions []expression.Expression
    EqualConditions []*expression.ScalarFunction
    NAEqualConditions []expression.Expression

    // Join hints and preferences
    HintInfo *tableHintInfo
    PreferJoinType uint
    PreferJoinOrder bool
}

// Logical join optimization methods
func (p *LogicalJoin) PredicatePushDown(predicates []expression.Expression) ([]expression.Expression, LogicalPlan) {
    // Implementation splits predicates based on join type and conditions
    // Returns remaining predicates and modified plan
}

func (p *LogicalJoin) PruneColumns(parentUsedCols []*expression.Column) error {
    // Determines which columns are needed from children
    // Prunes unnecessary columns to reduce data movement
}

func (p *LogicalJoin) FindBestTask(prop *property.PhysicalProperty) (task, error) {
    // Generates all possible physical join algorithms
    // Estimates costs and selects optimal implementation
    hashJoinTask := p.getHashJoinTask(prop)
    mergeJoinTask := p.getMergeJoinTask(prop)
    nestedLoopTask := p.getNestedLoopTask(prop)

    return cheapestTask(hashJoinTask, mergeJoinTask, nestedLoopTask), nil
}

// Physical hash join plan
type PhysicalHashJoin struct {
    PhysicalJoin

    Concurrency int
    EqualConditions []*expression.ScalarFunction
    NAEqualConditions []expression.Expression
    UseOuterToBuild bool

    // Resource estimation
    EstimatedMemoryUsage int64
}

func (p *PhysicalHashJoin) Cost() float64 {
    // Cost model based on:
    // - Build side cardinality and memory usage
    // - Probe side cardinality and CPU cost
    // - Hash function overhead
    // - Memory access patterns
    buildCost := p.children[p.buildSideIndex()].Cost()
    probeCost := p.children[1-p.buildSideIndex()].Cost()

    hashCost := p.buildSideCardinality() * hashCostFactor
    joinCost := p.probeSideCardinality() * probeCostFactor

    return buildCost + probeCost + hashCost + joinCost
}
```

## Executor Interfaces

### Base Executor Framework

The executor system uses a pull-based iterator model with vectorized processing capabilities.

```go
// Core executor interface
type Executor interface {
    // Lifecycle management
    Open(ctx context.Context) error
    Next(ctx context.Context, req *chunk.Chunk) error
    Close() error

    // Schema information
    Schema() *expression.Schema
}

// Extended executor interface with additional capabilities
type ExecNode interface {
    Executor

    // Resource tracking
    SetMemTracker(*memory.Tracker)
    SetDiskTracker(*disk.Tracker)

    // Runtime statistics
    RuntimeStats() *execdetails.RuntimeStats
    SetRuntimeStats(*execdetails.RuntimeStats)
}

// Base executor implementation providing common functionality
type baseExecutor struct {
    ctx           sessionctx.Context
    id            int
    schema        *expression.Schema
    initCap       int
    maxChunkSize  int
    children      []Executor
    retFieldTypes []*types.FieldType
    runtimeStats  *execdetails.RuntimeStats
}

// Common executor operations
func (e *baseExecutor) Open(ctx context.Context) error {
    // Initialize children executors
    for _, child := range e.children {
        if err := child.Open(ctx); err != nil {
            return err
        }
    }
    return nil
}

func (e *baseExecutor) Close() error {
    // Cleanup children executors in reverse order
    for i := len(e.children) - 1; i >= 0; i-- {
        if err := e.children[i].Close(); err != nil {
            return err
        }
    }
    return nil
}

func (e *baseExecutor) Schema() *expression.Schema {
    return e.schema
}

func (e *baseExecutor) ID() int {
    return e.id
}
```

### Specialized Executor Interfaces

#### Join Executor Interface
```go
// Join-specific executor interface
type JoinExecutor interface {
    Executor

    // Join configuration
    SetJoinType(JoinType)
    SetOnConditions([]expression.Expression)
    SetEqualConditions([]*expression.ScalarFunction)

    // Performance tuning
    SetConcurrency(int)
    SetUseOuterToBuild(bool)

    // Memory management
    SetMemoryLimit(int64)
    SetSpillToDisk(bool)
}

// Hash join specific interface
type HashJoinExecutor interface {
    JoinExecutor

    // Hash join specific operations
    BuildHashTable(ctx context.Context) error
    ProbeHashTable(ctx context.Context, input *chunk.Chunk) (*chunk.Chunk, error)

    // Partitioning for large data sets
    EnablePartitioning(bool)
    SetPartitionCount(int)
}

// Implementation example
type HashJoinExec struct {
    baseExecutor

    // Join configuration
    joinType     JoinType
    isOuterJoin  bool
    useOuterToBuild bool

    // Join conditions
    onConditions    []expression.Expression
    equalConditions []*expression.ScalarFunction
    otherConditions []expression.Expression

    // Execution state
    prepared     bool
    hashTable    *hashRowContainer
    buildFinished chan error

    // Resource management
    memTracker   *memory.Tracker
    diskTracker  *disk.Tracker
    concurrency  int

    // Worker coordination
    buildWorker  *hashJoinBuildWorker
    probeWorkers []*hashJoinProbeWorker
    workerWg     sync.WaitGroup
}

func (e *HashJoinExec) SetJoinType(jt JoinType) {
    e.joinType = jt
    e.isOuterJoin = (jt == LeftOuterJoin || jt == RightOuterJoin || jt == FullOuterJoin)
}

func (e *HashJoinExec) BuildHashTable(ctx context.Context) error {
    buildSideExec := e.children[e.buildSideIndex()]

    // Start build worker
    e.buildWorker = &hashJoinBuildWorker{
        HashJoinExec: e,
        buildSideExec: buildSideExec,
        hashTable:    e.hashTable,
    }

    go func() {
        defer close(e.buildFinished)
        e.buildFinished <- e.buildWorker.run(ctx)
    }()

    return nil
}
```

#### Aggregation Executor Interface
```go
// Aggregation executor interface
type AggregationExecutor interface {
    Executor

    // Aggregation configuration
    SetAggFuncs([]aggfuncs.AggFunc)
    SetGroupByExpressions([]expression.Expression)

    // Execution modes
    SetStreamAggMode(bool)      // Stream aggregation for ordered input
    SetHashAggMode(bool)        // Hash aggregation for unordered input
    SetParallelExecution(bool)  // Enable parallel aggregation

    // Memory management
    SetMemoryLimit(int64)
    SetSpillStrategy(SpillStrategy)
}

// Aggregation function interface
type AggFunc interface {
    // Partial aggregation (for parallel execution)
    UpdatePartialResult(sctx sessionctx.Context, rowsInGroup []chunk.Row, pr PartialResult) error
    MergePartialResult(sctx sessionctx.Context, src, dst PartialResult) error

    // Final result generation
    AppendFinalResult2Chunk(sctx sessionctx.Context, pr PartialResult, chk *chunk.Chunk) error

    // Memory management
    AllocPartialResult() PartialResult
    ResetPartialResult(PartialResult)

    // Function metadata
    GetType() *types.FieldType
    Clone() AggFunc
}

// Count aggregation function example
type countOriginal struct {
    baseAggFunc
}

func (c *countOriginal) UpdatePartialResult(sctx sessionctx.Context, rowsInGroup []chunk.Row, pr PartialResult) error {
    p := (*partialResult4Count)(pr)
    for _, row := range rowsInGroup {
        if c.args[0].EvalNullable() {
            // COUNT(column) - skip NULL values
            val, isNull, err := c.args[0].EvalInt(sctx, row)
            if err != nil {
                return err
            }
            if !isNull {
                p.count++
            }
        } else {
            // COUNT(*) - count all rows
            p.count++
        }
    }
    return nil
}

func (c *countOriginal) AppendFinalResult2Chunk(sctx sessionctx.Context, pr PartialResult, chk *chunk.Chunk) error {
    p := (*partialResult4Count)(pr)
    chk.AppendInt64(0, p.count)
    return nil
}
```

## Storage Interfaces

### Key-Value Storage Interface

The storage layer provides a transactional key-value interface that abstracts over TiKV.

```go
// Main storage interface
type Storage interface {
    // Transaction lifecycle
    Begin() (Transaction, error)
    BeginWithStartTS(startTS uint64) (Transaction, error)
    GetSnapshot(ver Version) Snapshot

    // Metadata operations
    CurrentVersion(txnScope string) (Version, error)
    GetOracle() oracle.Oracle

    // Resource management
    Close() error
    UUID() string

    // Configuration
    GetCodec() tikv.Codec
    GetTiKVClient() tikv.Client
}

// Transaction interface for ACID operations
type Transaction interface {
    // Basic operations
    Get(ctx context.Context, k Key) ([]byte, error)
    BatchGet(ctx context.Context, keys []Key) (map[string][]byte, error)
    Iter(k Key, upperBound Key) (Iterator, error)
    IterReverse(k Key) (Iterator, error)

    // Mutation operations
    Set(k Key, v []byte) error
    Delete(k Key) error

    // Batch operations for efficiency
    BatchSet(keys []Key, values [][]byte) error
    BatchDelete(keys []Key) error

    // Transaction control
    Commit(ctx context.Context) error
    Rollback() error

    // State inspection
    Size() int
    Len() int
    GetMemBuffer() MemBuffer
    GetSnapshot() Snapshot

    // Lock management
    LockKeys(ctx context.Context, keys ...Key) error

    // Properties
    StartTS() uint64
    Valid() bool
    GetUnionStore() UnionStore
}

// Snapshot interface for consistent reads
type Snapshot interface {
    // Read operations (no modifications)
    Get(ctx context.Context, k Key) ([]byte, error)
    BatchGet(ctx context.Context, keys []Key) (map[string][]byte, error)
    Iter(k Key, upperBound Key) (Iterator, error)
    IterReverse(k Key) (Iterator, error)

    // Snapshot properties
    GetTS() uint64
    SetOption(opt int, val interface{})
    DelOption(opt int)

    // Statistics and debugging
    SetPriority(priority int)
    SetNotFillCache(notFillCache bool)
    SetKeyOnly(keyOnly bool)
}

// Iterator interface for range scans
type Iterator interface {
    // Navigation
    Valid() bool
    Key() Key
    Value() []byte
    Next() error
    Close()

    // Bulk operations
    GetRowKeyCount() int64
}
```

#### Implementation Examples

```go
// TiKV storage implementation
type tikvStore struct {
    client     tikv.Client
    pdClient   pd.Client
    regionCache *tikv.RegionCache
    lockResolver *tikv.LockResolver
    uuid       string
    oracle     oracle.Oracle
    clientMu   sync.RWMutex
    closed     bool
}

func (s *tikvStore) Begin() (Transaction, error) {
    startTS, err := s.oracle.GetTimestamp(context.Background())
    if err != nil {
        return nil, err
    }
    return s.BeginWithStartTS(startTS)
}

func (s *tikvStore) BeginWithStartTS(startTS uint64) (Transaction, error) {
    snapshot := s.GetSnapshot(NewVersion(startTS))
    return &tikvTxn{
        snapshot:    snapshot,
        us:          newUnionStore(snapshot),
        store:       s,
        startTS:     startTS,
        valid:       true,
        lockKeys:    make([][]byte, 0),
        mutations:   make(map[string]*pb.Mutation),
        memBufCap:   defaultTxnMembufCap,
    }, nil
}

func (s *tikvStore) GetSnapshot(ver Version) Snapshot {
    return &tikvSnapshot{
        store:    s,
        version:  ver,
        priority: tikv.PriorityNormal,
        vars:     make(map[tikv.Option]interface{}),
    }
}

// Transaction implementation
type tikvTxn struct {
    snapshot  Snapshot
    us        UnionStore    // Local read/write buffer
    store     *tikvStore
    startTS   uint64
    commitTS  uint64
    valid     bool
    lockKeys  [][]byte
    mutations map[string]*pb.Mutation
    memBufCap int
    mu        sync.Mutex
}

func (txn *tikvTxn) Get(ctx context.Context, k Key) ([]byte, error) {
    // First check local buffer
    val, err := txn.us.Get(k)
    if err == nil {
        return val, nil
    }

    if !kv.IsErrNotFound(err) {
        return nil, err
    }

    // Fall back to snapshot read
    return txn.snapshot.Get(ctx, k)
}

func (txn *tikvTxn) Set(k Key, v []byte) error {
    txn.mu.Lock()
    defer txn.mu.Unlock()

    if !txn.valid {
        return kv.ErrInvalidTxn
    }

    // Add to local buffer
    return txn.us.Set(k, v)
}

func (txn *tikvTxn) Commit(ctx context.Context) error {
    if !txn.valid {
        return kv.ErrInvalidTxn
    }

    defer func() {
        txn.valid = false
    }()

    // Check if transaction is read-only
    if txn.us.Size() == 0 {
        return nil
    }

    // Execute two-phase commit
    committer := newTwoPhaseCommitter(txn, 0)
    return committer.execute(ctx)
}
```

### Memory Buffer Interface

```go
// MemBuffer interface for transaction local storage
type MemBuffer interface {
    // Basic operations
    Get(k Key) ([]byte, error)
    Set(k Key, v []byte) error
    Delete(k Key) error

    // Iteration
    Iter(k Key, upperBound Key) (Iterator, error)
    IterReverse(k Key) (Iterator, error)

    // Bulk operations
    GetMutations() map[string]*pb.Mutation
    GetKeyLengths() []int

    // Buffer management
    Size() int
    Len() int
    Reset()
    SetCap(cap int)

    // Snapshots for nested transactions
    Checkpoint() *MemDBCheckpoint
    RevertToCheckpoint(*MemDBCheckpoint)
    ReleaseCheckpoint(*MemDBCheckpoint)
}

// Union store combining snapshot and buffer
type UnionStore interface {
    MemBuffer

    // Access underlying components
    GetSnapshot() Snapshot
    GetMemBuffer() MemBuffer

    // Cache management
    SetOption(opt int, val interface{})
    DelOption(opt int)
}
```

## Session and Context Interfaces

### Session Context Interface

The session context provides access to session state, variables, and execution context.

```go
// Main session context interface
type Context interface {
    // Session state
    GetSessionVars() *variable.SessionVars
    GetInfoSchema() infoschema.InfoSchema
    GetStore() kv.Storage
    GetClient() kv.Client

    // Transaction management
    Txn(active bool) (kv.Transaction, error)
    GetTxnWriteThroughputSLI() *sli.TxnWriteThroughputSLI

    // Execution context
    GetExprCtx() exprctx.ExprContext
    GetEvalCtx() exprctx.EvalContext
    GetBuildPBCtx() exprctx.BuildPBContext

    // Resource tracking
    GetMemoryUsage() int64
    GetDiskUsage() int64

    // Error handling
    HandleError(error) error

    // SQL execution
    Execute(ctx context.Context, sql string) ([]ast.RecordSet, error)
    ExecuteStmt(ctx context.Context, stmt ast.StmtNode) (ast.RecordSet, error)

    // Privilege checking
    GetPrivilegeManager() privilege.Manager

    // Statistics
    UpdateStatsUsage(usage []int64)
    GetStatsUsage() []int64
}

// Expression context for evaluation
type ExprContext interface {
    // Type system
    GetEvalCtx() EvalContext
    GetCharsetInfo() (string, string)  // charset, collation
    GetDefaultCollationForUTF8MB4() string

    // Session information
    GetSQLMode() mysql.SQLMode
    GetTimeZone() *time.Location
    GetSystemVar(name string) (string, bool)

    // Function resolution
    GetOptionalPropSet() OptionalPropSet
    GetOptionalPropProvider(OptionalPropKey) (OptionalPropProvider, bool)
}

// Evaluation context for expression evaluation
type EvalContext interface {
    // Current state
    CurrentTime() (time.Time, error)
    GetSessionVars() *SessionVars

    // Type conversion
    TypeCtx() types.Context
    ErrCtx() errctx.Context

    // Location and timezone
    Location() *time.Location
    AppendWarning(error)

    // Random number generation
    Rng() *rand.Rand
}
```

#### Implementation Examples

```go
// Session implementation
type session struct {
    // Core session state
    sessionVars    *variable.SessionVars
    store          kv.Storage
    parser         *parser.Parser
    preprocessor   core.Preprocessor

    // Transaction management
    txn            kv.Transaction
    txnFuture      oracle.Future

    // Statement execution
    stmtCtx        *stmtctx.StatementContext

    // Resource tracking
    memTracker     *memory.Tracker
    diskTracker    *disk.Tracker

    // Concurrency control
    mu             sync.RWMutex

    // Plugins and extensions
    plugins        map[string]Plugin
}

func (s *session) Execute(ctx context.Context, sql string) ([]ast.RecordSet, error) {
    // Parse SQL statements
    stmts, warns, err := s.parser.Parse(sql, s.sessionVars.GetCharsetInfo())
    if err != nil {
        return nil, err
    }

    // Handle warnings
    for _, warn := range warns {
        s.sessionVars.StmtCtx.AppendWarning(warn)
    }

    var recordSets []ast.RecordSet

    // Execute each statement
    for _, stmt := range stmts {
        recordSet, err := s.executeStatement(ctx, stmt)
        if err != nil {
            return nil, err
        }
        if recordSet != nil {
            recordSets = append(recordSets, recordSet)
        }
    }

    return recordSets, nil
}

func (s *session) executeStatement(ctx context.Context, stmt ast.StmtNode) (ast.RecordSet, error) {
    // Preprocess statement
    preprocessedStmt, err := s.preprocessor.Preprocess(ctx, stmt, s.GetInfoSchema())
    if err != nil {
        return nil, err
    }

    // Build execution plan
    compiler := executor.Compiler{Ctx: s}
    plan, err := compiler.Compile(ctx, preprocessedStmt)
    if err != nil {
        return nil, err
    }

    // Execute plan
    return s.executePlan(ctx, plan)
}
```

### Statement Context Interface

```go
// Statement context for execution tracking
type StatementContext interface {
    // Warning management
    AppendWarning(error)
    GetWarnings() []error
    WarningCount() uint16
    TruncateWarnings(start int)

    // Error context
    HandleTruncate(error) error
    HandleOverflow(error, error) error

    // Execution flags
    IgnoreTruncate() bool
    IgnoreZeroInDate() bool
    NoZeroDate() bool

    // Statistics
    AddAffectedRows(int64)
    GetAffectedRows() uint64
    AddFoundRows(int64)
    GetFoundRows() uint64

    // Memory tracking
    SetMemTracker(*memory.Tracker)
    GetMemTracker() *memory.Tracker

    // Runtime information
    GetExecDetails() *execdetails.ExecDetails
    SetExecDetails(*execdetails.ExecDetails)
}

// Implementation
type StatementContext struct {
    // Warning and error tracking
    warnings         []SQLWarn
    warningCount     uint16
    histogramsNotLoad bool
    mu               sync.Mutex

    // Execution flags
    ignoreNoPartition  bool
    ignoreTruncate     bool
    ignoreZeroInDate   bool
    noZeroDate         bool
    strictSQLMode      bool

    // Affected rows tracking
    affectedRows      uint64
    foundRows         uint64
    updatedRows       uint64
    copiedRows        uint64
    touchedRows       uint64
    deletedRows       uint64
    insertedRows      uint64

    // Memory and resource tracking
    memTracker        *memory.Tracker
    diskTracker       *disk.Tracker

    // Runtime details
    execDetails       execdetails.ExecDetails
    allExecDetails    []*execdetails.ExecDetails
}
```

## Expression System Interfaces

### Expression Interface Hierarchy

```go
// Base expression interface
type Expression interface {
    // Evaluation methods
    Eval(row chunk.Row) (types.Datum, error)
    EvalInt(ctx sessionctx.Context, row chunk.Row) (int64, bool, error)
    EvalReal(ctx sessionctx.Context, row chunk.Row) (float64, bool, error)
    EvalString(ctx sessionctx.Context, row chunk.Row) (string, bool, error)
    EvalTime(ctx sessionctx.Context, row chunk.Row) (types.Time, bool, error)
    EvalDuration(ctx sessionctx.Context, row chunk.Row) (types.Duration, bool, error)
    EvalJSON(ctx sessionctx.Context, row chunk.Row) (types.BinaryJSON, bool, error)

    // Vectorized evaluation
    VecEvalInt(ctx sessionctx.Context, input *chunk.Chunk, result *chunk.Column) error
    VecEvalReal(ctx sessionctx.Context, input *chunk.Chunk, result *chunk.Column) error
    VecEvalString(ctx sessionctx.Context, input *chunk.Chunk, result *chunk.Column) error
    VecEvalTime(ctx sessionctx.Context, input *chunk.Chunk, result *chunk.Column) error
    VecEvalDuration(ctx sessionctx.Context, input *chunk.Chunk, result *chunk.Column) error
    VecEvalJSON(ctx sessionctx.Context, input *chunk.Chunk, result *chunk.Column) error

    // Type information
    GetType() *types.FieldType
    SetType(*types.FieldType)

    // Expression properties
    Clone() Expression
    Equal(ctx sessionctx.Context, e Expression) bool
    IsCorrelated() bool
    Decorrelate(schema *Schema) Expression

    // Hash and comparison
    HashCode(sc *stmtctx.StatementContext) []byte

    // Utility methods
    ExplainInfo() string
    CanonicalHashCode(sc *stmtctx.StatementContext) []byte
}

// Scalar function interface
type ScalarFunction interface {
    Expression

    // Function metadata
    FuncName() model.CIStr
    GetArgs() []Expression
    SetArgs([]Expression)

    // Folding and optimization
    GetCtx() sessionctx.Context
    Clone() *ScalarFunction

    // Function signature
    GetSig() builtinFunc
}

// Built-in function signature interface
type builtinFunc interface {
    // Type-specific evaluation
    evalInt(row chunk.Row) (int64, bool, error)
    evalReal(row chunk.Row) (float64, bool, error)
    evalString(row chunk.Row) (string, bool, error)
    evalTime(row chunk.Row) (types.Time, bool, error)
    evalDuration(row chunk.Row) (types.Duration, bool, error)
    evalJSON(row chunk.Row) (types.BinaryJSON, bool, error)

    // Vectorized evaluation
    vecEvalInt(input *chunk.Chunk, result *chunk.Column) error
    vecEvalReal(input *chunk.Chunk, result *chunk.Column) error
    vecEvalString(input *chunk.Chunk, result *chunk.Column) error
    vecEvalTime(input *chunk.Chunk, result *chunk.Column) error
    vecEvalDuration(input *chunk.Chunk, result *chunk.Column) error
    vecEvalJSON(input *chunk.Chunk, result *chunk.Column) error

    // Function properties
    Clone() builtinFunc
    getArgs() []Expression
    getRetTp() *types.FieldType

    // Optimization support
    canFold() bool
    fold(datum types.Datum) (types.Datum, error)
}
```

#### Expression Implementation Examples

```go
// Arithmetic addition function
type builtinArithmeticPlusRealSig struct {
    baseBuiltinFunc
}

func (b *builtinArithmeticPlusRealSig) evalReal(row chunk.Row) (float64, bool, error) {
    // Evaluate left operand
    left, isNull, err := b.args[0].EvalReal(b.ctx, row)
    if err != nil || isNull {
        return 0, isNull, err
    }

    // Evaluate right operand
    right, isNull, err := b.args[1].EvalReal(b.ctx, row)
    if err != nil || isNull {
        return 0, isNull, err
    }

    // Perform addition with overflow checking
    result := left + right
    if math.IsInf(result, 0) {
        return 0, false, types.ErrOverflow.GenWithStackByArgs("DOUBLE", fmt.Sprintf("(%s + %s)", left, right))
    }

    return result, false, nil
}

func (b *builtinArithmeticPlusRealSig) vecEvalReal(input *chunk.Chunk, result *chunk.Column) error {
    // Vectorized evaluation for performance
    n := input.NumRows()

    // Prepare result column
    result.ResizeFloat64(n, false)

    // Get left and right operand columns
    leftCol, err := b.bufAllocator.get(types.ETReal, n)
    if err != nil {
        return err
    }
    defer b.bufAllocator.put(leftCol)

    if err := b.args[0].VecEvalReal(b.ctx, input, leftCol); err != nil {
        return err
    }

    rightCol, err := b.bufAllocator.get(types.ETReal, n)
    if err != nil {
        return err
    }
    defer b.bufAllocator.put(rightCol)

    if err := b.args[1].VecEvalReal(b.ctx, input, rightCol); err != nil {
        return err
    }

    // Perform vectorized addition
    leftData := leftCol.Float64s()
    rightData := rightCol.Float64s()
    resultData := result.Float64s()

    for i := 0; i < n; i++ {
        if leftCol.IsNull(i) || rightCol.IsNull(i) {
            result.SetNull(i, true)
            continue
        }

        sum := leftData[i] + rightData[i]
        if math.IsInf(sum, 0) {
            return types.ErrOverflow.GenWithStackByArgs("DOUBLE", fmt.Sprintf("(%g + %g)", leftData[i], rightData[i]))
        }

        resultData[i] = sum
    }

    return nil
}

// Column reference expression
type Column struct {
    UniqueID int64
    ID       int64
    Index    int
    RetType  *types.FieldType

    // Correlation information
    IsCorrelated bool
    IsReferenced bool

    // Original information
    OrigName     string
    OrigTblName  model.CIStr
    DBName       model.CIStr
    TblName      model.CIStr
    ColName      model.CIStr
}

func (col *Column) EvalInt(ctx sessionctx.Context, row chunk.Row) (int64, bool, error) {
    if col.Index < 0 || col.Index >= row.Len() {
        return 0, false, errors.Errorf("column index %d out of range", col.Index)
    }

    datum := row.GetDatum(col.Index, col.RetType)
    if datum.IsNull() {
        return 0, true, nil
    }

    return datum.GetInt64(), false, nil
}

func (col *Column) VecEvalInt(ctx sessionctx.Context, input *chunk.Chunk, result *chunk.Column) error {
    if col.Index < 0 || col.Index >= input.NumCols() {
        return errors.Errorf("column index %d out of range", col.Index)
    }

    // Direct column reference - copy from input
    srcCol := input.Column(col.Index)
    result.CopyConstruct(srcCol)

    return nil
}
```

## Schema and Metadata Interfaces

### Information Schema Interface

```go
// Information schema interface for metadata access
type InfoSchema interface {
    // Schema operations
    SchemaByName(schema model.CIStr) (*model.DBInfo, bool)
    SchemaExists(schema model.CIStr) bool
    TableByName(schema, table model.CIStr) (table.Table, error)
    TableExists(schema, table model.CIStr) bool

    // Table operations
    TableByID(id int64) (table.Table, bool)
    AllocByID(id int64) (autoid.Allocator, bool)

    // Schema version
    SchemaMetaVersion() int64

    // Iteration
    AllSchemas() []*model.DBInfo
    AllSchemaNames() []string

    // Clone and modification
    Clone() InfoSchema

    // Policy and privilege information
    PolicyByName(name model.CIStr) (*model.PolicyInfo, bool)
    ResourceGroupByName(name model.CIStr) (*model.ResourceGroupInfo, bool)
}

// Table interface for table metadata and operations
type Table interface {
    // Metadata
    Meta() *model.TableInfo
    Type() table.Type
    GetPhysicalID() int64

    // Column operations
    Cols() []*table.Column
    VisibleCols() []*table.Column
    HiddenCols() []*table.Column
    WarpColTypes(schema []types.FieldType) []types.FieldType

    // Index operations
    Indices() []table.Index
    WritableIndices() []table.Index
    DeletableIndices() []table.Index

    // Row operations
    AddRecord(ctx sessionctx.Context, r []types.Datum, opts ...table.AddRecordOption) (recordID kv.Handle, err error)
    UpdateRecord(ctx context.Context, sctx sessionctx.Context, h kv.Handle, currData, newData []types.Datum, touched []bool) error
    RemoveRecord(ctx sessionctx.Context, h kv.Handle, r []types.Datum) error

    // Key encoding
    RecordKey(h kv.Handle) kv.Key
    RecordPrefix() kv.Key
    IndexPrefix(indexID int64) kv.Key

    // Allocators
    Allocators(ctx sessionctx.Context) autoid.Allocators

    // Partitioning
    GetPartition(physicalID int64) table.PhysicalTable
    GetPartitionByRow(ctx sessionctx.Context, r []types.Datum) (table.PhysicalTable, error)
}

// Index interface for index metadata and operations
type Index interface {
    // Metadata
    Meta() *model.IndexInfo
    Name() model.CIStr
    ID() int64

    // Index properties
    Unique() bool
    Primary() bool
    State() model.SchemaState

    // Key operations
    Create(ctx sessionctx.Context, txn kv.Transaction, indexedValues []types.Datum, h kv.Handle) (kv.Handle, error)
    Delete(ctx sessionctx.Context, txn kv.Transaction, indexedValues []types.Datum, h kv.Handle) error
    Drop(ctx sessionctx.Context, txn kv.Transaction) error

    // Iteration
    SeekFirst(r kv.Retriever) (iter table.IndexIterator, err error)
    Seek(ctx sessionctx.Context, r kv.Retriever, indexedValues []types.Datum) (iter table.IndexIterator, hit bool, err error)
}
```

#### Implementation Examples

```go
// Table implementation
type tableCommon struct {
    tableID int64
    meta    *model.TableInfo
    cols    []*table.Column
    indices []table.Index

    // Allocators for auto-increment columns
    allocs autoid.Allocators

    // Partition information
    partitioned bool
    partitions  map[int64]table.PhysicalTable
}

func (t *tableCommon) AddRecord(ctx sessionctx.Context, r []types.Datum, opts ...table.AddRecordOption) (kv.Handle, error) {
    // Generate record handle
    recordID, err := t.allocs.Get(autoid.RowIDAllocType).Alloc(ctx, 1, 1, 1)
    if err != nil {
        return nil, err
    }

    handle := kv.IntHandle(recordID)

    // Build record key and value
    key := t.RecordKey(handle)
    value, err := tablecodec.EncodeRow(ctx.GetSessionVars().StmtCtx, r, t.meta.Columns, nil, nil)
    if err != nil {
        return nil, err
    }

    // Insert record
    if err := ctx.Txn(true).Set(key, value); err != nil {
        return nil, err
    }

    // Update indices
    for _, index := range t.WritableIndices() {
        indexVals, err := t.buildIndexValues(r, index)
        if err != nil {
            return nil, err
        }

        _, err = index.Create(ctx, ctx.Txn(true), indexVals, handle)
        if err != nil {
            return nil, err
        }
    }

    return handle, nil
}

func (t *tableCommon) buildIndexValues(record []types.Datum, index table.Index) ([]types.Datum, error) {
    indexColumns := index.Meta().Columns
    indexValues := make([]types.Datum, len(indexColumns))

    for i, indexCol := range indexColumns {
        columnIndex := indexCol.Offset
        if columnIndex >= len(record) {
            return nil, errors.Errorf("column index %d out of range", columnIndex)
        }
        indexValues[i] = record[columnIndex]
    }

    return indexValues, nil
}

// Index implementation
type index struct {
    idxInfo *model.IndexInfo
    tblInfo *model.TableInfo
    prefix  kv.Key
}

func (idx *index) Create(ctx sessionctx.Context, txn kv.Transaction, indexedValues []types.Datum, h kv.Handle) (kv.Handle, error) {
    // Encode index key
    key, distinct, err := tablecodec.EncodeIndexKey(ctx.GetSessionVars().StmtCtx, idx.tblInfo, idx.idxInfo, indexedValues, h)
    if err != nil {
        return nil, err
    }

    // Check for duplicate keys in unique index
    if idx.Unique() {
        existing, err := txn.Get(context.Background(), key)
        if err == nil && len(existing) > 0 {
            return nil, kv.ErrKeyExists.FastGenByArgs(key)
        }
        if !kv.IsErrNotFound(err) {
            return nil, err
        }
    }

    // Encode index value
    value, err := tablecodec.EncodeIndexValue(ctx.GetSessionVars().StmtCtx, indexedValues, h)
    if err != nil {
        return nil, err
    }

    // Insert index entry
    err = txn.Set(key, value)
    if err != nil {
        return nil, err
    }

    return h, nil
}
```

This comprehensive API documentation provides developers with the knowledge needed to understand, extend, and debug TiDB's internal interfaces. Each interface is designed with specific responsibilities and clear contracts, enabling modular development and testing.