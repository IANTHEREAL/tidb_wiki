# TiDB SQL Layer Deep Dive

The SQL layer is the brain of TiDB, responsible for processing SQL queries from parsing to execution. This layer implements MySQL-compatible SQL processing while providing distributed database capabilities.

## 🏗️ Architecture Overview

```
Client SQL Request
        ↓
┌─────────────────┐
│  Server Layer   │ ← Protocol handling, connection management
│  pkg/server/    │
└─────────┬───────┘
          ↓
┌─────────────────┐
│ Session Manager │ ← Context, transactions, state management
│  pkg/session/   │
└─────────┬───────┘
          ↓
┌─────────────────┐
│   SQL Parser    │ ← SQL → AST conversion
│  pkg/parser/    │
└─────────┬───────┘
          ↓
┌─────────────────┐
│ Query Planner   │ ← Optimization, cost-based planning
│  pkg/planner/   │
└─────────┬───────┘
          ↓
┌─────────────────┐
│ Query Executor  │ ← Plan execution, storage access
│  pkg/executor/  │
└─────────────────┘
```

## 🔧 Component Deep Dive

### 1. Server Layer (`pkg/server/`)

The entry point for all client connections, implementing the MySQL wire protocol.

#### Key Files and Components

**Main Server Implementation**
- **`server.go`**: Server lifecycle and connection management
- **`conn.go`**: Individual connection handling and MySQL protocol
- **`conn_stmt.go`**: Prepared statement protocol

```go
// Core server structure
type Server struct {
    cfg             *config.Config
    tlsConfig       *tls.Config
    driver          IDriver
    listener        net.Listener
    capability      uint32
    status          int32
}

// Connection handling
type clientConn struct {
    server          *Server
    bufReadConn     *bufferedReadConn
    tlsConn         *tls.Conn
    sessionVars     *variable.SessionVars
    ctx             sessionctx.Context
}
```

#### Protocol Implementation

**Connection Lifecycle**:
1. `(s *Server) onConn(conn *clientConn)` - New connection setup
2. `(cc *clientConn) handshake()` - MySQL handshake protocol
3. `(cc *clientConn) Run()` - Main command processing loop
4. `(cc *clientConn) dispatch()` - Command routing

**Command Processing**:
```go
func (cc *clientConn) dispatch(ctx context.Context, data []byte) error {
    cmd := data[0]
    switch cmd {
    case mysql.ComQuery:
        return cc.handleQuery(ctx, hack.String(data[1:]))
    case mysql.ComStmtPrepare:
        return cc.handleStmtPrepare(ctx, hack.String(data[1:]))
    case mysql.ComStmtExecute:
        return cc.handleStmtExecute(ctx, data[1:])
    // ... other commands
    }
}
```

#### Integration Points
- **Session Management**: Creates and manages session contexts
- **Query Processing**: Orchestrates SQL execution pipeline
- **Result Formatting**: Converts execution results to MySQL protocol format

### 2. Session Management (`pkg/session/`)

Manages connection state, transactions, and execution context.

#### Key Files and Components

**Core Session Implementation**
- **`session.go`**: Main session logic and SQL execution coordination
- **`bootstrap.go`**: System initialization and schema setup
- **`txn.go`**: Transaction lifecycle management

```go
// Session context interface
type sessionctx.Context interface {
    GetSessionVars() *variable.SessionVars
    SetSessionVars(vars *variable.SessionVars)
    GetStore() kv.Storage
    Txn(bool) (kv.Transaction, error)
    // ... many more methods
}

// Session implementation
type session struct {
    values      map[string]interface{}
    store       kv.Storage
    parser      *parser.Parser
    sessionVars *variable.SessionVars
    mu          struct {
        sync.RWMutex
        buf         *chunk.Chunk
    }
}
```

#### Session Lifecycle

**Initialization Flow**:
```go
func (s *session) ExecuteStmt(ctx context.Context, stmtNode ast.StmtNode) (sqlexec.RecordSet, error) {
    // 1. Transaction management
    if err := s.PrepareTxnCtx(ctx); err != nil {
        return nil, err
    }
    
    // 2. Statement compilation
    compiledStmt, err := s.compiler.Compile(ctx, stmtNode)
    if err != nil {
        return nil, err
    }
    
    // 3. Execution
    return s.runStmt(ctx, compiledStmt)
}
```

