# TiDB Query Optimizer Deep Dive

The TiDB query optimizer is responsible for transforming SQL queries into efficient execution plans. It combines rule-based optimization (RBO) with cost-based optimization (CBO) to generate optimal query execution strategies for distributed environments.

## 🏗️ Optimizer Architecture

```
SQL Query (AST)
      ↓
┌─────────────────────────────────────────────────────┐
│              Logical Plan Builder                   │
│              (pkg/planner/core/)                    │
└─────────────────────┬───────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│          Rule-Based Optimization (RBO)              │
│  ┌─────────────────────────────────────────────────┐ │
│  │ • Predicate Pushdown                            │ │
│  │ • Column Pruning                                │ │
│  │ • Join Reordering                               │ │
│  │ • Projection Elimination                        │ │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│         Cost-Based Optimization (CBO)               │
│  ┌─────────────────────────────────────────────────┐ │
│  │ • Statistics-based Cost Estimation             │ │
│  │ • Physical Plan Selection                       │ │
│  │ • Index Selection                               │ │
│  │ • Join Algorithm Selection                      │ │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────┘
                      ↓
                Physical Plan
```

## 🔧 Core Components

### 1. Logical Plan Builder (`pkg/planner/core/`)

Converts AST into logical execution plans:

```go
// Core plan building interface
type PlanBuilder struct {
    ctx       sessionctx.Context      // Session context
    is        infoschema.InfoSchema   // Schema information
    colMapper map[*ast.ColumnNameExpr]int  // Column mapping
    
    // Optimization context
    curClause       clauseCode         // Current clause being processed
    rewriterPool    []expressionRewriter
    inUpdateStmt    bool              // Inside UPDATE statement
    inDeleteStmt    bool              // Inside DELETE statement
    
    // Error tracking
    err             error             // Building errors
    hasAgg          bool              // Has aggregation
    hasWindowFunc   bool              // Has window functions
}

// Build logical plan from AST
func (b *PlanBuilder) Build(ctx context.Context, node ast.Node) (Plan, error) {
    switch x := node.(type) {
    case *ast.SelectStmt:
        return b.buildSelect(ctx, x)
    case *ast.InsertStmt:
        return b.buildInsert(ctx, x)
    case *ast.UpdateStmt:
        return b.buildUpdate(ctx, x)
    case *ast.DeleteStmt:
        return b.buildDelete(ctx, x)
    default:
        return nil, ErrUnsupportedType.GenWithStack("Unsupported type %T", node)
    }
}
```

#### SELECT Statement Processing

```go
func (b *PlanBuilder) buildSelect(ctx context.Context, sel *ast.SelectStmt) (LogicalPlan, error) {
    // 1. Build FROM clause
    p, err := b.buildResultSetNode(ctx, sel.From)
    if err != nil {
        return nil, err
    }
    
    // 2. Process WHERE clause
    if sel.Where != nil {
        p, err = b.buildSelection(ctx, p, sel.Where, nil)
        if err != nil {
            return nil, err
        }
    }
    
    // 3. Process GROUP BY clause
    if sel.GroupBy != nil {
        p, err = b.buildAggregation(ctx, p, sel.GroupBy.Items, sel.Having)
        if err != nil {
            return nil, err
        }
    }
    
    // 4. Process SELECT clause (projection)
    p, err = b.buildProjection(ctx, p, sel.Fields.Fields, nil)
    if err != nil {
        return nil, err
    }
    
    // 5. Process ORDER BY clause
    if sel.OrderBy != nil {
        p, err = b.buildSort(ctx, p, sel.OrderBy.Items, nil)
        if err != nil {
            return nil, err
        }
    }
    
    // 6. Process LIMIT clause
    if sel.Limit != nil {
        p, err = b.buildLimit(p, sel.Limit)
        if err != nil {
            return nil, err
        }
    }
    
    return p, nil
}
```

### 2. Rule-Based Optimization

TiDB applies a series of transformation rules to logical plans:

#### Optimization Rules Framework

