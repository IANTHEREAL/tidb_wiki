# TiDB Code Walkthrough

This guide provides a practical walkthrough through TiDB's codebase, helping developers understand how different components work together through real code examples and execution traces.

## 🗺️ Repository Structure Overview

```
tidb/
├── pkg/                          # Core TiDB packages
│   ├── parser/                   # SQL parser and AST
│   ├── planner/                  # Query optimizer and planner
│   ├── executor/                 # Query execution engine
│   ├── session/                  # Session management
│   ├── server/                   # MySQL protocol server
│   ├── store/                    # Storage layer integration
│   ├── kv/                       # Key-value abstractions
│   ├── tablecodec/               # Table data encoding
│   ├── meta/                     # Metadata management
│   ├── ddl/                      # DDL operations
│   ├── domain/                   # Schema and cluster info
│   └── ...                       # Many other packages
├── cmd/                          # Command-line tools
│   └── tidb-server/              # Main TiDB server
├── docs/                         # Design documents
├── tests/                        # Integration tests
├── build/                        # Build scripts
└── tools/                        # Development tools
```

## 🚀 Query Execution Journey

Let's trace a simple SQL query through the entire TiDB stack:

**Query**: `SELECT name FROM users WHERE id = 1`

### Step 1: Server Layer - Connection Handling

**File**: `pkg/server/conn.go`

```go
// Entry point: MySQL protocol handling
func (cc *clientConn) Run(ctx context.Context) {
    // Main connection loop
    for {
        data, err := cc.readPacket()
        if err != nil {
            return err
        }
        
        // Dispatch command based on type
        if err = cc.dispatch(ctx, data); err != nil {
            cc.writeError(ctx, err)
        }
    }
}

// Query command handling - line 1234
func (cc *clientConn) handleQuery(ctx context.Context, sql string) error {
    // Create execution context
    ctx = context.WithValue(ctx, execdetails.StmtExecDetailKey, &execdetails.StmtExecDetails{})
    
    // Execute SQL statement
    rs, err := cc.ctx.Execute(ctx, sql)
    if err != nil {
        return err
    }
    
    // Send results back to client
    return cc.writeResultset(ctx, rs, false, false, false)
}
```

**Key Learning**: The server layer is a thin protocol adapter that delegates actual SQL processing to the session layer.

### Step 2: Session Layer - SQL Execution Coordination

**File**: `pkg/session/session.go`

```go
// Main SQL execution entry point - line 1567
func (s *session) Execute(ctx context.Context, sql string) ([]sqlexec.RecordSet, error) {
    // Parse SQL to AST
    stmtNodes, warns, err := s.ParseSQL(ctx, sql)
    if err != nil {
        return nil, err
    }
    
    var rs []sqlexec.RecordSet
    for _, stmtNode := range stmtNodes {
        // Execute each statement
        r, err := s.ExecuteStmt(ctx, stmtNode)
        if err != nil {
            return nil, err
        }
        if r != nil {
            rs = append(rs, r)
        }
    }
    return rs, nil
}

// Execute single statement - line 1634
func (s *session) ExecuteStmt(ctx context.Context, stmtNode ast.StmtNode) (sqlexec.RecordSet, error) {
    // Prepare transaction context
    if err := s.PrepareTxnCtx(ctx); err != nil {
        return nil, err
    }
    
    // Compile statement (parse → plan → optimize)
    stmt, err := s.compiler.Compile(ctx, stmtNode)
    if err != nil {
        return nil, err
    }
    
    // Execute the compiled statement
    return s.runStmt(ctx, stmt)
}
```

**Key Learning**: The session manages the complete SQL lifecycle: parsing, planning, optimization, and execution.

### Step 3: SQL Parser - Text to AST

**File**: `pkg/parser/parser.go`

```go
// Parse SQL text to AST nodes - line 234
func (parser *Parser) Parse(sql, charset, collation string) ([]ast.StmtNode, []error, error) {
    // Initialize lexer with SQL text
    parser.lexer.reset(sql)
    parser.lexer.SetSQLMode(parser.lexer.sqlMode)
    
    // Run yacc-generated parser
    parser.result = parser.result[:0]
    parser.src = sql
    parser.charset = charset
    parser.collation = collation
    
    // Parse using state machine
    ret := parser.yyParse(parser.lexer)
    
    if ret != 0 && ret != 1 {
        return nil, nil, parser.scanner.lastError
    }
    
    return parser.result, parser.warnings, nil
}
```