#### Transaction Management
- **Optimistic Transactions**: Default transaction model
- **Two-Phase Commit**: Ensures ACID properties in distributed environment
- **Snapshot Isolation**: MVCC-based isolation level

### 3. SQL Parser (`pkg/parser/`)

Converts SQL text into Abstract Syntax Tree (AST) for further processing.

#### Key Files and Components

**Parser Core**
- **`parser.go`**: Main parser generated from yacc grammar (7.6MB)
- **`parser.y`**: Yacc grammar file defining MySQL syntax
- **`lexer.go`**: Lexical analyzer for tokenization

**AST Definitions**
- **`ast/ast.go`**: Core AST node interfaces
- **`ast/ddl.go`**: DDL statement nodes
- **`ast/dml.go`**: DML statement nodes
- **`ast/expressions.go`**: Expression and operator nodes

```go
// Core AST node interface
type Node interface {
    Accept(v Visitor) (node Node, ok bool)
    Restore(ctx *RestoreCtx) error
    Text() string
    SetText(text string)
}

// Statement node interface
type StmtNode interface {
    Node
    statement()
}

// Expression node interface
type ExprNode interface {
    Node
    SetType(tp *types.FieldType)
    GetType() *types.FieldType
    SetFlag(flag uint64)
    GetFlag() uint64
}
```

#### Parsing Process

**SQL to AST Flow**:
```go
func (parser *Parser) Parse(sql, charset, collation string) ([]ast.StmtNode, []error, error) {
    // 1. Lexical analysis
    parser.lexer.reset(sql)
    
    // 2. Syntax analysis using yacc-generated state machine
    parser.yyParse(parser.lexer)
    
    // 3. AST construction
    return parser.result, parser.warnings, parser.lastErrorAsWarn
}
```

#### Key Patterns
- **Visitor Pattern**: For AST traversal and transformation
- **Bottom-up Parsing**: Using yacc/bison approach
- **Error Recovery**: Graceful handling of syntax errors

### 4. Query Planner (`pkg/planner/`)

Transforms AST into optimized execution plans using advanced optimization techniques.

#### Key Files and Components

**Plan Building**
- **`optimize.go`**: Main optimization entry point
- **`core/planbuilder.go`**: Converts AST to logical plans
- **`core/logical_plan_builder.go`**: Logical operator construction

**Optimization**
- **`core/optimizer.go`**: Optimization framework coordination
- **`core/rule/`**: Rule-based optimization transformations
- **`cascades/`**: Advanced Cascades optimization framework

```go
// Plan interface hierarchy
type base.Plan interface {
    ID() int
    TP() string
    Schema() *expression.Schema
    Stats() *property.StatsInfo
    ExplainInfo() string
}

type base.LogicalPlan interface {
    base.Plan
    HashCode() []byte
    PredicatePushDown([]expression.Expression) ([]expression.Expression, base.LogicalPlan)
    PruneColumns([]*expression.Column) error
    FindBestTask(prop *property.PhysicalProperty) (task base.Task, cntPlan int64, err error)
}

type base.PhysicalPlan interface {
    base.Plan
    Attach2Task(...base.Task) base.Task
    ToPB(ctx sessionctx.Context, storeType kv.StoreType) (*tipb.Executor, error)
    GetChildReqProps(idx int) *property.PhysicalProperty
}
```

#### Optimization Process

**Two-Phase Optimization**:
```go
func DoOptimize(ctx context.Context, sctx sessionctx.Context, flag uint64, logic LogicalPlan) (PhysicalPlan, float64, error) {
    // Phase 1: Rule-based logical optimization
    logic, err := logicalOptimize(ctx, flag, logic)
    if err != nil {
        return nil, 0, err
    }
    
    // Phase 2: Cost-based physical optimization
    physical, cost, err := physicalOptimize(logic, &property.PhysicalProperty{})
    if err != nil {
        return nil, 0, err
    }
    
    return physical, cost, nil
}
```

**Key Optimization Rules**:
- **Predicate Pushdown**: Push filters close to data sources
- **Column Pruning**: Eliminate unnecessary column reads
- **Join Reordering**: Optimize join order based on cardinality
- **Index Selection**: Choose optimal indexes for data access