```go
// Optimization rule interface
type logicalOptRule interface {
    optimize(context.Context, LogicalPlan) (LogicalPlan, error)
    name() string
}

// Rule registry
var optRuleList = []logicalOptRule{
    &columnPruner{},                  // Remove unused columns
    &buildKeySolver{},                // Derive key information
    &decorrelateSolver{},             // Remove correlated subqueries
    &aggregationEliminator{},         // Eliminate unnecessary aggregations
    &projectionEliminator{},          // Remove redundant projections
    &maxMinEliminator{},              // Optimize MAX/MIN functions
    &ppdSolver{},                     // Predicate pushdown
    &outerJoinEliminator{},           // Convert outer joins to inner joins
    &partitionProcessor{},            // Partition pruning
    &aggregationPushDownSolver{},     // Push down aggregations
    &pushDownTopNOptimizer{},         // Push down TOP-N operations
    &joinReOrderSolver{},             // Reorder join operations
    &columnPruner{},                  // Final column pruning
    &ppdSolver{},                     // Final predicate pushdown
}
```

#### Predicate Pushdown Implementation

```go
type ppdSolver struct{}

func (s *ppdSolver) optimize(ctx context.Context, lp LogicalPlan) (LogicalPlan, error) {
    return s.predicatePushDown(lp, nil)
}

func (s *ppdSolver) predicatePushDown(p LogicalPlan, predicates []expression.Expression) (LogicalPlan, []expression.Expression) {
    switch x := p.(type) {
    case *LogicalTableScan:
        // Push predicates to table scan
        x.AccessConds = append(x.AccessConds, predicates...)
        return x, nil
        
    case *LogicalJoin:
        // Analyze join conditions for pushdown opportunities
        leftCond, rightCond, otherCond := s.splitJoinPredicates(x, predicates)
        
        // Recursively push down to children
        x.children[0], leftCond = s.predicatePushDown(x.children[0], leftCond)
        x.children[1], rightCond = s.predicatePushDown(x.children[1], rightCond)
        
        // Keep non-pushable conditions at join level
        x.OtherConditions = append(x.OtherConditions, otherCond...)
        return x, leftCond
        
    case *LogicalProjection:
        // Transform predicates for projection pushdown
        canBePushed, cannotBePushed := s.canPredicateBePushed(x, predicates)
        x.children[0], canBePushed = s.predicatePushDown(x.children[0], canBePushed)
        return x, cannotBePushed
        
    default:
        // Default: cannot push down
        return p, predicates
    }
}
```

#### Column Pruning

```go
type columnPruner struct{}

func (s *columnPruner) optimize(ctx context.Context, lp LogicalPlan) (LogicalPlan, error) {
    return s.pruneColumns(lp, lp.Schema().Columns)
}

func (s *columnPruner) pruneColumns(p LogicalPlan, required []*expression.Column) (LogicalPlan, error) {
    switch x := p.(type) {
    case *LogicalProjection:
        // Determine which columns from children are actually needed
        childReq := s.getChildReqCols(x, required)
        
        // Recursively prune children
        var err error
        x.children[0], err = s.pruneColumns(x.children[0], childReq)
        if err != nil {
            return nil, err
        }
        
        // Remove unused expressions from projection
        x.Exprs = s.pruneProjectionExprs(x.Exprs, required)
        return x, nil
        
    case *LogicalTableScan:
        // Prune columns at table scan level
        x.Columns = s.pruneTableColumns(x.Columns, required)
        return x, nil
        
    default:
        // Recursively prune children
        for i, child := range p.Children() {
            newChild, err := s.pruneColumns(child, required)
            if err != nil {
                return nil, err
            }
            p.SetChild(i, newChild)
        }
        return p, nil
    }
}
```

### 3. Cost-Based Optimization

After rule-based optimization, TiDB uses cost-based optimization for physical plan selection:

#### Statistics Integration