**Example AST for our query**:
```go
// Simplified AST structure
&ast.SelectStmt{
    Fields: &ast.FieldList{
        Fields: []*ast.SelectField{
            {Expr: &ast.ColumnNameExpr{Name: &ast.ColumnName{Name: "name"}}},
        },
    },
    From: &ast.TableRefsClause{
        TableRefs: &ast.TableSource{Source: &ast.TableName{Name: "users"}},
    },
    Where: &ast.BinaryOperationExpr{
        Op: opcode.EQ,
        L:  &ast.ColumnNameExpr{Name: &ast.ColumnName{Name: "id"}},
        R:  &ast.ValueExpr{Value: types.NewIntDatum(1)},
    },
}
```

### Step 4: Query Planner - AST to Logical Plan

**File**: `pkg/planner/core/planbuilder.go`

```go
// Build logical plan from SELECT AST - line 892
func (b *PlanBuilder) buildSelect(ctx context.Context, sel *ast.SelectStmt) (LogicalPlan, error) {
    // 1. Build FROM clause (data source)
    p, err := b.buildResultSetNode(ctx, sel.From)
    if err != nil {
        return nil, err
    }
    
    // 2. Handle WHERE clause (selection)
    if sel.Where != nil {
        p, err = b.buildSelection(ctx, p, sel.Where, nil)
        if err != nil {
            return nil, err
        }
    }
    
    // 3. Handle SELECT fields (projection)
    p, err = b.buildProjection(ctx, p, sel.Fields.Fields, nil)
    if err != nil {
        return nil, err
    }
    
    return p, nil
}

// Build table scan - line 3456
func (b *PlanBuilder) buildDataSource(ctx context.Context, tn *ast.TableName) (LogicalPlan, error) {
    // Get table metadata
    tableInfo, err := b.is.TableByName(tn.Schema, tn.Name)
    if err != nil {
        return nil, err
    }
    
    // Create data source logical plan
    ds := DataSource{
        basePlan:        newBasePlan(ctx, plancodec.TypeTableScan, 0),
        TableAsName:     tn.Name,
        table:           tableInfo,
        statisticTable:  b.getStatsTable(tableInfo.Meta().ID),
    }.Init(b.ctx, b.getSelectOffset())
    
    return ds, nil
}
```

**Logical Plan Structure**:
```
LogicalProjection(name)
└── LogicalSelection(id = 1) 
    └── DataSource(users)
```

### Step 5: Query Optimizer - Logical to Physical Plan

**File**: `pkg/planner/core/optimizer.go`

```go
// Main optimization entry point - line 123
func DoOptimize(ctx context.Context, sctx sessionctx.Context, flag uint64, logic LogicalPlan) (PhysicalPlan, float64, error) {
    // Phase 1: Logical optimization (rule-based)
    logic, err := logicalOptimize(ctx, flag, logic)
    if err != nil {
        return nil, 0, err
    }
    
    // Phase 2: Physical optimization (cost-based)
    physical, cost, err := physicalOptimize(logic, &property.PhysicalProperty{})
    if err != nil {
        return nil, 0, err
    }
    
    return physical, cost, nil
}
```

**File**: `pkg/planner/core/find_best_task.go`

```go
// Find best physical plan - line 567
func (ds *DataSource) findBestTask(prop *property.PhysicalProperty) (t task, cntPlan int64, err error) {
    // Generate possible access paths
    paths := ds.generateAccessPaths()
    
    var bestTask task
    var bestCost float64 = math.MaxFloat64
    
    // Evaluate each access path
    for _, path := range paths {
        // Consider table scan
        if path.IsTablePath {
            task, err := ds.convertToTableScan(prop, path)
            if err != nil {
                continue
            }
            
            cost := ds.calculateCost(task)
            if cost < bestCost {
                bestTask = task
                bestCost = cost
            }
        }
        
        // Consider index scans
        if !path.IsTablePath {
            task, err := ds.convertToIndexScan(prop, path)
            if err != nil {
                continue
            }
            
            cost := ds.calculateCost(task)
            if cost < bestCost {
                bestTask = task
                bestCost = cost
            }
        }
    }
    
    return bestTask, 1, nil
}
```

