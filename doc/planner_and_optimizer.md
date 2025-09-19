# TiDB Query Planning and Optimization

This document provides comprehensive documentation for TiDB's query planning and optimization modules, covering the complete transformation from SQL AST to executable physical plans.

## Table of Contents

1. [Planner Module Deep Dive](#1-planner-module-deep-dive)
2. [Optimization Rules](#2-optimization-rules)
3. [Physical Plan Generation](#3-physical-plan-generation)
4. [Statistics Module](#4-statistics-module)
5. [Practical Examples](#5-practical-examples)
6. [Performance Tuning](#6-performance-tuning)

## 1. Planner Module Deep Dive

### 1.1 Architecture Overview

The TiDB planner is located in `pkg/planner/` and consists of several key components:

```
pkg/planner/
├── core/               # Core planning logic
├── cardinality/        # Cardinality estimation
├── cascades/           # Cascades optimizer framework
├── property/           # Physical properties
├── util/              # Utility functions
└── optimize.go        # Main optimization entry point
```

### 1.2 Logical Plan Generation from AST

**Entry Point**: `pkg/planner/core/logical_plan_builder.go:1`

The logical plan generation process follows these steps:

#### Step 1: AST to Logical Plan Conversion
```go
// Located in pkg/planner/core/logical_plan_builder.go
func (b *PlanBuilder) Build(ctx context.Context, node ast.Node) (Plan, error)
```

**Key Components**:
- **PlanBuilder**: Main builder struct that converts AST nodes to logical plans
- **Logical Operators**: Representation of SQL operations in logical form
- **Schema Building**: Column and table schema resolution

#### Step 2: Plan Node Types
The logical plan consists of various operator types:

| Operator | File Location | Purpose |
|----------|---------------|---------|
| LogicalSelection | `pkg/planner/core/operator/logicalop/` | WHERE clause filtering |
| LogicalProjection | `pkg/planner/core/operator/logicalop/` | SELECT column projection |
| LogicalJoin | `pkg/planner/core/operator/logicalop/` | JOIN operations |
| LogicalAggregation | `pkg/planner/core/operator/logicalop/` | GROUP BY and aggregations |
| LogicalSort | `pkg/planner/core/operator/logicalop/` | ORDER BY operations |
| LogicalLimit | `pkg/planner/core/operator/logicalop/` | LIMIT/OFFSET operations |

### 1.3 Cost-Based Optimization Framework

**Primary Files**:
- `pkg/planner/core/find_best_task.go:1` - Task selection logic
- `pkg/planner/core/plan_cost_ver2.go:1` - Cost calculation v2

#### Cost Model Architecture

TiDB uses a sophisticated two-version cost model:

**Cost Model V1** (`plan_cost_ver1.go`):
- Simple operator-based cost calculation
- Basic CPU and I/O cost factors
- Limited cardinality estimation

**Cost Model V2** (`plan_cost_ver2.go:41`):
```go
func GetPlanCost(p base.PhysicalPlan, taskType property.TaskType, option *optimizetrace.PlanCostOption) (float64, error)
```

**Cost Components**:
1. **CPU Cost**: Processing overhead for operators
2. **Memory Cost**: Memory consumption for operations
3. **I/O Cost**: Disk and network access costs
4. **Network Cost**: Data transmission between nodes

#### Cost Calculation Process

```go
// Example cost calculation for PhysicalSelection
func getPlanCostVer24PhysicalSelection(pp base.PhysicalPlan, taskType property.TaskType, option *optimizetrace.PlanCostOption) (costusage.CostVer2, error) {
    p := pp.(*physicalop.PhysicalSelection)
    inputRows := getCardinality(p.Children()[0], option.CostFlag)
    cpuFactor := getTaskCPUFactorVer2(p, taskType)
    
    filterCost := filterCostVer2(option, inputRows, p.Conditions, cpuFactor)
    childCost, err := p.Children()[0].GetPlanCostVer2(taskType, option)
    
    return costusage.SumCostVer2(filterCost, childCost), nil
}
```

### 1.4 Rule-Based Transformations

**Rule Engine Location**: `pkg/planner/core/optimizer.go:89`

```go
var optRuleList = []base.LogicalOptRule{
    &GcSubstituter{},
    &rule.ColumnPruner{},
    &ResultReorder{},
    &rule.BuildKeySolver{},
    &DecorrelateSolver{},
    &SemiJoinRewriter{},
    &AggregationEliminator{},
    &SkewDistinctAggRewriter{},
    &ProjectionEliminator{},
    &MaxMinEliminator{},
    &rule.ConstantPropagationSolver{},
    // ... more rules
}
```

### 1.5 Statistics-Driven Decisions

**Integration Point**: `pkg/planner/core/stats.go:1`

The optimizer uses statistics for:
- **Cardinality Estimation**: Row count predictions
- **Selectivity Calculation**: Filter effectiveness
- **Join Order Selection**: Cost-based join reordering
- **Index Selection**: Optimal access path selection

### 1.6 Plan Caching Mechanisms

**Plan Cache Implementation**: `pkg/planner/core/plan_cache.go:1`

TiDB supports multiple plan caching strategies:

#### Prepared Statement Cache
```go
// Located in plan_cache.go
func (e *PreparedStmt) PointGet(is infoschema.InfoSchema) *PointGetPlan
```

#### Non-Prepared Plan Cache
```go
// Located in pkg/planner/optimize.go:75
func getPlanFromNonPreparedPlanCache(ctx context.Context, sctx sessionctx.Context, node *resolve.NodeW, is infoschema.InfoSchema) (p base.Plan, ns types.NameSlice, ok bool, err error)
```

**Cache Key Components**:
- SQL text fingerprint
- Schema version
- Session variables
- System variables affecting optimization

## 2. Optimization Rules

### 2.1 Column Pruning

**Implementation**: `pkg/planner/core/rule/rule_column_pruning.go:1`

**Purpose**: Remove unnecessary columns from the query plan to reduce I/O and memory usage.

**Algorithm**:
1. Start from the root of the plan tree
2. Collect required columns from parent operators
3. Propagate requirements down the tree
4. Remove unused columns from each operator

**Example Transformation**:
```sql
-- Original Query
SELECT name FROM users WHERE age > 25;

-- Before Column Pruning: Reads (id, name, age, email, created_at)
-- After Column Pruning: Reads only (name, age)
```

### 2.2 Predicate Pushdown

**Implementation**: `pkg/planner/core/rule_predicate_push_down.go:43`

**PPDSolver Structure**:
```go
type PPDSolver struct{}

func (*PPDSolver) Optimize(_ context.Context, lp base.LogicalPlan, opt *optimizetrace.LogicalOptimizeOp) (base.LogicalPlan, bool, error) {
    planChanged := false
    _, p, err := lp.PredicatePushDown(nil, opt)
    return p, planChanged, err
}
```

**Predicate Pushdown Benefits**:
- **Early Filtering**: Reduces intermediate result sets
- **Index Utilization**: Enables index-based filtering
- **Network Reduction**: Less data transfer between nodes

**Example**:
```sql
-- Original Query
SELECT u.name, d.department 
FROM users u 
JOIN departments d ON u.dept_id = d.id 
WHERE u.age > 30;

-- After Predicate Pushdown
-- Filter "age > 30" is pushed to users table scan
-- Reducing join input size significantly
```

### 2.3 Join Reordering Algorithms

**Implementation**: `pkg/planner/core/rule_join_reorder.go:1`

TiDB implements multiple join reordering algorithms:

#### Dynamic Programming Algorithm
**File**: `pkg/planner/core/rule_join_reorder_dp.go:1`

```go
func (s *joinReorderDPSolver) solve(joinGroup []base.LogicalPlan, eqEdges []expression.ScalarFunction) (base.LogicalPlan, error)
```

**Algorithm Steps**:
1. **Subset Enumeration**: Generate all possible join subsets
2. **Cost Calculation**: Compute cost for each join order
3. **Optimal Selection**: Choose minimum cost configuration
4. **Memoization**: Cache intermediate results

#### Greedy Algorithm
**File**: `pkg/planner/core/rule_join_reorder_greedy.go:1`

**When Used**: For queries with many tables (>6) where DP becomes expensive

**Algorithm**:
1. Start with the smallest estimated cardinality table
2. Iteratively add the next best table to join
3. Consider join selectivity and result cardinality

### 2.4 Index Selection Strategies

**Implementation**: `pkg/planner/core/find_best_task.go:1`

**Index Selection Process**:

#### Step 1: Access Path Generation
```go
// Generate all possible access paths for a table
func (ds *DataSource) DeriveStats(childStats []*property.StatsInfo, selfSchema *expression.Schema, childSchema []*expression.Schema, colGroups [][]*expression.Column) (*property.StatsInfo, error)
```

#### Step 2: Cost Comparison
- **Table Scan Cost**: Full table read cost
- **Index Scan Cost**: Index read + potential table lookup cost
- **Index Merge Cost**: Multiple index combination cost

#### Index Selection Factors:
1. **Selectivity**: How many rows the index can filter
2. **Covering**: Whether index covers all required columns
3. **Order**: Whether index provides required sort order
4. **Cardinality**: Number of distinct values in index

### 2.5 Subquery Optimization

**Implementation**: `pkg/planner/core/rule_decorrelate.go:1`

**Optimization Techniques**:

#### Decorrelation
```go
type DecorrelateSolver struct{}

func (s *DecorrelateSolver) Optimize(ctx context.Context, p base.LogicalPlan, opt *optimizetrace.LogicalOptimizeOp) (base.LogicalPlan, bool, error)
```

**Transformations**:
- **EXISTS to Semi-Join**: Convert EXISTS subqueries to semi-joins
- **IN to Join**: Transform IN subqueries to inner joins
- **Scalar Subquery Optimization**: Cache scalar subquery results

#### Example Decorrelation:
```sql
-- Original Correlated Subquery
SELECT * FROM orders o 
WHERE EXISTS (SELECT 1 FROM order_items oi WHERE oi.order_id = o.id);

-- After Decorrelation (Semi-Join)
SELECT DISTINCT o.* FROM orders o 
SEMI JOIN order_items oi ON oi.order_id = o.id;
```

## 3. Physical Plan Generation

### 3.1 Access Path Selection

**Implementation**: `pkg/planner/core/exhaust_physical_plans.go:54`

The physical plan generation follows this process:

```go
func exhaustPhysicalPlans(lp base.LogicalPlan, prop *property.PhysicalProperty) ([]base.PhysicalPlan, bool, error) {
    switch x := lp.(type) {
    case *logicalop.LogicalSelection:
        return physicalop.ExhaustPhysicalPlans4LogicalSelection(x, prop)
    case *logicalop.LogicalJoin:
        return exhaustPhysicalPlans4LogicalJoin(x, prop)
    // ... other operators
    }
}
```

**Access Path Types**:
1. **Table Scan**: Sequential read of table data
2. **Index Scan**: Index-based data access
3. **Index Lookup**: Index scan + table lookup for non-covering queries
4. **Index Merge**: Combining multiple index scans

### 3.2 Join Algorithm Selection

**Join Algorithm Implementation**: `pkg/planner/core/exhaust_physical_plans.go:99`

#### Hash Join Selection
```go
func getHashJoins(super base.LogicalPlan, prop *property.PhysicalProperty) (joins []base.PhysicalPlan, forced bool) {
    ge, p := base.GetGEAndLogical[*logicalop.LogicalJoin](super)
    
    // Hash join doesn't promise any orders
    if !prop.IsSortItemEmpty() {
        return
    }
    
    // Determine build/probe side based on hints and cardinality
    forceLeftToBuild := ((p.PreferJoinType & h.PreferLeftAsHJBuild) > 0)
    forceRightToBuild := ((p.PreferJoinType & h.PreferRightAsHJBuild) > 0)
    
    // Generate hash join plans
    switch p.JoinType {
    case base.SemiJoin, base.AntiSemiJoin:
        joins = append(joins, getHashJoin(ge, p, prop, 1, false))
    // ... other join types
    }
}
```

#### Join Algorithm Characteristics:

| Algorithm | Best Use Case | Properties | Implementation |
|-----------|---------------|------------|----------------|
| **Hash Join** | Large tables, no sort order needed | O(M+N), Memory intensive | `getHashJoins()` |
| **Merge Join** | Pre-sorted inputs, order preservation | O(M+N), Memory efficient | `getMergeJoins()` |
| **Index Join** | One small table, available index | O(M*log N), Index dependent | `getIndexJoins()` |
| **Nested Loop Join** | Very small inner table | O(M*N), Simple | Fallback option |

### 3.3 Distributed Execution Planning

**MPP Planning**: `pkg/planner/core/task.go:1`

**Task Types**:
```go
type TaskType int

const (
    RootTaskType TaskType = iota
    CopTaskType           // Coprocessor task (TiKV)
    MppTaskType           // MPP task (TiFlash)
)
```

#### MPP Task Generation
```go
type MppTask struct {
    p           base.PhysicalPlan
    partTp      property.MPPPartitionType
    hashCols    []*expression.Column
    cstoreNum   int
}
```

**MPP Optimization Features**:
- **Partition-wise Join**: Join data on the same partition
- **Broadcast Join**: Broadcast small tables to all nodes
- **Aggregation Pushdown**: Execute aggregations closer to data

### 3.4 MPP (Massively Parallel Processing) Mode

**MPP Task Planning**:

#### When MPP is Used:
1. **OLAP Queries**: Analytical workloads with large data scans
2. **TiFlash Storage**: Columnar storage engine available
3. **Complex Aggregations**: GROUP BY with large result sets
4. **Join-Heavy Queries**: Multiple table joins

#### MPP Task Properties:
```go
type MPPPartitionType int

const (
    AnyType MPPPartitionType = iota
    BroadcastType
    HashType
)
```

**Example MPP Plan**:
```sql
-- Query requiring large aggregation
SELECT region, COUNT(*), SUM(sales) 
FROM large_sales_table 
WHERE year = 2023 
GROUP BY region;

-- MPP Execution:
-- 1. Scan data from TiFlash in parallel
-- 2. Pre-aggregate on each node
-- 3. Final aggregation with reduced data
```

## 4. Statistics Module

### 4.1 Histogram Collection and Usage

**Implementation**: `pkg/statistics/histogram.go:62`

```go
type Histogram struct {
    Tp *types.FieldType
    
    // Histogram elements
    Bounds  *chunk.Chunk  // Bucket boundaries
    Buckets []Bucket      // Bucket statistics
    
    // Statistics metadata
    ID        int64   // Column ID
    NDV       int64   // Number of distinct values
    NullCount int64   // Number of null values
    LastUpdateVersion uint64
    
    // Correlation coefficient for physical vs logical order
    Correlation float64
}
```

#### Histogram Structure:
- **Bucket Bounds**: Min/max values in each bucket
- **Bucket Counts**: Cumulative row counts
- **Bucket Repeats**: Frequency of most common value in bucket

#### Usage in Estimation:
```go
func (h *Histogram) EqualRowCount(sctx planctx.PlanContext, value types.Datum, encodedVal []byte, realtimeRowCount int64) (result float64, err error)
```

### 4.2 Cardinality Estimation

**Implementation**: `pkg/planner/cardinality/selectivity.go:53`

```go
func Selectivity(
    ctx planctx.PlanContext,
    coll *statistics.HistColl,
    exprs []expression.Expression,
    filledPaths []*planutil.AccessPath,
) (result float64, retStatsNodes []*StatsNode, err error)
```

#### Estimation Process:

1. **Single Column Predicates**:
   - Use histogram buckets for range queries
   - Use NDV for equality queries
   - Handle null values separately

2. **Multi-Column Predicates**:
   - Independence assumption for uncorrelated columns
   - Use extended statistics for correlated columns
   - Apply selectivity correlation factors

3. **Join Cardinality**:
   - **Inner Join**: `|R ⋈ S| = |R| × |S| / max(NDV(R.key), NDV(S.key))`
   - **Semi Join**: `|R ⋉ S| = |R| × selectivity`
   - **Anti Join**: `|R ⋈̅ S| = |R| × (1 - selectivity)`

### 4.3 Statistics Feedback Mechanisms

**Feedback Collection**: `pkg/statistics/table.go:1`

```go
type Table struct {
    ExtendedStats *ExtendedStatsColl
    ColAndIdxExistenceMap *ColAndIdxExistenceMap
    HistColl
    Version uint64
    LastAnalyzeVersion uint64
}
```

#### Feedback Process:
1. **Runtime Collection**: Gather actual vs estimated cardinalities
2. **Bucket Adjustment**: Update histogram buckets based on feedback
3. **NDV Updates**: Refine distinct value estimates
4. **Selectivity Correction**: Adjust predicate selectivity estimates

#### Pseudo Statistics:
When no statistics are available, TiDB uses pseudo statistics:
- **Default Row Count**: 10,000 rows
- **Equal Selectivity**: 1/1000
- **Range Selectivity**: 1/3
- **Between Selectivity**: 1/40

## 5. Practical Examples

### 5.1 Plan Transformation Examples

#### Example 1: Simple SELECT with WHERE

**Original Query**:
```sql
SELECT name, email FROM users WHERE age > 25 AND city = 'Beijing';
```

**Transformation Steps**:

1. **AST to Logical Plan**:
```
LogicalProjection [name, email]
├── LogicalSelection [age > 25 AND city = 'Beijing']
    └── LogicalTableScan [users]
```

2. **After Column Pruning**:
```
LogicalProjection [name, email]
├── LogicalSelection [age > 25 AND city = 'Beijing']
    └── LogicalTableScan [name, email, age, city]  // Pruned other columns
```

3. **Physical Plan Options**:
```
Option 1: Table Scan + Selection
PhysicalProjection [name, email]
├── PhysicalSelection [age > 25 AND city = 'Beijing']
    └── PhysicalTableScan [users] Cost: 1000

Option 2: Index Scan (if index on city exists)
PhysicalProjection [name, email]
├── PhysicalSelection [age > 25]
    └── PhysicalIndexScan [idx_city] Cost: 100 (Selected)
```

#### Example 2: Complex Join Query

**Original Query**:
```sql
SELECT u.name, d.department_name, COUNT(*)
FROM users u
JOIN departments d ON u.dept_id = d.id
WHERE u.salary > 50000
GROUP BY u.name, d.department_name
ORDER BY COUNT(*) DESC;
```

**Optimization Process**:

1. **Predicate Pushdown**:
```sql
-- Push salary filter to users table
-- Filter: u.salary > 50000 pushed to users scan
```

2. **Join Reordering** (if more tables involved):
```sql
-- Cost-based selection of join order
-- Smaller filtered result set (users) becomes left side
```

3. **Physical Plan Selection**:
```
PhysicalSort [COUNT(*) DESC]
├── PhysicalHashAgg [u.name, d.department_name]
    └── PhysicalHashJoin [u.dept_id = d.id]
        ├── PhysicalSelection [salary > 50000]
        │   └── PhysicalTableScan [users]
        └── PhysicalTableScan [departments]
```

### 5.2 Index Selection Examples

#### Scenario: Query with Multiple Index Options

**Table Definition**:
```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    status VARCHAR(20),
    amount DECIMAL(10,2),
    INDEX idx_customer (customer_id),
    INDEX idx_date (order_date),
    INDEX idx_status (status),
    INDEX idx_date_status (order_date, status)
);
```

**Query**:
```sql
SELECT * FROM orders 
WHERE order_date = '2023-01-15' 
  AND status = 'completed';
```

**Index Selection Analysis**:

| Index | Estimated Rows | Cost Analysis |
|-------|---------------|---------------|
| `idx_date` | 1,000 rows | Scan index + 1,000 lookups |
| `idx_status` | 5,000 rows | Scan index + 5,000 lookups |
| `idx_date_status` | 50 rows | Scan index + 50 lookups (Selected) |
| Table Scan | 100,000 rows | Full table scan + filter |

**Winner**: `idx_date_status` (composite index) provides best selectivity.

## 6. Performance Tuning

### 6.1 Query Optimization Guidelines

#### Understanding EXPLAIN Output
```sql
EXPLAIN FORMAT='brief' SELECT * FROM users WHERE age > 25;
```

**Plan Reading**:
- **id**: Operator identifier
- **estRows**: Estimated row count
- **task**: Execution location (root/cop)
- **access object**: Table/index being accessed
- **operator info**: Detailed operator information

#### Common Optimization Patterns:

1. **Index Usage Verification**:
```sql
-- Check if indexes are being used
EXPLAIN SELECT * FROM users WHERE name = 'John';

-- Expected: IndexScan instead of TableScan
```

2. **Join Order Optimization**:
```sql
-- Use hints to force specific join order if needed
SELECT /*+ LEADING(small_table, large_table) */ *
FROM large_table 
JOIN small_table ON large_table.id = small_table.ref_id;
```

### 6.2 Using Hints for Plan Control

#### Available Hint Types:

**Join Method Hints**:
```sql
-- Force hash join
SELECT /*+ HASH_JOIN(t1, t2) */ * FROM t1 JOIN t2 ON t1.id = t2.id;

-- Force merge join
SELECT /*+ MERGE_JOIN(t1, t2) */ * FROM t1 JOIN t2 ON t1.id = t2.id;

-- Force index nested loop join
SELECT /*+ INL_JOIN(t1, t2) */ * FROM t1 JOIN t2 ON t1.id = t2.id;
```

**Index Hints**:
```sql
-- Force specific index usage
SELECT /*+ USE_INDEX(users, idx_age) */ * FROM users WHERE age > 25;

-- Ignore specific index
SELECT /*+ IGNORE_INDEX(users, idx_name) */ * FROM users WHERE name = 'John';
```

**Access Method Hints**:
```sql
-- Force table scan
SELECT /*+ USE_TBLSCAN(users) */ * FROM users WHERE age > 25;

-- Force index scan
SELECT /*+ USE_INDEXSCAN(users, idx_age) */ * FROM users WHERE age > 25;
```

### 6.3 Plan Analysis and Debugging

#### Performance Monitoring
```sql
-- Enable slow query log analysis
SET SESSION tidb_slow_log_threshold = 1000; -- 1 second

-- Enable optimizer trace
SET SESSION tidb_opt_write_row_id = 1;
SET SESSION tidb_optimizer_trace_level = 1;
```

#### Statistics Management
```sql
-- Update table statistics
ANALYZE TABLE users;

-- Check statistics freshness
SHOW STATS_HEALTHY;

-- View histogram information
SHOW STATS_BUCKETS WHERE table_name = 'users';
```

#### Plan Cache Management
```sql
-- Check plan cache hit ratio
SELECT * FROM INFORMATION_SCHEMA.STATEMENTS_SUMMARY 
WHERE QUERY_SAMPLE_TEXT LIKE '%your_query%';

-- Clear plan cache
ADMIN FLUSH QUERY_CACHE;
```

### 6.4 Common Performance Issues and Solutions

#### Issue 1: Poor Index Selection
**Problem**: Query uses suboptimal index or full table scan

**Diagnosis**:
```sql
EXPLAIN FORMAT='verbose' SELECT * FROM users WHERE age > 25;
-- Check if appropriate index is used
```

**Solutions**:
- Add covering indexes
- Use index hints to force specific index
- Update table statistics with `ANALYZE TABLE`

#### Issue 2: Cartesian Product Joins
**Problem**: Missing join conditions causing cartesian products

**Diagnosis**:
```sql
-- Look for missing join predicates
EXPLAIN SELECT * FROM users u, departments d;
-- Should show huge estimated row count
```

**Solutions**:
- Add proper join conditions
- Use hints to control join order
- Break complex queries into smaller parts

#### Issue 3: Suboptimal Join Order
**Problem**: Large intermediate result sets due to poor join order

**Diagnosis**:
```sql
-- Check estimated rows at each join step
EXPLAIN FORMAT='verbose' 
SELECT * FROM large_table l 
JOIN small_table s ON l.id = s.ref_id 
JOIN medium_table m ON s.id = m.ref_id;
```

**Solutions**:
- Use `LEADING()` hint to specify join order
- Ensure statistics are up-to-date
- Consider query rewriting to add selective predicates

This comprehensive documentation covers TiDB's query planning and optimization system, providing both theoretical understanding and practical guidance for effective query performance tuning.

---

## Related Documentation

- **[Architecture Analysis](architecture_analysis.md)** - Complete system architecture overview
- **[Getting Started](getting_started.md)** - Development environment setup
- **[Best Practices](best_practices.md)** - Code style and testing guidelines
- **[API Reference](api_reference.md)** - Key interfaces for extension development
- **[Troubleshooting](troubleshooting.md)** - Performance debugging and common issues

## Next Steps

1. **For Understanding Architecture**: Start with [Architecture Analysis](architecture_analysis.md)
2. **For Development Setup**: Follow [Getting Started](getting_started.md) guide
3. **For Code Quality**: Adhere to [Best Practices](best_practices.md)
4. **For Extension Development**: Reference [API Reference](api_reference.md)
5. **For Performance Issues**: Consult [Troubleshooting](troubleshooting.md)