### 5. Query Executor (`pkg/executor/`)

Executes physical plans using the Volcano iterator model with optimizations.

#### Key Files and Components

**Execution Framework**
- **`adapter.go`**: Bridges plans and executors
- **`builder.go`**: Builds executor trees from plans
- **`internal/exec/executor.go`**: Core executor interface

**Operator Implementations**
- **`join/`**: Join algorithm implementations
- **`aggregate/`**: Aggregation operators
- **`sort.go`**: Sort operations
- **`table_reader.go`**: Table scan execution

```go
// Core executor interface
type Executor interface {
    Open(ctx context.Context) error
    Next(ctx context.Context, req *chunk.Chunk) error
    Close() error
    Schema() *expression.Schema
}

// Execution runtime
type baseExecutor struct {
    ctx         sessionctx.Context
    id          int
    schema      *expression.Schema
    initCap     int
    maxChunkSize int
    children    []Executor
}
```

#### Execution Model

**Volcano Iterator Pattern**:
```go
func (e *SelectionExec) Next(ctx context.Context, req *chunk.Chunk) error {
    // 1. Get input from child executor
    err := Next(ctx, e.children[0], e.inputChunk)
    if err != nil {
        return err
    }
    
    // 2. Apply selection condition
    selected, err := expression.VectorizedFilter(e.ctx, e.filters, e.inputChunk, req)
    if err != nil {
        return err
    }
    
    // 3. Return filtered results
    req.SetSel(selected)
    return nil
}
```

#### Distributed Execution

**TiKV Integration**:
- **Coprocessor Requests**: Push computation to storage layer
- **Region-based Processing**: Parallel execution across data regions
- **Result Aggregation**: Combine partial results from multiple regions

**TiFlash MPP**:
- **MPP Task Distribution**: Coordinate massively parallel processing
- **Columnar Processing**: Vectorized execution on columnar data
- **Pipeline Execution**: Streaming data processing

## 🔄 Component Interactions

### SQL Execution Flow

```
1. Server receives SQL text
   ↓
2. Session provides execution context
   ↓
3. Parser generates AST
   ↓
4. Planner creates optimized physical plan
   ↓
5. Executor executes plan with storage access
   ↓
6. Results flow back through the pipeline
```

### Key Integration Patterns

#### Context Propagation
```go
// Context flows through all components
func (s *session) ExecuteStmt(ctx context.Context, stmt ast.StmtNode) (sqlexec.RecordSet, error) {
    // Parser uses session context for variables
    // Planner uses context for optimization settings
    // Executor uses context for resource management
}
```

#### Interface-Based Design
- Clean separation between components
- Easy testing and mocking
- Plugin architecture support

#### Resource Management
- Memory tracking across execution
- Connection pooling
- Transaction lifecycle management

## 🚀 Performance Optimizations

### Vectorized Processing
- **Chunk-based execution**: Batch processing for efficiency
- **Column-oriented operations**: SIMD-friendly data layout
- **Expression evaluation**: Vectorized expression computation

### Memory Management
- **Memory pools**: Reduce allocation overhead
- **Spill-to-disk**: Handle large datasets
- **Resource tracking**: Prevent memory leaks

### Parallel Execution
- **Intra-query parallelism**: Parallel operators within single query
- **Pipeline parallelism**: Overlapped execution stages
- **MPP execution**: Massively parallel processing for analytics

## 🔧 Development Guidelines

### Adding New Operators

1. **Define logical operator** in `pkg/planner/core/`
2. **Implement optimization rules** for the operator
3. **Create physical implementation** with cost estimation
4. **Build executor** in `pkg/executor/`
5. **Add tests** for all components

### Parser Extensions

1. **Update grammar** in `parser.y`
2. **Define AST nodes** in `ast/` package
3. **Regenerate parser** using `make parser`
4. **Add semantic analysis** in planner

### Integration Testing

- **SQL test framework**: `pkg/testkit/`
- **Plan verification**: Check optimization results
- **Performance testing**: Benchmark critical paths

## 📚 Related Documentation

- [Query Optimizer Deep Dive](optimizer.md)
- [Storage Engine Integration](storage-engine.md)
- [Performance Guidelines](../patterns/performance.md)
- [Development Setup](../development/setup.md)