**Physical Plan Structure**:
```
PhysicalProjection(name)
└── PhysicalSelection(id = 1)
    └── PhysicalTableScan(users, range: [1,1])
```

### Step 6: Query Executor - Physical Plan Execution

**File**: `pkg/executor/adapter.go`

```go
// Execute physical plan - line 456
func (a *ExecStmt) Exec(ctx context.Context) (_ sqlexec.RecordSet, err error) {
    // Build executor tree from physical plan
    e, err := a.buildExecutor()
    if err != nil {
        return nil, err
    }
    
    // Initialize executor
    if err = e.Open(ctx); err != nil {
        return nil, err
    }
    
    // Create record set for result streaming
    return &recordSet{
        executor:   e,
        stmt:       a,
        ctx:        ctx,
    }, nil
}
```

**File**: `pkg/executor/builder.go`

```go
// Build executor tree - line 234
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
    // ... many other executor types
    }
}

// Build table reader executor - line 1234
func (b *ExecutorBuilder) buildTableReader(v *plannercore.PhysicalTableScan) Executor {
    // Create table reader executor
    ret := &TableReaderExecutor{
        baseExecutor:      newBaseExecutor(b.ctx, v.Schema(), v.ID()),
        table:            v.Table,
        physicalTableID:  v.PhysicalTableID,
        keepOrder:        v.KeepOrder,
        desc:            v.Desc,
        ranges:          v.Ranges,
        dagPB:           b.constructDAGReq([]plannercore.PhysicalPlan{v}),
    }
    
    return ret
}
```

### Step 7: Storage Access - Distributed Execution

**File**: `pkg/executor/table_reader.go`

```go
// Execute table scan - line 123
func (e *TableReaderExecutor) Open(ctx context.Context) error {
    // Build distributed SQL request
    kvReq, err := e.buildKVReq(ctx)
    if err != nil {
        return err
    }
    
    // Send request to storage layer
    e.resultHandler = &tableResultHandler{}
    result, err := e.SelectResult(ctx, e.ctx, kvReq, retTypes(e))
    if err != nil {
        return err
    }
    
    e.resultHandler.open(result)
    return nil
}

// Build key-value request - line 234
func (e *TableReaderExecutor) buildKVReq(ctx context.Context) (*kv.Request, error) {
    // Convert table ranges to key ranges
    keyRanges, err := e.buildKeyRanges()
    if err != nil {
        return nil, err
    }
    
    // Create distributed SQL request
    return &kv.Request{
        Tp:            kv.ReqTypeDAG,
        StartKey:      keyRanges[0].StartKey,
        EndKey:        keyRanges[len(keyRanges)-1].EndKey,
        KeyRanges:     keyRanges,
        Data:          e.dagPB,
        Concurrency:   e.concurrency,
        StoreType:     kv.TiKV,
    }, nil
}
```

**File**: `pkg/store/driver/txn.go`

```go
// Send request to TiKV - line 345
func (txn *tikvTxn) Get(ctx context.Context, k kv.Key) ([]byte, error) {
    // Get snapshot for consistent read
    snapshot := txn.us.GetSnapshot()
    
    // Read from TiKV
    val, err := snapshot.Get(ctx, k)
    if err != nil {
        return nil, err
    }
    
    return val, nil
}
```

## 🔍 Deep Dive Examples

### Example 1: Adding a New Built-in Function

Let's walk through adding a new string function `REVERSE()`:

#### Step 1: Define Function Signature

**File**: `pkg/expression/builtin_string.go`

```go
// Add to function registry - line 89
var funcs = map[string]functionClass{
    // ... existing functions
    ast.Reverse: &reverseFunctionClass{baseFunctionClass{ast.Reverse, 1, 1}},
}

// Function class definition - line 2345
type reverseFunctionClass struct {
    baseFunctionClass
}

func (c *reverseFunctionClass) getFunction(ctx sessionctx.Context, args []Expression) (builtinFunc, error) {
    if err := c.verifyArgs(args); err != nil {
        return nil, err
    }
    
    bf, err := newBaseBuiltinFuncWithTp(ctx, c.funcName, args, types.ETString, types.ETString)
    if err != nil {
        return nil, err
    }
    
    sig := &builtinReverseSig{bf}
    sig.setPbCode(tipb.ScalarFuncSig_Reverse)
    return sig, nil
}
```