```go
// Table statistics for cost estimation
type Table struct {
    HistColl    *HistColl              // Histogram collection
    Version     uint64                 // Statistics version
    TblInfoID   int64                  // Table info ID
    Count       int64                  // Row count
    ModifyCount int64                  // Modified row count
    
    // Column statistics
    Columns     map[int64]*Column      // Column ID -> Statistics
    Indices     map[int64]*Index       // Index ID -> Statistics
    
    // Pseudo statistics flag
    Pseudo      bool                   // Whether using pseudo statistics
}

// Cost estimation using statistics
func (coll *HistColl) GetRowCountByColumnRanges(colID int64, ranges []*ranger.Range) (float64, error) {
    c, ok := coll.Columns[colID]
    if !ok || c.IsInvalid() {
        // Use pseudo statistics
        return pseudoRowCount(ranges), nil
    }
    
    // Use histogram for accurate estimation
    rowCount := 0.0
    for _, ran := range ranges {
        count, err := c.GetColumnRowCount(coll.Count, ran)
        if err != nil {
            return 0, err
        }
        rowCount += count
    }
    
    return rowCount, nil
}
```

#### Physical Plan Enumeration

```go
// Generate physical plans for logical plan
func (p *LogicalJoin) exhaustPhysicalPlans(prop *property.PhysicalProperty) ([]PhysicalPlan, bool) {
    var physicalPlans []PhysicalPlan
    
    // 1. Hash Join
    hashJoin := p.generateHashJoin(prop)
    if hashJoin != nil {
        physicalPlans = append(physicalPlans, hashJoin)
    }
    
    // 2. Merge Join (if ordered)
    if p.canUseMergeJoin(prop) {
        mergeJoin := p.generateMergeJoin(prop)
        if mergeJoin != nil {
            physicalPlans = append(physicalPlans, mergeJoin)
        }
    }
    
    // 3. Index Join (if applicable)
    if p.canUseIndexJoin() {
        indexJoin := p.generateIndexJoin(prop)
        if indexJoin != nil {
            physicalPlans = append(physicalPlans, indexJoin)
        }
    }
    
    return physicalPlans, true
}
```

#### Cost Calculation

```go
// Cost estimation for physical plans
type costVer2 struct {
    plan PhysicalPlan
}

func (c *costVer2) GetPlanCost(option *PlanCostOption) (float64, error) {
    switch x := c.plan.(type) {
    case *PhysicalTableScan:
        return c.GetTableScanCost(x, option)
    case *PhysicalIndexScan:
        return c.GetIndexScanCost(x, option)
    case *PhysicalHashJoin:
        return c.GetHashJoinCost(x, option)
    case *PhysicalMergeJoin:
        return c.GetMergeJoinCost(x, option)
    default:
        return c.GetDefaultCost(x, option)
    }
}

func (c *costVer2) GetTableScanCost(ts *PhysicalTableScan, option *PlanCostOption) (float64, error) {
    // Cost factors
    rows := getCardinality(ts, option.CostFlag)
    scanFactor := getTableScanFactor(ts.Table)
    
    // Network cost for distributed scan
    netCost := rows * scanFactor * option.GetNetworkFactor()
    
    // Seek cost for random access
    seekCost := getTableScanSeekCost(ts) * option.GetSeekFactor()
    
    // CPU cost for result processing
    cpuCost := rows * option.GetCPUFactor()
    
    return netCost + seekCost + cpuCost, nil
}
```

### 4. Advanced Optimization Techniques

#### Cascades Framework

TiDB implements the Cascades optimization framework for advanced optimization:

```go
// Cascades optimizer implementation
type Optimizer struct {
    // Core components
    memo         *Memo              // Memo structure for plan storage
    rules        []Rule             // Transformation rules
    
    // Optimization context
    ctx          OptimizeContext    // Optimization context
    costModel    CostModel          // Cost estimation model
    
    // Control parameters
    maxIterations int               // Maximum optimization iterations
    timeLimit    time.Duration      // Time limit for optimization
}

// Memo structure for efficient plan storage
type Memo struct {
    groups       []*Group           // Expression groups
    groupExprs   []*GroupExpr       // Group expressions
    
    // Hash tables for deduplication
    groupMap     map[string]*Group  // Hash -> Group
    exprMap      map[string]*GroupExpr // Hash -> Expression
}

// Optimization using Cascades framework
func (opt *Optimizer) Optimize(ctx context.Context, logical LogicalPlan) (PhysicalPlan, float64, error) {
    // 1. Build initial memo structure
    rootGroup := opt.memo.CopyIn(logical)
    
    // 2. Apply transformation rules
    for i := 0; i < opt.maxIterations; i++ {
        if opt.applyRules(rootGroup) == 0 {
            break // No more transformations possible
        }
    }
    
    // 3. Find optimal physical plan
    bestPlan, cost := opt.findBestPlan(rootGroup, &property.PhysicalProperty{})
    
    return bestPlan, cost, nil
}
```

