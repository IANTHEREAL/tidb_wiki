# TiDB Development Guide

## Overview

This guide provides comprehensive information for developers working on TiDB, covering everything from setting up a development environment to understanding the contribution workflow, debugging techniques, and best practices.

## Table of Contents

1. [Development Environment Setup](#development-environment-setup)
2. [Code Organization and Standards](#code-organization-and-standards)
3. [Building and Testing](#building-and-testing)
4. [Debugging and Profiling](#debugging-and-profiling)
5. [Contributing Guidelines](#contributing-guidelines)
6. [Development Workflows](#development-workflows)
7. [Code Generation](#code-generation)
8. [Performance Considerations](#performance-considerations)

## Development Environment Setup

### Prerequisites

#### System Requirements
```bash
# Minimum requirements
- OS: Linux, macOS, or Windows with WSL2
- RAM: 8GB minimum, 16GB recommended
- CPU: 4 cores minimum, 8 cores recommended
- Disk: 50GB free space for full development setup
- Go: 1.21+ (latest stable version recommended)
```

#### Required Tools
```bash
# Core development tools
go version          # Go 1.21+
git version         # Git 2.0+
make --version      # GNU Make 4.0+

# Optional but recommended
docker --version    # For integration testing
bazel version       # Alternative build system
golangci-lint --version  # Code linting
```

### Setting Up the Development Environment

#### 1. Clone and Setup Repository
```bash
# Clone the repository
git clone https://github.com/pingcap/tidb.git
cd tidb

# Set up Git hooks (recommended)
make dev-setup

# Install development dependencies
make deps

# Verify setup
make check-setup
```

#### 2. IDE Configuration

**VS Code Setup (.vscode/settings.json)**
```json
{
    "go.buildTags": "intest",
    "go.testTags": "intest",
    "go.lintTool": "golangci-lint",
    "go.lintOnSave": "workspace",
    "go.formatTool": "goimports",
    "go.useLanguageServer": true,
    "gopls": {
        "build.buildFlags": ["-tags=intest"],
        "ui.semanticTokens": true,
        "ui.codelenses": {
            "generate": true,
            "test": true
        }
    }
}
```

**GoLand/IntelliJ Setup**
```bash
# Build tags for better IDE support
# Go to Settings > Go > Build Tags & Vendoring
# Set build tags: intest

# Enable Go modules
# Go to Settings > Go > Go Modules
# Enable Go modules integration: checked
```

### Environment Variables

#### Development Configuration
```bash
# Essential environment variables
export GOPATH=$HOME/go
export GO111MODULE=on
export GOPROXY=https://proxy.golang.org,direct

# TiDB-specific settings
export TIDB_EDITION=Community
export TIDB_LOG_LEVEL=debug
export TIDB_CONFIG_PATH=./config/config.toml

# Testing configuration
export TIDB_TEST_STORE_TYPE=tikv    # or unistore for testing
export TIDB_PARALLELISM=8           # Parallel test execution
```

#### Development Makefile Targets
```bash
# Common development commands
make build           # Build TiDB binary
make dev            # Build with development flags
make server         # Build and start TiDB server
make test           # Run unit tests
make integration    # Run integration tests
make check          # Run all checks (lint, test, etc.)
make clean          # Clean build artifacts
```

## Code Organization and Standards

### Directory Structure Guidelines

#### Package Organization
```
pkg/
├── parser/          # SQL parsing and AST
│   ├── ast/         # AST node definitions
│   ├── mysql/       # MySQL compatibility
│   └── types/       # Type system
├── planner/         # Query optimization
│   ├── core/        # Core planning logic
│   ├── cascades/    # Cascades optimizer
│   └── property/    # Physical properties
├── executor/        # Query execution
│   ├── join/        # Join algorithms
│   ├── aggregate/   # Aggregation operators
│   └── sortexec/    # Sorting operations
├── session/         # Session management
├── server/          # Protocol handling
├── store/           # Storage abstraction
└── util/            # Utility packages
```

#### File Naming Conventions
```bash
# Go file naming
component.go           # Main implementation
component_test.go      # Unit tests
component_bench_test.go  # Benchmark tests
component_example_test.go  # Example tests

# Test file organization
integration_test.go    # Integration tests
real_tikv_test.go     # Tests requiring real TiKV
```

### Coding Standards

#### Go Style Guidelines
```go
// Package documentation - required for all packages
// Package executor implements the SQL execution engine for TiDB.
//
// The execution engine uses the Volcano iterator model with vectorized
// processing optimizations. It coordinates between local and distributed
// execution through the DistSQL framework.
package executor

// Struct definition with clear field grouping
type HashJoinExec struct {
    baseExecutor

    // Join configuration
    joinType     JoinType
    isOuterJoin  bool
    useOuterToBuild bool

    // Runtime state
    prepared     bool
    hashTable    *hashRowContainer
    joiners      []joiner

    // Resource management
    memTracker   *memory.Tracker
    diskTracker  *disk.Tracker

    // Concurrency control
    concurrency  int
    workerWg     sync.WaitGroup
}

// Method documentation with clear behavior description
// Open initializes the HashJoin executor and prepares for execution.
// It builds the hash table from the build side and sets up join workers.
//
// Returns error if memory allocation fails or build side is too large.
func (e *HashJoinExec) Open(ctx context.Context) error {
    if err := e.baseExecutor.Open(ctx); err != nil {
        return err
    }

    // Initialize hash table with memory tracking
    e.hashTable = newHashRowContainer(e.ctx, e.memTracker)

    // Build hash table from build side
    return e.fetchAndBuildHashTable(ctx)
}
```

#### Error Handling Patterns
```go
// Error wrapping with context
func (e *TableReaderExecutor) buildDAGRequest() (*tipb.DAGRequest, error) {
    dagReq, err := e.constructDAG()
    if err != nil {
        return nil, errors.Trace(err)  // Preserve stack trace
    }

    if err := e.validateDAG(dagReq); err != nil {
        return nil, errors.Annotate(err, "DAG validation failed")
    }

    return dagReq, nil
}

// Sentinel errors for specific conditions
var (
    ErrTableNotFound = errors.New("table not found")
    ErrInvalidPlan   = errors.New("invalid execution plan")
    ErrMemoryExceeded = errors.New("memory limit exceeded")
)

// Error classification for retry logic
func isRetryableError(err error) bool {
    switch errors.Cause(err).(type) {
    case *tikverr.RegionUnavailable:
        return true
    case *tikverr.ServerIsBusy:
        return true
    default:
        return false
    }
}
```

#### Interface Design Principles
```go
// Small, focused interfaces
type Executor interface {
    Open(ctx context.Context) error
    Next(ctx context.Context, req *chunk.Chunk) error
    Close() error
    Schema() *expression.Schema
}

// Composition over inheritance
type baseExecutor struct {
    ctx           sessionctx.Context
    id            int
    schema        *expression.Schema
    initCap       int
    maxChunkSize  int
    children      []Executor
}

// Embedding for behavior sharing
type TableReaderExecutor struct {
    baseExecutor    // Embedded base functionality

    // Specific fields for table reading
    table         *model.TableInfo
    ranges        []*ranger.Range
    dagPB         *tipb.DAGRequest
}
```

### Documentation Standards

#### Function Documentation
```go
// PerformJoin executes the join operation between two input streams.
//
// Parameters:
//   - ctx: Execution context for cancellation and timeouts
//   - leftInput: Left side input chunk stream
//   - rightInput: Right side input chunk stream
//   - joinType: Type of join (INNER, LEFT, RIGHT, FULL)
//
// Returns:
//   - Result chunks containing joined rows
//   - Error if join operation fails or is cancelled
//
// The function implements hash join algorithm with the following phases:
//   1. Build phase: Creates hash table from smaller input
//   2. Probe phase: Probes hash table with larger input
//   3. Result phase: Outputs matched rows according to join type
//
// Memory usage is tracked and spilling is triggered if memory limit exceeded.
// The operation supports cancellation through the provided context.
func PerformJoin(ctx context.Context, leftInput, rightInput Executor, joinType JoinType) (*JoinResult, error)
```

#### Package Documentation Examples
```go
// Package parser implements SQL parsing for TiDB.
//
// The parser converts MySQL-compatible SQL statements into Abstract Syntax Trees (ASTs)
// that can be processed by the planner and executor. It supports the full MySQL 8.0
// syntax including:
//
//   - DML: SELECT, INSERT, UPDATE, DELETE
//   - DDL: CREATE, ALTER, DROP for tables, indexes, views
//   - DCL: GRANT, REVOKE, user management
//   - Transactions: BEGIN, COMMIT, ROLLBACK
//   - Stored procedures and functions
//   - Window functions and CTEs
//
// The parser is implemented using a YACC-based grammar that generates efficient
// Go code. Error recovery and reporting provide detailed syntax error messages
// with suggestions for common mistakes.
//
// Example usage:
//
//   parser := parser.New()
//   stmt, err := parser.ParseOneStmt("SELECT * FROM users WHERE id = ?", "", "")
//   if err != nil {
//       log.Fatal(err)
//   }
//
//   // Process the AST
//   selectStmt := stmt.(*ast.SelectStmt)
//   fmt.Printf("Selected from: %s\n", selectStmt.From.TableRefs.Left.(*ast.TableSource).Source.(*ast.TableName).Name)
//
// Package parser is safe for concurrent use by multiple goroutines.
package parser
```

## Building and Testing

### Build System

#### Makefile Targets
```bash
# Development builds
make dev                    # Development build with debug info
make build                  # Production build
make race                   # Build with race detector
make failpoint-enable       # Enable failpoint testing

# Testing targets
make test                   # Unit tests
make gotest                 # Go test with coverage
make integration           # Integration tests
make bazel-test            # Bazel-based testing

# Code quality
make check                 # All checks (lint, vet, etc.)
make lint                  # Code linting
make vet                   # Go vet analysis
make fmt                   # Code formatting

# Documentation
make docs                  # Generate documentation
make dev-doc              # Development documentation
```

#### Build Configuration
```go
// Build tags for conditional compilation
// +build !race

package executor

// Race detector specific code
// +build race

package executor

// Integration test specific code
// +build intest

package executor
```

### Testing Strategy

#### Test Organization
```
tests/
├── integrationtest/     # Integration tests
├── realtikvtest/       # Tests requiring real TiKV
├── clustertest/        # Multi-node cluster tests
└── benchtest/          # Performance benchmarks

pkg/executor/
├── executor.go          # Implementation
├── executor_test.go     # Unit tests
├── benchmark_test.go    # Benchmarks
└── example_test.go      # Usage examples
```

#### Unit Testing Patterns
```go
// Table-driven tests
func TestHashJoinExecutor(t *testing.T) {
    tests := []struct {
        name        string
        leftRows    [][]interface{}
        rightRows   [][]interface{}
        joinType    JoinType
        conditions  []string
        expected    [][]interface{}
        expectError bool
    }{
        {
            name:      "inner join basic",
            leftRows:  [][]interface{}{{1, "a"}, {2, "b"}},
            rightRows: [][]interface{}{{1, "x"}, {2, "y"}},
            joinType:  InnerJoin,
            conditions: []string{"t1.id = t2.id"},
            expected:   [][]interface{}{{1, "a", 1, "x"}, {2, "b", 2, "y"}},
        },
        {
            name:      "left join with nulls",
            leftRows:  [][]interface{}{{1, "a"}, {3, "c"}},
            rightRows: [][]interface{}{{1, "x"}, {2, "y"}},
            joinType:  LeftOuterJoin,
            conditions: []string{"t1.id = t2.id"},
            expected:   [][]interface{}{{1, "a", 1, "x"}, {3, "c", nil, nil}},
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup test environment
            ctx := context.Background()
            sctx := mock.NewContext()

            // Create test executors
            leftExec := buildMockExecutor(tt.leftRows)
            rightExec := buildMockExecutor(tt.rightRows)

            // Build join executor
            joinExec := &HashJoinExec{
                baseExecutor: newBaseExecutor(sctx, nil, 0, leftExec, rightExec),
                joinType:     tt.joinType,
            }

            // Execute and verify results
            result, err := executeAndCollect(ctx, joinExec)
            if tt.expectError {
                require.Error(t, err)
                return
            }
            require.NoError(t, err)
            require.Equal(t, tt.expected, result)
        })
    }
}

// Benchmark tests
func BenchmarkHashJoin(b *testing.B) {
    sizes := []int{100, 1000, 10000, 100000}

    for _, leftSize := range sizes {
        for _, rightSize := range sizes {
            name := fmt.Sprintf("left_%d_right_%d", leftSize, rightSize)
            b.Run(name, func(b *testing.B) {
                // Setup benchmark data
                leftData := generateTestData(leftSize)
                rightData := generateTestData(rightSize)

                b.ResetTimer()
                for i := 0; i < b.N; i++ {
                    // Run join operation
                    result := performHashJoin(leftData, rightData)
                    _ = result // Prevent optimization
                }
            })
        }
    }
}
```

#### Integration Testing
```go
// Integration test using TestKit
func TestSelectWithJoin(t *testing.T) {
    store, clean := testkit.CreateMockStore(t)
    defer clean()

    tk := testkit.NewTestKit(t, store)
    tk.MustExec("use test")

    // Setup test data
    tk.MustExec("create table t1(id int, name varchar(20))")
    tk.MustExec("create table t2(id int, value varchar(20))")
    tk.MustExec("insert into t1 values (1, 'a'), (2, 'b'), (3, 'c')")
    tk.MustExec("insert into t2 values (1, 'x'), (2, 'y')")

    // Test join query
    result := tk.MustQuery("select t1.name, t2.value from t1 left join t2 on t1.id = t2.id order by t1.id")
    result.Check(testkit.Rows("a x", "b y", "c <nil>"))

    // Test performance characteristics
    tk.MustExec("analyze table t1, t2")
    rows := tk.MustQuery("explain analyze select * from t1 join t2 on t1.id = t2.id").Rows()

    // Verify join algorithm selection
    require.Contains(t, fmt.Sprintf("%v", rows[0]), "HashJoin")
}
```

### Mock and Test Utilities

#### Mock Framework Usage
```go
// Mock session context for testing
func createMockSessionCtx() sessionctx.Context {
    ctx := mock.NewContext()
    ctx.GetSessionVars().StmtCtx = &stmtctx.StatementContext{
        TimeZone:     time.UTC,
        SqlMode:      mysql.ModeStrictTransTables,
    }
    return ctx
}

// Mock storage for unit tests
func createMockStore() kv.Storage {
    store, err := mockstore.NewMockStore()
    if err != nil {
        panic(err)
    }
    return store
}

// Test data generators
func generateTestData(size int) [][]interface{} {
    data := make([][]interface{}, size)
    for i := 0; i < size; i++ {
        data[i] = []interface{}{
            i + 1,                           // ID
            fmt.Sprintf("value_%d", i+1),    // String value
            rand.Intn(100),                  // Random number
        }
    }
    return data
}
```

## Debugging and Profiling

### Debugging Techniques

#### Logging Configuration
```go
// Structured logging with context
import (
    "go.uber.org/zap"
    "github.com/pingcap/tidb/util/logutil"
)

func (e *HashJoinExec) debugJoinProgress(phase string, rowCount int) {
    logutil.BgLogger().Debug("join execution progress",
        zap.String("phase", phase),
        zap.Int("processed_rows", rowCount),
        zap.Int("join_type", int(e.joinType)),
        zap.String("executor_id", e.ID()),
    )
}

// Conditional debug logging
func (e *HashJoinExec) Next(ctx context.Context, req *chunk.Chunk) error {
    if logutil.BgLogger().Core().Enabled(zap.DebugLevel) {
        defer func() {
            e.debugJoinProgress("next", req.NumRows())
        }()
    }

    return e.performJoin(ctx, req)
}
```

#### Debugging with Delve
```bash
# Build with debug symbols
go build -gcflags="all=-N -l" -o tidb-server ./cmd/tidb-server

# Start debugging session
dlv --listen=:40000 --headless=true --api-version=2 --accept-multiclient exec ./tidb-server

# Or debug tests
dlv test ./pkg/executor -- -test.run TestHashJoin
```

#### Using Failpoints
```go
// Add failpoint for testing error conditions
import "github.com/pingcap/failpoint"

func (e *HashJoinExec) buildHashTable(ctx context.Context) error {
    // Failpoint for simulating memory pressure
    failpoint.Inject("hashJoinMemoryPressure", func() {
        failpoint.Return(errors.New("simulated memory pressure"))
    })

    return e.doBuildHashTable(ctx)
}

// Enable failpoints in tests
func TestHashJoinMemoryPressure(t *testing.T) {
    require.NoError(t, failpoint.Enable("github.com/pingcap/tidb/pkg/executor/hashJoinMemoryPressure", "return"))
    defer func() {
        require.NoError(t, failpoint.Disable("github.com/pingcap/tidb/pkg/executor/hashJoinMemoryPressure"))
    }()

    // Test error handling
    executor := createHashJoinExecutor()
    err := executor.buildHashTable(context.Background())
    require.Error(t, err)
    require.Contains(t, err.Error(), "simulated memory pressure")
}
```

### Performance Profiling

#### CPU Profiling
```go
// Enable CPU profiling in tests
func BenchmarkHashJoin(b *testing.B) {
    // Start CPU profiling
    f, err := os.Create("cpu.prof")
    require.NoError(b, err)
    defer f.Close()

    require.NoError(b, pprof.StartCPUProfile(f))
    defer pprof.StopCPUProfile()

    // Run benchmark
    for i := 0; i < b.N; i++ {
        performHashJoin(leftData, rightData)
    }
}

// Analyze profile
// go tool pprof cpu.prof
// (pprof) top10
// (pprof) web
```

#### Memory Profiling
```go
// Memory usage tracking
func (e *HashJoinExec) trackMemoryUsage() {
    if e.memTracker != nil {
        e.memTracker.Consume(int64(unsafe.Sizeof(*e.hashTable)))
    }
}

// Heap profiling
func BenchmarkHashJoinMemory(b *testing.B) {
    for i := 0; i < b.N; i++ {
        performHashJoin(leftData, rightData)

        if i%1000 == 0 {
            runtime.GC()
            var m runtime.MemStats
            runtime.ReadMemStats(&m)
            b.Logf("Iteration %d: Alloc = %d KB", i, m.Alloc/1024)
        }
    }
}
```

#### Execution Plan Analysis
```sql
-- Analyze query execution
EXPLAIN ANALYZE SELECT * FROM t1 JOIN t2 ON t1.id = t2.id;

-- Enable execution statistics
SET tidb_enable_collect_execution_info = ON;

-- View detailed execution information
SELECT * FROM INFORMATION_SCHEMA.STATEMENTS_SUMMARY WHERE SCHEMA_NAME = 'test';
```

## Contributing Guidelines

### Development Workflow

#### Branch Strategy
```bash
# Feature development
git checkout -b feature/join-optimization
git push origin feature/join-optimization

# Bug fixes
git checkout -b fix/memory-leak-in-hashjoin
git push origin fix/memory-leak-in-hashjoin

# Performance improvements
git checkout -b perf/vectorized-aggregation
git push origin perf/vectorized-aggregation
```

#### Commit Message Format
```
<type>(<scope>): <description>

<body>

<footer>
```

**Examples:**
```
feat(executor): add vectorized hash join implementation

- Implement SIMD-optimized hash table probing
- Add memory-efficient hash table with spilling
- Support all join types with improved performance
- Add comprehensive benchmarks and tests

Fixes #12345
Closes #12346
```

```
fix(planner): resolve memory leak in join reordering

The join reordering algorithm was not properly releasing
intermediate results, causing memory usage to grow unbounded
for complex queries with many joins.

- Add proper cleanup in JoinReorderSolver
- Implement reference counting for intermediate plans
- Add memory usage tests for complex join queries

Fixes #12347
```

### Code Review Process

#### Review Checklist
```markdown
## Code Review Checklist

### Functionality
- [ ] Code correctly implements the intended functionality
- [ ] Edge cases are properly handled
- [ ] Error conditions are appropriately managed
- [ ] Public APIs are well-designed and documented

### Performance
- [ ] No obvious performance regressions
- [ ] Memory usage is reasonable and tracked
- [ ] CPU-intensive operations are optimized
- [ ] Database operations are efficient

### Testing
- [ ] Unit tests cover new functionality
- [ ] Integration tests verify end-to-end behavior
- [ ] Edge cases and error conditions are tested
- [ ] Performance benchmarks are included if relevant

### Code Quality
- [ ] Code follows project style guidelines
- [ ] Functions are appropriately sized and focused
- [ ] Variable and function names are clear
- [ ] Comments explain complex logic

### Documentation
- [ ] Public APIs are documented
- [ ] Complex algorithms are explained
- [ ] Configuration options are documented
- [ ] Breaking changes are noted
```

#### Review Response Examples
```markdown
## Reviewer Comments

**Performance Concern:**
The hash table implementation in `buildHashTable()` allocates a new map for each partition. Consider using a pool of pre-allocated maps to reduce GC pressure.

**Suggestion:**
```go
type hashTablePool struct {
    pool sync.Pool
}

func (p *hashTablePool) get() map[string][]chunk.Row {
    if v := p.pool.Get(); v != nil {
        return v.(map[string][]chunk.Row)
    }
    return make(map[string][]chunk.Row, 1024)
}
```

**Documentation Request:**
Please add a comment explaining the join algorithm choice logic in `chooseJoinAlgorithm()`. The cost calculation is complex and would benefit from explanation.

## Author Response

**Addressed performance concern:**
Good catch! I've implemented the hash table pool as suggested. Benchmarks show 15% reduction in GC overhead for large joins.

**Added documentation:**
Added detailed comments explaining the cost model and algorithm selection criteria. Also added references to the academic papers that informed the implementation.
```

### Performance Considerations

#### Memory Management
```go
// Use object pools for frequently allocated objects
var chunkPool = sync.Pool{
    New: func() interface{} {
        return chunk.New(nil, 0, 1024) // Pre-allocate with reasonable capacity
    },
}

func (e *HashJoinExec) getChunk() *chunk.Chunk {
    chk := chunkPool.Get().(*chunk.Chunk)
    chk.Reset() // Clear previous data
    return chk
}

func (e *HashJoinExec) putChunk(chk *chunk.Chunk) {
    if chk.Capacity() <= 4096 { // Don't pool oversized chunks
        chunkPool.Put(chk)
    }
}
```

#### Goroutine Management
```go
// Use worker pools instead of unlimited goroutines
type joinWorkerPool struct {
    workers   chan *joinWorker
    ctx       context.Context
    wg        sync.WaitGroup
}

func newJoinWorkerPool(ctx context.Context, size int) *joinWorkerPool {
    pool := &joinWorkerPool{
        workers: make(chan *joinWorker, size),
        ctx:     ctx,
    }

    // Pre-create workers
    for i := 0; i < size; i++ {
        worker := &joinWorker{id: i}
        pool.workers <- worker
    }

    return pool
}

func (p *joinWorkerPool) execute(task *joinTask) error {
    select {
    case worker := <-p.workers:
        defer func() {
            p.workers <- worker // Return worker to pool
        }()
        return worker.process(task)
    case <-p.ctx.Done():
        return p.ctx.Err()
    }
}
```

#### Allocation Optimization
```go
// Pre-allocate slices with known capacity
func (e *HashJoinExec) buildJoinResult(matched []chunk.Row) *chunk.Chunk {
    // Pre-allocate based on input size estimation
    estimatedRows := len(matched)
    resultChunk := chunk.New(e.schema, estimatedRows, e.maxChunkSize)

    // Reuse buffers for column building
    if cap(e.columnBuilders) < len(e.schema.Columns) {
        e.columnBuilders = make([]*chunk.ColumnBuilder, len(e.schema.Columns))
    }

    return resultChunk
}

// Avoid allocations in hot paths
func (e *HashJoinExec) joinRows(left, right chunk.Row) bool {
    // Use stack-allocated arrays for small comparisons
    const maxStackSize = 8
    var stackBuf [maxStackSize]types.Datum

    if len(e.joinKeys) <= maxStackSize {
        datums := stackBuf[:len(e.joinKeys)]
        return e.compareRowsWithBuf(left, right, datums)
    }

    // Fall back to heap allocation for large comparisons
    datums := make([]types.Datum, len(e.joinKeys))
    return e.compareRowsWithBuf(left, right, datums)
}
```

This development guide provides the foundation for effective TiDB development, covering everything from environment setup to advanced performance optimization techniques. Following these guidelines will help maintain code quality and ensure consistent development practices across the project.