#### Step 2: Implement Function Logic

```go
// Function implementation - line 2389
type builtinReverseSig struct {
    baseBuiltinFunc
}

func (b *builtinReverseSig) Clone() builtinFunc {
    newSig := &builtinReverseSig{}
    newSig.cloneFrom(&b.baseBuiltinFunc)
    return newSig
}

func (b *builtinReverseSig) evalString(row chunk.Row) (string, bool, error) {
    str, isNull, err := b.args[0].EvalString(b.ctx, row)
    if isNull || err != nil {
        return "", isNull, err
    }
    
    // Reverse the string
    runes := []rune(str)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    
    return string(runes), false, nil
}
```

#### Step 3: Update Parser (if needed)

**File**: `pkg/parser/parser.y`

```yacc
// Add to function name tokens - line 1234
%token REVERSE "REVERSE"

// Add to function call rules - line 5678
simple_func:
    REVERSE '(' expression ')'
    {
        $$ = &ast.FuncCallExpr{FnName: model.NewCIStr("REVERSE"), Args: []ast.ExprNode{$3}}
    }
```

#### Step 4: Add Tests

**File**: `pkg/expression/builtin_string_test.go`

```go
func TestReverse(t *testing.T) {
    ctx := createContext(t)
    tests := []struct {
        input    interface{}
        expected interface{}
    }{
        {"hello", "olleh"},
        {"", ""},
        {"a", "a"},
        {"世界", "界世"},
        {nil, nil},
    }
    
    for _, test := range tests {
        f, err := newFunctionForTest(ctx, ast.Reverse, primitiveValsToConstants(ctx, []interface{}{test.input})...)
        require.NoError(t, err)
        
        result, err := evalBuiltinFunc(f, chunk.Row{})
        require.NoError(t, err)
        
        if test.expected == nil {
            require.True(t, result.IsNull())
        } else {
            require.Equal(t, test.expected, result.GetString())
        }
    }
}
```

### Example 2: Tracing a DDL Operation

Let's trace a `CREATE TABLE` statement:

#### DDL Entry Point

**File**: `pkg/ddl/ddl.go`

```go
// DDL execution entry point - line 456
func (d *ddl) DoDDLJob(ctx sessionctx.Context, job *model.Job) error {
    // Validate DDL job
    if err := d.validateDDLJob(job); err != nil {
        return err
    }
    
    // Add job to DDL queue
    err := d.addDDLJob(ctx, job)
    if err != nil {
        return err
    }
    
    // Wait for job completion
    return d.waitDDLJobDone(ctx, job)
}
```

#### Job Processing

**File**: `pkg/ddl/ddl_worker.go`

```go
// DDL worker main loop - line 234
func (w *worker) start(d *ddlCtx) {
    defer w.wg.Done()
    
    for {
        select {
        case <-w.ddlJobCh:
            // Process DDL job
            err := w.handleDDLJobQueue(d)
            if err != nil {
                logutil.Logger(w.logCtx).Error("handle DDL job failed", zap.Error(err))
            }
        case <-w.ctx.Done():
            return
        }
    }
}

// Handle specific DDL job - line 567
func (w *worker) runDDLJob(d *ddlCtx, t *meta.Meta, job *model.Job) (ver int64, runJobErr error) {
    // Dispatch based on job type
    switch job.Type {
    case model.ActionCreateTable:
        ver, runJobErr = w.onCreateTable(d, t, job)
    case model.ActionDropTable:
        ver, runJobErr = w.onDropTable(d, t, job)
    case model.ActionAddColumn:
        ver, runJobErr = w.onAddColumn(d, t, job)
    // ... other DDL types
    }
    
    return
}
```

## 🔧 Debugging Techniques

### 1. Adding Debug Logs

```go
// Add debug logging in any package
import "github.com/pingcap/log"
import "go.uber.org/zap"

func someFunction() {
    // Debug level logging
    log.Debug("debugging info", zap.String("key", "value"))
    
    // Info level logging
    log.Info("important event", zap.Int64("id", 123))
    
    // Error logging
    log.Error("error occurred", zap.Error(err))
}
```