#### Join Reordering

```go
type joinReOrderSolver struct{}

func (s *joinReOrderSolver) optimize(ctx context.Context, p LogicalPlan) (LogicalPlan, error) {
    return s.optimizeRecursively(ctx, p)
}

func (s *joinReOrderSolver) optimizeRecursively(ctx context.Context, p LogicalPlan) (LogicalPlan, error) {
    // Extract join group for reordering
    joinGroup := s.extractJoinGroup(p)
    if len(joinGroup) <= 1 {
        return p, nil // Nothing to reorder
    }
    
    // Dynamic programming for optimal join order
    bestPlan, err := s.dpReorder(ctx, joinGroup)
    if err != nil {
        return p, err
    }
    
    return bestPlan, nil
}

func (s *joinReOrderSolver) dpReorder(ctx context.Context, joinGroup []*joinGroupEqEdge) (LogicalPlan, error) {
    n := len(joinGroup)
    
    // dp[mask] represents the best plan for subset mask
    dp := make(map[uint]*jrNode, 1<<n)
    
    // Initialize with single tables
    for i := 0; i < n; i++ {
        mask := uint(1 << i)
        dp[mask] = &jrNode{
            p:    joinGroup[i].edge[0],
            cost: s.getBaseCost(joinGroup[i].edge[0]),
        }
    }
    
    // Fill DP table
    for size := 2; size <= n; size++ {
        s.enumSubset(n, size, func(mask uint) {
            s.fillDP(mask, dp, joinGroup)
        })
    }
    
    // Return optimal plan
    fullMask := uint((1 << n) - 1)
    return dp[fullMask].p, nil
}
```

## 📊 Statistics and Cardinality Estimation

### Histogram-Based Statistics

```go
// Histogram for cardinality estimation
type Histogram struct {
    ID        int64              // Column/Index ID
    NDV       int64              // Number of distinct values
    NullCount int64              // Number of NULL values
    
    // Histogram buckets
    Buckets   []Bucket           // Histogram buckets
    Bounds    []types.Datum      // Bucket boundaries
    
    // Additional statistics
    LastUpdateVersion uint64     // Last update version
    Correlation      float64     // Correlation with rowid
}

// Estimate row count for range
func (h *Histogram) BetweenRowCount(a, b types.Datum) (float64, error) {
    if h.Len() == 0 {
        return float64(h.TotalRowCount()) / pseudoFactor, nil
    }
    
    lessCountA, err := h.LessRowCount(a)
    if err != nil {
        return 0, err
    }
    
    lessEqualCountB, err := h.LessRowCount(b)
    if err != nil {
        return 0, err
    }
    
    return lessEqualCountB - lessCountA, nil
}
```

### Automatic Statistics Collection

```go
// Auto analyze for maintaining statistics
type AutoAnalyze struct {
    ctx          sessionctx.Context
    workers      []*worker         // Analysis workers
    ratioWorker  *ratioWorker      // Ratio-based triggering
    
    // Configuration
    start        time.Time         // Start time
    end          time.Time         // End time
    maxTime      time.Duration     // Max analysis time
}

func (aa *AutoAnalyze) execAutoAnalyze(ctx context.Context) {
    // 1. Identify tables needing analysis
    candidates := aa.getCandidates(ctx)
    
    // 2. Sort by priority (modified ratio, size, etc.)
    sort.Slice(candidates, func(i, j int) bool {
        return candidates[i].changeRatio > candidates[j].changeRatio
    })
    
    // 3. Analyze high-priority tables
    for _, candidate := range candidates {
        if aa.shouldStop() {
            break
        }
        
        err := aa.analyzeTable(ctx, candidate)
        if err != nil {
            logutil.BgLogger().Error("auto analyze failed", zap.Error(err))
        }
    }
}
```

## 🔧 Index Selection and Access Path Optimization

### Access Path Generation

