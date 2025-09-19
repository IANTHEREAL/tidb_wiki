# TiDB Development Best Practices

This guide outlines development best practices for contributing to TiDB, covering code style, testing strategies, performance optimization, and security considerations.

## Table of Contents

1. [Code Style Guidelines](#1-code-style-guidelines)
2. [Testing Strategies](#2-testing-strategies)
3. [Performance Optimization](#3-performance-optimization)
4. [Security Considerations](#4-security-considerations)
5. [Error Handling](#5-error-handling)
6. [Documentation Standards](#6-documentation-standards)
7. [Code Review Guidelines](#7-code-review-guidelines)
8. [Commit and PR Practices](#8-commit-and-pr-practices)

## 1. Code Style Guidelines

### 1.1 Go Language Standards

Follow the official [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments) and these TiDB-specific guidelines:

#### Naming Conventions
```go
// Good: Clear, descriptive names
type QueryOptimizer struct {
    costModel CostModel
    statsInfo *statistics.Table
}

func (q *QueryOptimizer) OptimizeLogicalPlan(plan LogicalPlan) (PhysicalPlan, error) {
    // Implementation
}

// Bad: Unclear abbreviations
type QO struct {
    cm CM
    si *SI
}

func (q *QO) OLP(p LP) (PP, error) {
    // Implementation
}
```

#### Package Organization
```go
// Good: Logical grouping with clear dependencies
pkg/
├── planner/
│   ├── core/          # Core planning logic
│   ├── cardinality/   # Cardinality estimation
│   └── property/      # Physical properties
├── executor/
│   ├── join/          # Join operators
│   └── aggregate/     # Aggregation operators
└── statistics/
    ├── histogram.go   # Histogram implementation
    └── table.go       # Table statistics
```

#### Interface Design
```go
// Good: Small, focused interfaces
type Optimizer interface {
    Optimize(ctx context.Context, plan LogicalPlan) (PhysicalPlan, error)
}

type CostModel interface {
    GetCost(plan PhysicalPlan) float64
}

// Good: Composition over large interfaces
type PlanBuilder interface {
    Optimizer
    CostModel
}
```

### 1.2 File Structure and Imports

#### Import Organization
```go
package core

import (
    // Standard library first
    "context"
    "fmt"
    "sort"
    
    // Third-party packages
    "github.com/pingcap/errors"
    "go.uber.org/zap"
    
    // TiDB packages (grouped by function)
    "github.com/pingcap/tidb/pkg/expression"
    "github.com/pingcap/tidb/pkg/parser/ast"
    "github.com/pingcap/tidb/pkg/types"
    
    // Local package imports last
    "github.com/pingcap/tidb/pkg/planner/property"
    "github.com/pingcap/tidb/pkg/planner/util"
)
```

#### File Organization
```go
// file_name.go structure:
// 1. Package declaration and imports
// 2. Constants and package variables
// 3. Type definitions
// 4. Constructor functions
// 5. Interface implementations
// 6. Public methods
// 7. Private methods
// 8. Helper functions

const (
    DefaultTimeout = 30 * time.Second
    MaxRetries     = 3
)

var (
    ErrInvalidPlan = errors.New("invalid plan")
)

type PlanOptimizer struct {
    // Fields
}

func NewPlanOptimizer(config Config) *PlanOptimizer {
    // Constructor
}

func (p *PlanOptimizer) PublicMethod() error {
    // Public implementation
}

func (p *PlanOptimizer) privateMethod() error {
    // Private implementation
}
```

### 1.3 Error Handling Patterns

#### Structured Error Handling
```go
// Good: Use errors package for structured errors
func (p *PlanBuilder) buildSelection(node *ast.SelectStmt) (LogicalPlan, error) {
    if node == nil {
        return nil, errors.New("selection node cannot be nil")
    }
    
    plan, err := p.buildTableScan(node.From)
    if err != nil {
        return nil, errors.Trace(err)
    }
    
    if node.Where != nil {
        selection, err := p.buildFilter(node.Where)
        if err != nil {
            return nil, errors.Annotate(err, "failed to build WHERE clause")
        }
        selection.SetChildren(plan)
        return selection, nil
    }
    
    return plan, nil
}

// Good: Define domain-specific errors
var (
    ErrUnsupportedType = dbterror.ClassOptimizer.NewStd(mysql.ErrUnsupportedType)
    ErrInvalidConfig   = dbterror.ClassOptimizer.NewStd(mysql.ErrInvalidConfig)
)
```

### 1.4 Comments and Documentation

#### Function Documentation
```go
// OptimizeLogicalPlan transforms a logical plan into an optimized physical plan.
// It applies rule-based optimizations followed by cost-based optimization.
//
// Parameters:
//   ctx: Context for cancellation and timeouts
//   logical: The input logical plan to optimize
//   required: Required physical properties (sort order, distribution)
//
// Returns:
//   The optimized physical plan and any error encountered during optimization.
//
// The optimization process consists of:
//  1. Rule-based logical transformations (predicate pushdown, column pruning)
//  2. Physical plan enumeration for each logical operator
//  3. Cost-based selection of the optimal physical plan
func (o *Optimizer) OptimizeLogicalPlan(ctx context.Context, logical LogicalPlan, required *property.PhysicalProperty) (PhysicalPlan, error) {
    // Implementation
}
```

#### Code Comments
```go
func (ds *DataSource) findBestTask(prop *property.PhysicalProperty) (task, error) {
    // Generate all possible access paths for this data source.
    // This includes table scan, index scans, and index merges.
    paths := ds.generateAccessPaths()
    
    var bestTask task
    var bestCost float64 = math.MaxFloat64
    
    for _, path := range paths {
        // Calculate cost for this access path considering:
        // 1. I/O cost for reading data
        // 2. CPU cost for processing
        // 3. Network cost for distributed execution
        cost := ds.calculatePathCost(path, prop)
        
        if cost < bestCost {
            bestCost = cost
            bestTask = ds.convertToTask(path, prop)
        }
    }
    
    return bestTask, nil
}
```

## 2. Testing Strategies

### 2.1 Test Organization

#### Test File Structure
```go
// plan_test.go - follow naming convention: <file>_test.go
package core

import (
    "testing"
    
    "github.com/stretchr/testify/require"
    "github.com/pingcap/tidb/pkg/testkit"
)

// Test naming: Test<Function><Scenario>
func TestPlanBuilderSelectBasic(t *testing.T) {
    // Basic SELECT statement planning
}

func TestPlanBuilderSelectWithJoin(t *testing.T) {
    // SELECT with JOIN planning
}

func TestPlanBuilderSelectError(t *testing.T) {
    // Error cases in SELECT planning
}
```

#### Table-Driven Tests
```go
func TestSelectivityCalculation(t *testing.T) {
    tests := []struct {
        name     string
        sql      string
        expected float64
        hasError bool
    }{
        {
            name:     "simple equality",
            sql:      "SELECT * FROM t WHERE a = 1",
            expected: 0.1,
            hasError: false,
        },
        {
            name:     "range condition",
            sql:      "SELECT * FROM t WHERE a > 1 AND a < 100",
            expected: 0.3,
            hasError: false,
        },
        {
            name:     "invalid column",
            sql:      "SELECT * FROM t WHERE invalid_col = 1",
            expected: 0,
            hasError: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := calculateSelectivity(tt.sql)
            
            if tt.hasError {
                require.Error(t, err)
                return
            }
            
            require.NoError(t, err)
            require.InDelta(t, tt.expected, result, 0.01)
        })
    }
}
```

### 2.2 Integration Testing

#### Using TestKit
```go
func TestQueryExecution(t *testing.T) {
    store := testkit.CreateMockStore(t)
    tk := testkit.NewTestKit(t, store)
    
    // Setup test data
    tk.MustExec("use test")
    tk.MustExec("create table t (id int primary key, name varchar(50))")
    tk.MustExec("insert into t values (1, 'Alice'), (2, 'Bob')")
    
    // Test query execution
    result := tk.MustQuery("select * from t where id = 1")
    result.Check(testkit.Rows("1 Alice"))
    
    // Test with parameters
    tk.MustExec("prepare stmt from 'select * from t where id = ?'")
    tk.MustExec("set @id = 2")
    result = tk.MustQuery("execute stmt using @id")
    result.Check(testkit.Rows("2 Bob"))
}
```

### 2.3 Unit Testing Guidelines

#### Test Independence
```go
// Good: Each test is independent
func TestPlanCacheHit(t *testing.T) {
    cache := newPlanCache()
    plan := &mockPlan{id: 1}
    
    cache.Put("key1", plan)
    result, found := cache.Get("key1")
    
    require.True(t, found)
    require.Equal(t, plan, result)
}

func TestPlanCacheMiss(t *testing.T) {
    cache := newPlanCache()
    
    result, found := cache.Get("nonexistent")
    
    require.False(t, found)
    require.Nil(t, result)
}
```

#### Mock Usage
```go
// Good: Use interfaces for mocking
type MockStatistics struct {
    histograms map[int64]*statistics.Histogram
}

func (m *MockStatistics) GetHistogram(colID int64) *statistics.Histogram {
    return m.histograms[colID]
}

func TestCardinalityEstimation(t *testing.T) {
    mockStats := &MockStatistics{
        histograms: map[int64]*statistics.Histogram{
            1: createMockHistogram(1000, 100), // 1000 rows, 100 distinct values
        },
    }
    
    estimator := NewCardinalityEstimator(mockStats)
    result := estimator.EstimateEqual(1, types.NewIntDatum(42))
    
    require.InDelta(t, 10.0, result, 0.1) // 1000/100 = 10
}
```

### 2.4 Benchmark Testing

#### Performance Benchmarks
```go
func BenchmarkPlanBuilding(b *testing.B) {
    sql := "SELECT * FROM t1 JOIN t2 ON t1.id = t2.id WHERE t1.value > 100"
    parser := parser.New()
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        stmt, err := parser.ParseOneStmt(sql, "", "")
        if err != nil {
            b.Fatal(err)
        }
        
        builder := NewPlanBuilder()
        _, err = builder.Build(context.Background(), stmt)
        if err != nil {
            b.Fatal(err)
        }
    }
}

// Benchmark with different input sizes
func BenchmarkJoinOptimization(b *testing.B) {
    testCases := []struct {
        name      string
        tableCount int
    }{
        {"2 tables", 2},
        {"4 tables", 4},
        {"8 tables", 8},
    }
    
    for _, tc := range testCases {
        b.Run(tc.name, func(b *testing.B) {
            sql := generateJoinSQL(tc.tableCount)
            b.ResetTimer()
            
            for i := 0; i < b.N; i++ {
                optimizeQuery(sql)
            }
        })
    }
}
```

## 3. Performance Optimization

### 3.1 Memory Management

#### Avoid Memory Leaks
```go
// Good: Proper cleanup and resource management
func (p *PlanBuilder) buildLogicalPlan(node ast.Node) (LogicalPlan, error) {
    // Use object pools for frequently allocated objects
    expr := p.exprPool.Get().(*expression.Expression)
    defer p.exprPool.Put(expr)
    
    // Reuse slices when possible
    if cap(p.tempSlice) < requiredCap {
        p.tempSlice = make([]LogicalPlan, 0, requiredCap)
    }
    p.tempSlice = p.tempSlice[:0] // Reset length, keep capacity
    
    return plan, nil
}

// Good: Use streaming processing for large datasets
func processLargeResultSet(rows []Row) error {
    const batchSize = 1000
    
    for i := 0; i < len(rows); i += batchSize {
        end := min(i+batchSize, len(rows))
        batch := rows[i:end]
        
        if err := processBatch(batch); err != nil {
            return err
        }
        
        // Allow GC to reclaim processed rows
        for j := i; j < end; j++ {
            rows[j] = nil
        }
    }
    
    return nil
}
```

#### Efficient Data Structures
```go
// Good: Use appropriate data structures
type ColumnSet struct {
    // Use bitset for efficient set operations on column IDs
    bits *bitset.BitSet
}

func (cs *ColumnSet) Contains(colID int) bool {
    return cs.bits.Test(uint(colID))
}

func (cs *ColumnSet) Union(other *ColumnSet) *ColumnSet {
    result := &ColumnSet{bits: cs.bits.Clone()}
    result.bits.InPlaceUnion(other.bits)
    return result
}

// Good: Pre-allocate slices when size is known
func generateAccessPaths(table *model.TableInfo) []*AccessPath {
    // Pre-allocate with known capacity
    paths := make([]*AccessPath, 0, len(table.Indices)+1) // +1 for table scan
    
    // Add table scan path
    paths = append(paths, &AccessPath{IsTablePath: true})
    
    // Add index paths
    for _, idx := range table.Indices {
        paths = append(paths, &AccessPath{Index: idx})
    }
    
    return paths
}
```

### 3.2 Algorithm Optimization

#### Efficient Algorithms
```go
// Good: Use efficient algorithms for common operations
func (js *JoinSolver) findOptimalJoinOrder(tables []LogicalPlan) LogicalPlan {
    n := len(tables)
    
    // Use dynamic programming for small number of tables
    if n <= 6 {
        return js.dpJoinOrder(tables)
    }
    
    // Use greedy algorithm for larger sets
    return js.greedyJoinOrder(tables)
}

// Good: Cache expensive computations
type CostModel struct {
    costCache map[string]float64
    mutex     sync.RWMutex
}

func (cm *CostModel) GetPlanCost(plan PhysicalPlan) float64 {
    key := plan.HashCode()
    
    cm.mutex.RLock()
    if cost, exists := cm.costCache[key]; exists {
        cm.mutex.RUnlock()
        return cost
    }
    cm.mutex.RUnlock()
    
    cost := cm.calculateCost(plan)
    
    cm.mutex.Lock()
    cm.costCache[key] = cost
    cm.mutex.Unlock()
    
    return cost
}
```

### 3.3 Concurrency Optimization

#### Safe Concurrent Access
```go
// Good: Use appropriate synchronization
type StatisticsCache struct {
    cache map[int64]*statistics.Table
    mutex sync.RWMutex
}

func (sc *StatisticsCache) Get(tableID int64) (*statistics.Table, bool) {
    sc.mutex.RLock()
    defer sc.mutex.RUnlock()
    
    stats, exists := sc.cache[tableID]
    return stats, exists
}

func (sc *StatisticsCache) Update(tableID int64, stats *statistics.Table) {
    sc.mutex.Lock()
    defer sc.mutex.Unlock()
    
    sc.cache[tableID] = stats
}

// Good: Use channels for coordination
func (p *ParallelProcessor) ProcessBatches(batches []Batch) error {
    resultChan := make(chan result, len(batches))
    errorChan := make(chan error, 1)
    
    // Start workers
    var wg sync.WaitGroup
    for _, batch := range batches {
        wg.Add(1)
        go func(b Batch) {
            defer wg.Done()
            
            result, err := p.processBatch(b)
            if err != nil {
                select {
                case errorChan <- err:
                default: // Error channel is buffered(1), ignore if full
                }
                return
            }
            
            resultChan <- result
        }(batch)
    }
    
    // Wait for completion
    go func() {
        wg.Wait()
        close(resultChan)
    }()
    
    // Collect results
    var results []result
    for {
        select {
        case r, ok := <-resultChan:
            if !ok {
                return nil // All done
            }
            results = append(results, r)
            
        case err := <-errorChan:
            return err
        }
    }
}
```

## 4. Security Considerations

### 4.1 Input Validation

#### SQL Injection Prevention
```go
// Good: Use parameterized queries and validation
func (e *Executor) executeQuery(sql string, params []interface{}) ([]Row, error) {
    // Validate input parameters
    for i, param := range params {
        if err := validateParameter(param); err != nil {
            return nil, fmt.Errorf("invalid parameter at position %d: %w", i, err)
        }
    }
    
    // Use prepared statements
    stmt, err := e.prepare(sql)
    if err != nil {
        return nil, err
    }
    defer stmt.Close()
    
    return stmt.Execute(params)
}

func validateParameter(param interface{}) error {
    switch v := param.(type) {
    case string:
        // Validate string length and content
        if len(v) > maxStringLength {
            return errors.New("string parameter too long")
        }
        if containsSQLKeywords(v) {
            return errors.New("string parameter contains SQL keywords")
        }
    case int64:
        // Validate numeric ranges
        if v < minInt64 || v > maxInt64 {
            return errors.New("numeric parameter out of range")
        }
    }
    return nil
}
```

### 4.2 Access Control

#### Privilege Checking
```go
// Good: Consistent privilege checking
func (p *PlanBuilder) buildSelect(sel *ast.SelectStmt) (LogicalPlan, error) {
    // Check SELECT privilege on all referenced tables
    for _, table := range extractTables(sel) {
        if !p.ctx.CheckPrivilege(mysql.SelectPriv, table.Schema.L, table.Name.L) {
            return nil, ErrTableaccessDenied.GenWithStackByArgs("SELECT", 
                p.ctx.GetSessionVars().User.Hostname,
                p.ctx.GetSessionVars().User.Username,
                table.Name.L)
        }
    }
    
    return p.buildLogicalPlan(sel)
}

// Good: Row-level security
func (ds *DataSource) addRowLevelSecurityFilter(ctx PlanContext) error {
    if !ds.table.HasRowLevelSecurity() {
        return nil
    }
    
    user := ctx.GetSessionVars().User
    securityPolicy, err := ds.table.GetRowLevelSecurityPolicy(user)
    if err != nil {
        return err
    }
    
    if securityPolicy.FilterExpr != nil {
        ds.pushedDownConds = append(ds.pushedDownConds, securityPolicy.FilterExpr)
    }
    
    return nil
}
```

### 4.3 Resource Limits

#### Memory and CPU Limits
```go
// Good: Enforce resource limits
func (e *Executor) executeWithLimits(ctx context.Context, plan PhysicalPlan) error {
    // Set memory limit
    memTracker := memory.NewTracker(plan.ID(), e.ctx.GetSessionVars().MemQuotaQuery)
    defer memTracker.Detach()
    
    // Set execution timeout
    execCtx, cancel := context.WithTimeout(ctx, e.ctx.GetSessionVars().MaxExecutionTime)
    defer cancel()
    
    // Monitor resource usage
    done := make(chan error, 1)
    go func() {
        done <- e.executeInternal(execCtx, plan, memTracker)
    }()
    
    select {
    case err := <-done:
        return err
    case <-execCtx.Done():
        return ErrQueryTimeout.GenWithStackByArgs(e.ctx.GetSessionVars().MaxExecutionTime)
    }
}
```

## 5. Error Handling

### 5.1 Error Classification

#### Structured Error Types
```go
// Good: Define clear error hierarchy
var (
    // Parse errors
    ErrSyntaxError = dbterror.ClassParser.NewStd(mysql.ErrSyntaxError)
    ErrInvalidSQL  = dbterror.ClassParser.NewStd(mysql.ErrInvalidSQL)
    
    // Planning errors
    ErrTableNotFound = dbterror.ClassOptimizer.NewStd(mysql.ErrNoSuchTable)
    ErrColumnNotFound = dbterror.ClassOptimizer.NewStd(mysql.ErrBadFieldError)
    
    // Execution errors
    ErrDivisionByZero = dbterror.ClassExpression.NewStd(mysql.ErrDivisionByZero)
    ErrDataTooLong = dbterror.ClassTypes.NewStd(mysql.ErrDataTooLong)
)

// Good: Context-aware error handling
func (b *PlanBuilder) buildTableRef(node *ast.TableRefsClause) (LogicalPlan, error) {
    plan, err := b.buildTableSource(node.TableRefs)
    if err != nil {
        // Add context to error
        return nil, errors.Annotate(err, "failed to build table reference")
    }
    
    return plan, nil
}
```

### 5.2 Error Recovery

#### Graceful Degradation
```go
// Good: Fallback mechanisms
func (o *Optimizer) optimizeLogicalPlan(plan LogicalPlan) (PhysicalPlan, error) {
    // Try advanced optimization first
    if result, err := o.advancedOptimize(plan); err == nil {
        return result, nil
    }
    
    // Fall back to basic optimization if advanced fails
    logutil.BgLogger().Warn("advanced optimization failed, using basic optimization",
        zap.Error(err))
    
    return o.basicOptimize(plan)
}

// Good: Resource cleanup on error
func (e *Executor) executeJoin(ctx context.Context, join *PhysicalHashJoin) error {
    hashTable := newHashTable()
    defer hashTable.Close() // Always cleanup
    
    buildSide, err := e.executeChild(ctx, join.Children()[0])
    if err != nil {
        return errors.Trace(err)
    }
    defer buildSide.Close()
    
    // Continue with join execution...
    return nil
}
```

## 6. Documentation Standards

### 6.1 API Documentation

#### Interface Documentation
```go
// Optimizer defines the interface for query optimization.
// 
// Implementations should:
//  - Apply rule-based optimizations before cost-based optimization
//  - Use statistics for cardinality estimation when available
//  - Handle cancellation through the provided context
//  - Return an error if optimization fails for any reason
//
// Thread Safety: Implementations must be safe for concurrent use.
type Optimizer interface {
    // OptimizeLogicalPlan transforms a logical plan to an optimal physical plan.
    //
    // The optimization process considers the required physical properties
    // (such as sort order and data distribution) and uses cost-based
    // selection to choose the most efficient execution strategy.
    //
    // Parameters:
    //   ctx: Context for cancellation and timeouts
    //   logical: Input logical plan to optimize
    //   required: Required physical properties for the result
    //
    // Returns:
    //   Optimized physical plan and any error encountered
    OptimizeLogicalPlan(ctx context.Context, logical LogicalPlan, required *property.PhysicalProperty) (PhysicalPlan, error)
}
```

### 6.2 Code Examples

#### Include Usage Examples
```go
// Example usage:
//
//   // Create optimizer with statistics
//   stats := statistics.NewTable(tableInfo, 1000)
//   optimizer := NewCostBasedOptimizer(stats)
//   
//   // Optimize query plan
//   logical := buildLogicalPlan(sql)
//   required := &property.PhysicalProperty{
//       SortItems: []property.SortItem{{Col: col1, Desc: false}},
//   }
//   
//   physical, err := optimizer.OptimizeLogicalPlan(ctx, logical, required)
//   if err != nil {
//       return err
//   }
//   
//   // Execute optimized plan
//   return executor.Execute(ctx, physical)
func NewCostBasedOptimizer(stats *statistics.Table) Optimizer {
    // Implementation
}
```

## 7. Code Review Guidelines

### 7.1 Review Checklist

#### For Reviewers
- [ ] **Correctness**: Does the code solve the intended problem?
- [ ] **Performance**: Are there any performance regressions?
- [ ] **Security**: Are there security vulnerabilities?
- [ ] **Testing**: Is test coverage adequate?
- [ ] **Documentation**: Are public APIs documented?
- [ ] **Style**: Does code follow Go and TiDB conventions?
- [ ] **Error Handling**: Are errors handled appropriately?
- [ ] **Dependencies**: Are new dependencies justified?

#### For Authors
- [ ] **Self Review**: Review your own code before submitting
- [ ] **Tests Pass**: All tests pass locally
- [ ] **Linting**: Code passes all linting checks
- [ ] **Documentation**: Public APIs are documented
- [ ] **Backwards Compatibility**: Changes maintain compatibility
- [ ] **Performance**: No significant performance regressions

### 7.2 Review Comments

#### Constructive Feedback
```
// Good: Specific, actionable feedback
"Consider using a sync.RWMutex here instead of sync.Mutex since 
reads are much more frequent than writes. This should improve 
performance for the common case."

// Good: Explain the reasoning
"This function allocates a new slice on every call. Since the 
maximum size is known, consider pre-allocating with make([]Type, 0, maxSize) 
to reduce GC pressure."

// Bad: Vague or non-constructive
"This is wrong."
"Bad performance."
```

## 8. Commit and PR Practices

### 8.1 Commit Message Format

#### Conventional Commits
```
type(scope): description

[optional body]

[optional footer]

Examples:
feat(planner): add support for window functions in cost model
fix(executor): handle null values correctly in hash join
docs(wiki): update getting started guide with Go 1.21 requirements
test(statistics): add benchmark tests for histogram operations
refactor(parser): simplify AST node creation logic
```

#### Commit Types
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting, etc.)
- `refactor`: Code refactoring without behavior change
- `test`: Adding or updating tests
- `perf`: Performance improvements
- `chore`: Build system, CI, etc.

### 8.2 Pull Request Guidelines

#### PR Template
```markdown
## Description
Brief description of changes and motivation.

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Benchmark tests added/updated (if performance-related)
- [ ] Manual testing performed

## Checklist
- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Documentation updated (if needed)
- [ ] No new warnings introduced
- [ ] Related issues linked

## Performance Impact
Describe any performance implications of these changes.

## Breaking Changes
List any breaking changes and migration path (if applicable).
```

#### PR Size Guidelines
- **Small PRs** (< 200 lines): Preferred for most changes
- **Medium PRs** (200-500 lines): Acceptable with good description
- **Large PRs** (> 500 lines): Should be broken down when possible

---

Following these best practices will help maintain TiDB's code quality, performance, and security standards. For questions about specific practices, consult the [Architecture Analysis](architecture_analysis.md) or ask in [GitHub Discussions](https://github.com/pingcap/tidb/discussions).