### 2. Using Delve Debugger

```bash
# Debug specific test
dlv test ./pkg/executor -- -test.run TestTableReaderExecutor

# Set breakpoints
(dlv) break pkg/executor.(*TableReaderExecutor).Open
(dlv) break pkg/planner/core.(*DataSource).findBestTask

# Continue execution
(dlv) continue

# Inspect variables
(dlv) print e.ranges
(dlv) print ds.statisticTable.Count
```

### 3. Profiling Query Execution

```go
// Add profiling to session
import _ "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    
    // Run TiDB server
    // ...
}
```

```bash
# Profile CPU usage during query
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Profile memory allocation
go tool pprof http://localhost:6060/debug/pprof/heap

# Profile goroutines
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

## 🎯 Common Code Patterns

### 1. Error Handling Pattern

```go
// Standard error handling in TiDB
func someFunction() error {
    result, err := riskyOperation()
    if err != nil {
        // Wrap error with context
        return errors.Trace(err)
    }
    
    if result == nil {
        // Return specific error
        return errors.New("unexpected nil result")
    }
    
    return nil
}
```

### 2. Interface Usage Pattern

```go
// Common interface pattern - define behavior, not data
type Executor interface {
    Open(ctx context.Context) error
    Next(ctx context.Context, req *chunk.Chunk) error
    Close() error
    Schema() *expression.Schema
}

// Composition over inheritance
type baseExecutor struct {
    ctx      sessionctx.Context
    schema   *expression.Schema
    id       int
    children []Executor
}

type TableReaderExecutor struct {
    baseExecutor  // Embedded for common functionality
    
    // Specific fields
    table    table.Table
    ranges   []*ranger.Range
}
```

### 3. Context Propagation Pattern

```go
// Context flows through entire call stack
func (s *session) ExecuteStmt(ctx context.Context, stmt ast.StmtNode) (sqlexec.RecordSet, error) {
    // Add session info to context
    ctx = context.WithValue(ctx, sessionctx.ConnIDKey, s.sessionVars.ConnectionID)
    
    // Pass context down
    return s.executor.ExecuteStmt(ctx, stmt)
}

func (e *Executor) ExecuteStmt(ctx context.Context, stmt ast.StmtNode) (sqlexec.RecordSet, error) {
    // Extract session info from context
    connID := ctx.Value(sessionctx.ConnIDKey).(uint64)
    
    // Continue propagation
    return e.plan.Execute(ctx)
}
```

## 📊 Performance Analysis Walkthrough

### 1. Identifying Bottlenecks

```go
// Add timing measurements
import "time"

func (e *TableReaderExecutor) Open(ctx context.Context) error {
    start := time.Now()
    defer func() {
        duration := time.Since(start)
        if duration > time.Millisecond*100 { // Log slow operations
            log.Warn("slow table reader open", 
                zap.Duration("duration", duration),
                zap.String("table", e.table.Meta().Name.O))
        }
    }()
    
    // Original logic
    return e.buildKVReq(ctx)
}
```

### 2. Memory Usage Analysis

```go
// Track memory allocations
import "runtime"

func analyzeMemoryUsage() {
    var m1, m2 runtime.MemStats
    
    runtime.GC()
    runtime.ReadMemStats(&m1)
    
    // Run operation
    result := expensiveOperation()
    
    runtime.GC()
    runtime.ReadMemStats(&m2)
    
    log.Info("memory usage",
        zap.Uint64("alloc_delta", m2.Alloc - m1.Alloc),
        zap.Uint64("sys_delta", m2.Sys - m1.Sys))
}
```

## 🚀 Next Steps

After completing this walkthrough:

1. **Pick a Module**: Choose a specific module (executor, planner, etc.) to study in depth
2. **Read Tests**: Tests often show the best examples of how code is intended to be used
3. **Make Small Changes**: Start with simple bug fixes or test additions
4. **Use Debugging Tools**: Practice with delve, profiler, and logging
5. **Join Community**: Ask questions in Discord/Slack when stuck

## 📚 Related Documentation

- [Architecture Overview](../architecture/overview.md)
- [Development Setup](setup.md)
- [Module Documentation](../modules/)
- [Contributing Guidelines](../guides/contributing.md)