```go
// Generate possible access paths for table
func (ds *DataSource) generateAccessPaths() []*util.AccessPath {
    var accessPaths []*util.AccessPath
    
    // 1. Table scan path
    tablePath := &util.AccessPath{
        IsTablePath:     true,
        StoreType:       kv.TiKV,
        CountAfterAccess: float64(ds.statisticTable.Count),
    }
    accessPaths = append(accessPaths, tablePath)
    
    // 2. Index scan paths
    for _, index := range ds.possibleAccessPaths {
        if index.IsTablePath {
            continue
        }
        
        indexPath := &util.AccessPath{
            Index:           index.Index,
            FullIdxCols:     index.FullIdxCols,
            FullIdxColLens:  index.FullIdxColLens,
            IdxCols:         index.IdxCols,
            IdxColLens:      index.IdxColLens,
            CountAfterAccess: index.CountAfterAccess,
        }
        accessPaths = append(accessPaths, indexPath)
    }
    
    return accessPaths
}
```

### Index Selection Algorithm

```go
func (ds *DataSource) skylinePruning(paths []*util.AccessPath) []*util.AccessPath {
    // Skyline pruning to eliminate dominated paths
    var result []*util.AccessPath
    
    for i, path1 := range paths {
        dominated := false
        
        for j, path2 := range paths {
            if i == j {
                continue
            }
            
            // Check if path1 is dominated by path2
            if ds.isDominated(path1, path2) {
                dominated = true
                break
            }
        }
        
        if !dominated {
            result = append(result, path1)
        }
    }
    
    return result
}

func (ds *DataSource) isDominated(path1, path2 *util.AccessPath) bool {
    // Path2 dominates path1 if:
    // 1. Path2 covers more columns or same columns
    // 2. Path2 has better access conditions
    // 3. Path2 has lower estimated cost
    
    if len(path2.AccessConds) < len(path1.AccessConds) {
        return false
    }
    
    if path2.CountAfterAccess > path1.CountAfterAccess {
        return false
    }
    
    // Compare index column coverage
    if path1.Index != nil && path2.Index != nil {
        return len(path2.IdxCols) >= len(path1.IdxCols)
    }
    
    return true
}
```

## 🚀 Performance Optimizations

### Plan Cache

```go
// Plan cache for prepared statements
type PlanCache struct {
    mu       sync.RWMutex
    cache    map[string]*PlanCacheValue  // SQL -> Cached Plan
    lruList  *list.List                  // LRU eviction list
    
    // Configuration
    capacity int                         // Max cache size
    enabled  bool                        // Cache enabled flag
}

func (c *PlanCache) Get(key string, paramTypes []*types.FieldType) (*PlanCacheValue, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    if value, exists := c.cache[key]; exists {
        // Check parameter type compatibility
        if c.isTypeCompatible(value.OutPutNames, paramTypes) {
            // Move to front for LRU
            c.lruList.MoveToFront(value.element)
            return value, true
        }
    }
    
    return nil, false
}
```

### Parallel Planning

```go
// Parallel optimization for complex queries
type ParallelOptimizer struct {
    workers    int                      // Number of worker goroutines
    taskCh     chan *OptimizeTask       // Task channel
    resultCh   chan *OptimizeResult     // Result channel
    
    // Worker pool
    workerPool sync.Pool
}

func (p *ParallelOptimizer) OptimizeInParallel(plans []LogicalPlan) ([]PhysicalPlan, error) {
    results := make([]*OptimizeResult, len(plans))
    
    // Submit optimization tasks
    for i, plan := range plans {
        task := &OptimizeTask{
            ID:   i,
            Plan: plan,
        }
        p.taskCh <- task
    }
    
    // Collect results
    for i := 0; i < len(plans); i++ {
        result := <-p.resultCh
        results[result.ID] = result
    }
    
    // Extract physical plans
    physicalPlans := make([]PhysicalPlan, len(plans))
    for i, result := range results {
        if result.Error != nil {
            return nil, result.Error
        }
        physicalPlans[i] = result.PhysicalPlan
    }
    
    return physicalPlans, nil
}
```

## 🔧 Configuration and Tuning

### Optimizer Hints

TiDB supports various optimizer hints for query tuning:

```sql
-- Index hints
SELECT /*+ USE_INDEX(t, idx_name) */ * FROM t WHERE c1 = 1;
SELECT /*+ IGNORE_INDEX(t, idx_name) */ * FROM t WHERE c1 = 1;

-- Join hints  
SELECT /*+ HASH_JOIN(t1, t2) */ * FROM t1 JOIN t2 ON t1.id = t2.id;
SELECT /*+ MERGE_JOIN(t1, t2) */ * FROM t1 JOIN t2 ON t1.id = t2.id;

-- Storage engine hints
SELECT /*+ READ_FROM_storage(tikv[t1]) */ * FROM t1;
SELECT /*+ read_from_storage(tiflash[t1]) */ * FROM t1;

-- Aggregation hints
SELECT /*+ HASH_AGG() */ COUNT(*) FROM t GROUP BY c1;
SELECT /*+ STREAM_AGG() */ COUNT(*) FROM t GROUP BY c1;
```

### System Variables

Key system variables for optimizer control:

```sql
-- Statistics settings
SET SESSION tidb_auto_analyze_ratio = 0.5;
SET SESSION tidb_enable_extended_stats = ON;

-- Cost model settings
SET SESSION tidb_cost_model_version = 2;
SET SESSION tidb_opt_scan_factor = 1.5;
SET SESSION tidb_opt_desc_factor = 3.0;
SET SESSION tidb_opt_seek_factor = 20.0;

-- Join optimization
SET SESSION tidb_opt_join_reorder_threshold = 0;
SET SESSION tidb_enable_outer_join_reorder = ON;

-- Plan cache settings
SET SESSION tidb_enable_prepared_plan_cache = ON;
SET SESSION tidb_prepared_plan_cache_size = 100;
```

## 📈 Monitoring and Debugging

### Query Plan Analysis

```sql
-- Explain query execution plan
EXPLAIN SELECT * FROM t1 JOIN t2 ON t1.id = t2.id WHERE t1.status = 'active';

-- Analyze query with cost information
EXPLAIN ANALYZE SELECT * FROM t1 WHERE t1.created_at > '2023-01-01';

-- Detailed execution statistics
EXPLAIN FORMAT='verbose' SELECT COUNT(*) FROM t GROUP BY category;

-- Plan cache hit ratio
SHOW STATUS LIKE 'Com_stmt%';
```

### Optimizer Traces

```sql
-- Enable optimizer trace
SET SESSION tidb_evolve_plan_baselines = ON;
SET SESSION tidb_enable_stmt_summary = ON;

-- View query summary
SELECT * FROM information_schema.statements_summary 
WHERE digest_text LIKE '%your_query%';

-- Check plan binding
SHOW BINDINGS;
```

## 🚀 Best Practices

### 1. Statistics Management

- **Regular analysis**: Run `ANALYZE TABLE` regularly for accurate statistics
- **Monitor statistics health**: Check `SHOW STATS_HEALTHY` output
- **Use extended statistics**: Enable for correlated columns
- **Histogram maintenance**: Configure appropriate bucket counts

### 2. Index Design

- **Covering indexes**: Include all needed columns to avoid table lookups
- **Composite index order**: Place high-selectivity columns first
- **Avoid redundant indexes**: Remove overlapping index definitions
- **Monitor index usage**: Use `INFORMATION_SCHEMA.TIDB_INDEXES` 

### 3. Query Optimization

- **Use EXPLAIN**: Always analyze query execution plans
- **Leverage hints**: Guide optimizer when needed
- **Avoid function calls on columns**: Prevent index usage inhibition
- **Consider join order**: Smaller tables first in manual reordering

### 4. Performance Monitoring

- **Track slow queries**: Monitor `INFORMATION_SCHEMA.SLOW_QUERY`
- **Analyze plan cache hit ratio**: Optimize prepared statement usage
- **Monitor memory usage**: Set appropriate memory limits
- **Check region distribution**: Ensure balanced data distribution

## 📚 Related Documentation

- [SQL Layer Deep Dive](sql-layer.md)
- [Storage Engine Integration](storage-engine.md) 
- [Distributed Execution System](distributed-execution.md)
- [Performance Guidelines](../patterns/performance.md)