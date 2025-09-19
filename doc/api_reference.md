# TiDB API Reference and Extension Guide

This document provides a comprehensive reference for TiDB's key interfaces, extension points, and integration guidelines for developers building on or extending TiDB.

## Table of Contents

1. [Core Interfaces](#1-core-interfaces)
2. [Extension Points](#2-extension-points)
3. [Plugin Development](#3-plugin-development)
4. [Integration Guidelines](#4-integration-guidelines)
5. [Storage Interface](#5-storage-interface)
6. [Expression Framework](#6-expression-framework)
7. [Statistics and Cost Model](#7-statistics-and-cost-model)
8. [Custom Functions](#8-custom-functions)

## 1. Core Interfaces

### 1.1 Plan Interface Hierarchy

TiDB's planning system is built around a hierarchy of interfaces that separate logical and physical planning concerns.

#### Base Plan Interface
```go
// Location: pkg/planner/core/base/plan.go
type Plan interface {
    // ID returns the unique identifier for this plan node
    ID() int
    
    // TP returns the plan type (logical or physical)
    TP() string
    
    // ExplainID returns the ID used in EXPLAIN output
    ExplainID() fmt.Stringer
    
    // Stats returns the statistics for this plan node
    Stats() *property.StatsInfo
    
    // Schema returns the output schema of this plan
    Schema() *expression.Schema
    
    // OutputNames returns the output column names
    OutputNames() types.NameSlice
    
    // SetOutputNames sets the output column names
    SetOutputNames(names types.NameSlice)
    
    // Context returns the plan context
    SCtx() PlanContext
}
```

#### Logical Plan Interface
```go
// Location: pkg/planner/core/base/logical_plan.go
type LogicalPlan interface {
    Plan
    
    // Children returns child logical plans
    Children() []LogicalPlan
    
    // SetChild sets the child at the specified index
    SetChild(i int, child LogicalPlan)
    
    // SetChildren sets all children
    SetChildren(...LogicalPlan)
    
    // MaxOneRow returns true if this plan produces at most one row
    MaxOneRow() bool
    
    // Get possible property from children
    GetBaseLogicalPlan() *BaseLogicalPlan
    
    // Logical optimization methods
    PredicatePushDown([]expression.Expression, *optimizetrace.LogicalOptimizeOp) ([]expression.Expression, LogicalPlan, error)
    PruneColumns([]*expression.Column, *optimizetrace.LogicalOptimizeOp) (LogicalPlan, error)
    FindBestTask(property *property.PhysicalProperty, planCounter *PlanCounterTp, opt *optimizetrace.PhysicalOptimizeOp) (Task, int64, error)
}
```

#### Physical Plan Interface
```go
// Location: pkg/planner/core/base/physical_plan.go
type PhysicalPlan interface {
    Plan
    
    // Children returns child physical plans
    Children() []PhysicalPlan
    
    // SetChild sets the child at the specified index
    SetChild(i int, child PhysicalPlan)
    
    // SetChildren sets all children
    SetChildren(...PhysicalPlan)
    
    // Cost estimation
    GetPlanCostVer1(taskType property.TaskType, option *optimizetrace.PlanCostOption) (float64, error)
    GetPlanCostVer2(taskType property.TaskType, option *optimizetrace.PlanCostOption, ...bool) (costusage.CostVer2, error)
    
    // Memory usage estimation
    MemoryUsage() (int64, int64)
    
    // Clone creates a deep copy of the plan
    Clone() (PhysicalPlan, error)
    
    // Execution properties
    GetChildReqProps(idx int) *property.PhysicalProperty
}
```

### 1.2 Execution Interface

#### Executor Interface
```go
// Location: pkg/executor/internal/exec/executor.go
type Executor interface {
    // Base returns the base executor
    Base() *BaseExecutor
    
    // Open initializes the executor
    Open(ctx context.Context) error
    
    // Next returns the next chunk of data
    Next(ctx context.Context, req *chunk.Chunk) error
    
    // Close releases resources
    Close() error
    
    // Schema returns the output schema
    Schema() *expression.Schema
}

// BaseExecutor provides common functionality
type BaseExecutor struct {
    ctx          sessionctx.Context
    id           int
    schema       *expression.Schema
    children     []Executor
    retFieldTypes []*types.FieldType
    runtimeStats *execdetails.RuntimeStats
}
```

#### Iterator Pattern for Execution
```go
// Example: Custom executor implementation
type CustomExecutor struct {
    BaseExecutor
    
    // Custom fields
    dataSource DataSource
    filter     expression.Expression
}

func (e *CustomExecutor) Open(ctx context.Context) error {
    if err := e.BaseExecutor.Open(ctx); err != nil {
        return err
    }
    
    return e.dataSource.Open(ctx)
}

func (e *CustomExecutor) Next(ctx context.Context, req *chunk.Chunk) error {
    req.Reset()
    
    for !req.IsFull() {
        row, err := e.dataSource.NextRow(ctx)
        if err != nil {
            return err
        }
        if row == nil {
            break // No more data
        }
        
        // Apply filter if present
        if e.filter != nil {
            match, err := e.filter.EvalBool(ctx, row)
            if err != nil {
                return err
            }
            if !match {
                continue
            }
        }
        
        req.AppendRow(row)
    }
    
    return nil
}

func (e *CustomExecutor) Close() error {
    defer e.BaseExecutor.Close()
    return e.dataSource.Close()
}
```

### 1.3 Storage Interface

#### KV Storage Interface
```go
// Location: pkg/kv/kv.go
type Storage interface {
    // Begin creates a new transaction
    Begin() (Transaction, error)
    BeginWithStartTS(startTS uint64) (Transaction, error)
    
    // GetSnapshot gets a snapshot for read operations
    GetSnapshot(ts uint64) Snapshot
    
    // Close closes the storage
    Close() error
    
    // UUID returns the unique ID of this storage
    UUID() string
    
    // CurrentVersion returns current version
    CurrentVersion(txnScope string) (kv.Version, error)
    
    // GetClient returns the client instance
    GetClient() Client
    
    // GetMPPClient returns the MPP client for analytical queries
    GetMPPClient() MPPClient
}

// Transaction interface for ACID operations
type Transaction interface {
    Retriever
    Mutator
    
    // Commit commits the transaction
    Commit(ctx context.Context) error
    
    // Rollback rolls back the transaction
    Rollback() error
    
    // Size returns the size of the transaction
    Size() int
    
    // Len returns the number of entries in the transaction
    Len() int
    
    // GetMemBuffer returns the memory buffer
    GetMemBuffer() MemBuffer
    
    // GetSnapshot returns the snapshot
    GetSnapshot() Snapshot
    
    // SetOption sets transaction options
    SetOption(opt int, val interface{})
    
    // DelOption deletes a transaction option
    DelOption(opt int)
    
    // IsReadOnly returns true if the transaction is read-only
    IsReadOnly() bool
    
    // StartTS returns the start timestamp
    StartTS() uint64
    
    // Valid returns true if the transaction is valid
    Valid() bool
    
    // GetClusterID returns the cluster ID
    GetClusterID() uint64
}
```

## 2. Extension Points

### 2.1 Custom Logical Operators

#### Creating Custom Logical Plans
```go
// Step 1: Define your logical operator
type LogicalCustomOp struct {
    logicalop.BaseLogicalPlan
    
    // Custom fields
    customParam string
    filterExpr  expression.Expression
}

// Step 2: Implement required methods
func (p *LogicalCustomOp) Init(ctx base.PlanContext, offset int) *LogicalCustomOp {
    p.BaseLogicalPlan = logicalop.NewBaseLogicalPlan(ctx, plancodec.TypeCustom, p, offset)
    return p
}

func (p *LogicalCustomOp) TP() string {
    return "LogicalCustomOp"
}

// Step 3: Implement optimization interfaces
func (p *LogicalCustomOp) PredicatePushDown(predicates []expression.Expression, opt *optimizetrace.LogicalOptimizeOp) ([]expression.Expression, base.LogicalPlan, error) {
    // Implement predicate pushdown logic
    canPush, cannotPush := p.splitPredicates(predicates)
    
    // Push applicable predicates to children
    child := p.Children()[0]
    _, newChild, err := child.PredicatePushDown(canPush, opt)
    if err != nil {
        return nil, nil, err
    }
    
    p.SetChild(0, newChild)
    return cannotPush, p, nil
}

func (p *LogicalCustomOp) PruneColumns(parentUsedCols []*expression.Column, opt *optimizetrace.LogicalOptimizeOp) (base.LogicalPlan, error) {
    // Implement column pruning logic
    usedCols := p.extractUsedColumns(parentUsedCols)
    
    child := p.Children()[0]
    newChild, err := child.PruneColumns(usedCols, opt)
    if err != nil {
        return nil, err
    }
    
    p.SetChild(0, newChild)
    return p, nil
}

// Step 4: Implement cost estimation
func (p *LogicalCustomOp) FindBestTask(prop *property.PhysicalProperty, planCounter *base.PlanCounterTp, opt *optimizetrace.PhysicalOptimizeOp) (base.Task, int64, error) {
    // Generate physical implementations
    physicalPlans := p.exhaustPhysicalPlans(prop)
    
    var bestTask base.Task
    var bestCost float64 = math.MaxFloat64
    
    for _, physicalPlan := range physicalPlans {
        task, cost, err := physicalPlan.findBestTask(prop, planCounter, opt)
        if err != nil {
            return nil, 0, err
        }
        
        if cost < bestCost {
            bestCost = cost
            bestTask = task
        }
    }
    
    return bestTask, int64(bestCost), nil
}
```

### 2.2 Custom Physical Operators

#### Implementing Physical Plans
```go
// Step 1: Define physical operator
type PhysicalCustomOp struct {
    physicalop.BasePhysicalPlan
    
    // Custom fields
    customParam string
    algorithm   CustomAlgorithmType
}

// Step 2: Implement execution generation
func (p *PhysicalCustomOp) GetExecutor(ctx context.Context, sctx sessionctx.Context) (exec.Executor, error) {
    // Create child executors
    children := make([]exec.Executor, len(p.Children()))
    for i, child := range p.Children() {
        childExec, err := child.GetExecutor(ctx, sctx)
        if err != nil {
            return nil, err
        }
        children[i] = childExec
    }
    
    // Create custom executor
    executor := &CustomExecutor{
        BaseExecutor: exec.NewBaseExecutor(sctx, p.Schema(), p.ID(), children...),
        customParam:  p.customParam,
        algorithm:    p.algorithm,
    }
    
    return executor, nil
}

// Step 3: Implement cost calculation
func (p *PhysicalCustomOp) GetPlanCostVer2(taskType property.TaskType, option *optimizetrace.PlanCostOption, ...bool) (costusage.CostVer2, error) {
    if p.planCostInit {
        return p.planCostVer2, nil
    }
    
    // Calculate child costs
    childCost := costusage.ZeroCostVer2
    for _, child := range p.Children() {
        cost, err := child.GetPlanCostVer2(taskType, option)
        if err != nil {
            return costusage.ZeroCostVer2, err
        }
        childCost = costusage.SumCostVer2(childCost, cost)
    }
    
    // Add operator-specific costs
    inputRows := getCardinality(p.Children()[0], option.CostFlag)
    cpuCost := costusage.NewCostVer2(option.CostFlag, 
        inputRows * p.getCustomOpCPUFactor(),  // CPU cost
        0,                                      // Memory cost
        0,                                      // Disk cost
        0,                                      // Network cost
    )
    
    p.planCostVer2 = costusage.SumCostVer2(childCost, cpuCost)
    p.planCostInit = true
    
    return p.planCostVer2, nil
}
```

### 2.3 Custom Expression Functions

#### Registering Custom Functions
```go
// Step 1: Define function signature
type customFunctionSig struct {
    baseBuiltinFunc
    // Custom fields
}

// Step 2: Implement evaluation
func (c *customFunctionSig) evalString(row chunk.Row) (string, bool, error) {
    // Get arguments
    arg1, isNull1, err := c.args[0].EvalString(c.ctx, row)
    if isNull1 || err != nil {
        return "", isNull1, err
    }
    
    arg2, isNull2, err := c.args[1].EvalString(c.ctx, row)
    if isNull2 || err != nil {
        return "", isNull2, err
    }
    
    // Implement custom logic
    result := customStringOperation(arg1, arg2)
    return result, false, nil
}

// Step 3: Register the function
func init() {
    expression.RegisterFunction("CUSTOM_FUNC", &functionClass{
        fname: "CUSTOM_FUNC",
        minArgs: 2,
        maxArgs: 2,
    })
}

type customFunctionClass struct {
    baseFunctionClass
}

func (c *customFunctionClass) getFunction(ctx sessionctx.Context, args []expression.Expression) (builtinFunc, error) {
    // Validate arguments
    if err := c.verifyArgs(args); err != nil {
        return nil, err
    }
    
    // Create function signature
    sig := &customFunctionSig{
        baseBuiltinFunc: newBaseBuiltinFunc(ctx, c.funcName, args, types.ETString),
    }
    
    sig.tp.Flen = mysql.MaxBlobWidth
    return sig, nil
}
```

## 3. Plugin Development

### 3.1 Plugin Architecture

#### Plugin Interface
```go
// Location: pkg/plugin/plugin.go
type Plugin interface {
    // OnInit is called when the plugin is loaded
    OnInit(ctx context.Context, manifest *Manifest) error
    
    // OnShutdown is called when the plugin is unloaded
    OnShutdown(ctx context.Context) error
    
    // OnFlush is called periodically
    OnFlush(ctx context.Context, event FlushEvent) error
    
    // Validate validates the plugin configuration
    Validate(ctx context.Context, manifest *Manifest) error
}

// Manifest describes plugin metadata
type Manifest struct {
    Name           string                 `json:"name"`
    Description    string                 `json:"description"`
    Version        string                 `json:"version"`
    RequireVersion []string               `json:"requireVersion"`
    License        string                 `json:"license"`
    Author         string                 `json:"author"`
    AuthorEmail    string                 `json:"authorEmail"`
    URL            string                 `json:"url"`
    Kind           Kind                   `json:"kind"`
    SysVars        map[string]*SysVar     `json:"sysVars"`
    Validate       func(ctx context.Context, m *Manifest) error `json:"-"`
}
```

#### Audit Plugin Example
```go
// Step 1: Implement the plugin interface
type AuditPlugin struct {
    cfg    *AuditConfig
    logger *log.Logger
}

func (p *AuditPlugin) OnInit(ctx context.Context, manifest *Manifest) error {
    // Initialize plugin
    p.cfg = &AuditConfig{}
    if err := json.Unmarshal(manifest.Config, p.cfg); err != nil {
        return err
    }
    
    // Set up logging
    p.logger = log.New(os.Stdout, "[AUDIT] ", log.LstdFlags)
    return nil
}

func (p *AuditPlugin) OnShutdown(ctx context.Context) error {
    // Cleanup resources
    return nil
}

func (p *AuditPlugin) OnFlush(ctx context.Context, event FlushEvent) error {
    switch event.Type {
    case FlushTypeQuery:
        return p.auditQuery(event)
    case FlushTypeConnection:
        return p.auditConnection(event)
    default:
        return nil
    }
}

// Step 2: Implement audit logic
func (p *AuditPlugin) auditQuery(event FlushEvent) error {
    queryInfo := event.Data.(*QueryInfo)
    
    auditRecord := AuditRecord{
        Timestamp: time.Now(),
        User:      queryInfo.User,
        Database:  queryInfo.Database,
        Query:     queryInfo.SQL,
        Duration:  queryInfo.Duration,
        Success:   queryInfo.Error == nil,
    }
    
    if queryInfo.Error != nil {
        auditRecord.Error = queryInfo.Error.Error()
    }
    
    return p.writeAuditRecord(auditRecord)
}

// Step 3: Register the plugin
func init() {
    plugin.RegisterPlugin("audit", &AuditPlugin{})
}
```

### 3.2 Extension Framework

#### Custom Extension Points
```go
// Location: pkg/extension/extension.go
type Extension interface {
    // Name returns the extension name
    Name() string
    
    // Bootstrap initializes the extension
    Bootstrap(ctx context.Context) error
    
    // Shutdown cleans up the extension
    Shutdown() error
    
    // IsEnabled returns true if the extension is enabled
    IsEnabled() bool
}

// SessionExtension provides session-level extensions
type SessionExtension interface {
    Extension
    
    // OnConnectionEstablished is called when a new connection is established
    OnConnectionEstablished(ctx context.Context, info ConnectionInfo) error
    
    // OnStatementStart is called before statement execution
    OnStatementStart(ctx context.Context, node ast.StmtNode) error
    
    // OnStatementFinish is called after statement execution
    OnStatementFinish(ctx context.Context, node ast.StmtNode, result ExecResult) error
    
    // OnConnectionClosed is called when a connection is closed
    OnConnectionClosed(ctx context.Context, info ConnectionInfo) error
}
```

## 4. Integration Guidelines

### 4.1 Database Driver Integration

#### Custom Driver Implementation
```go
// Implement the driver.Driver interface
type CustomDriver struct {
    config DriverConfig
}

func (d *CustomDriver) Open(name string) (driver.Conn, error) {
    // Parse connection string
    cfg, err := ParseDSN(name)
    if err != nil {
        return nil, err
    }
    
    // Establish connection
    conn, err := d.connect(cfg)
    if err != nil {
        return nil, err
    }
    
    return &CustomConn{
        driver: d,
        conn:   conn,
        cfg:    cfg,
    }, nil
}

// Implement driver.Conn interface
type CustomConn struct {
    driver *CustomDriver
    conn   net.Conn
    cfg    *Config
}

func (c *CustomConn) Prepare(query string) (driver.Stmt, error) {
    // Implement prepared statement
    return &CustomStmt{
        conn:  c,
        query: query,
    }, nil
}

func (c *CustomConn) Begin() (driver.Tx, error) {
    // Begin transaction
    return &CustomTx{conn: c}, nil
}

func (c *CustomConn) Close() error {
    return c.conn.Close()
}
```

### 4.2 Monitoring Integration

#### Metrics Collection
```go
// Custom metrics collector
type MetricsCollector struct {
    registry *prometheus.Registry
    
    // Query metrics
    queryDuration *prometheus.HistogramVec
    queryCount    *prometheus.CounterVec
    
    // Connection metrics
    activeConnections prometheus.Gauge
    totalConnections  prometheus.Counter
}

func NewMetricsCollector() *MetricsCollector {
    collector := &MetricsCollector{
        registry: prometheus.NewRegistry(),
        
        queryDuration: prometheus.NewHistogramVec(
            prometheus.HistogramOpts{
                Name: "tidb_query_duration_seconds",
                Help: "Duration of SQL queries",
                Buckets: prometheus.ExponentialBuckets(0.001, 2, 15),
            },
            []string{"database", "user", "type"},
        ),
        
        queryCount: prometheus.NewCounterVec(
            prometheus.CounterOpts{
                Name: "tidb_queries_total",
                Help: "Total number of SQL queries",
            },
            []string{"database", "user", "type", "status"},
        ),
        
        activeConnections: prometheus.NewGauge(
            prometheus.GaugeOpts{
                Name: "tidb_active_connections",
                Help: "Number of active connections",
            },
        ),
    }
    
    // Register metrics
    collector.registry.MustRegister(
        collector.queryDuration,
        collector.queryCount,
        collector.activeConnections,
    )
    
    return collector
}

// Hook into query execution
func (c *MetricsCollector) ObserveQuery(database, user, queryType string, duration time.Duration, success bool) {
    status := "success"
    if !success {
        status = "error"
    }
    
    c.queryDuration.WithLabelValues(database, user, queryType).Observe(duration.Seconds())
    c.queryCount.WithLabelValues(database, user, queryType, status).Inc()
}
```

## 5. Storage Interface

### 5.1 Custom Storage Backend

#### Implementing Storage Interface
```go
// Custom storage implementation
type CustomStorage struct {
    config StorageConfig
    client CustomClient
}

func (s *CustomStorage) Begin() (kv.Transaction, error) {
    txn, err := s.client.BeginTransaction()
    if err != nil {
        return nil, err
    }
    
    return &CustomTransaction{
        storage: s,
        txn:     txn,
        buffer:  NewMemBuffer(),
    }, nil
}

func (s *CustomStorage) GetSnapshot(ts uint64) kv.Snapshot {
    return &CustomSnapshot{
        storage:   s,
        timestamp: ts,
        client:    s.client,
    }
}

// Custom transaction implementation
type CustomTransaction struct {
    storage   *CustomStorage
    txn       CustomTxn
    buffer    MemBuffer
    startTS   uint64
    commitTS  uint64
    valid     bool
}

func (t *CustomTransaction) Get(ctx context.Context, key kv.Key) ([]byte, error) {
    // Check memory buffer first
    if val, exists := t.buffer.Get(key); exists {
        return val, nil
    }
    
    // Fall back to storage
    return t.txn.Get(ctx, key)
}

func (t *CustomTransaction) Set(key kv.Key, value []byte) error {
    return t.buffer.Set(key, value)
}

func (t *CustomTransaction) Commit(ctx context.Context) error {
    if !t.valid {
        return errors.New("transaction is not valid")
    }
    
    // Prepare writes from buffer
    writes := t.buffer.GetWrites()
    
    // Commit through storage client
    err := t.txn.Commit(ctx, writes)
    if err != nil {
        return err
    }
    
    t.valid = false
    return nil
}
```

### 5.2 Distributed Storage Integration

#### TiKV Integration Example
```go
// Custom TiKV wrapper
type EnhancedTiKVStore struct {
    *tikv.KVStore
    
    interceptor RequestInterceptor
    metrics     *MetricsCollector
}

func (s *EnhancedTiKVStore) SendReq(ctx context.Context, req *tikvrpc.Request, regionID tikv.RegionVerID, timeout time.Duration) (*tikvrpc.Response, error) {
    // Pre-request interception
    if s.interceptor != nil {
        if err := s.interceptor.BeforeRequest(ctx, req); err != nil {
            return nil, err
        }
    }
    
    // Measure request duration
    start := time.Now()
    resp, err := s.KVStore.SendReq(ctx, req, regionID, timeout)
    duration := time.Since(start)
    
    // Record metrics
    if s.metrics != nil {
        s.metrics.RecordStorageRequest(req.Type.String(), duration, err == nil)
    }
    
    // Post-request interception
    if s.interceptor != nil {
        s.interceptor.AfterRequest(ctx, req, resp, err)
    }
    
    return resp, err
}

// Request interceptor interface
type RequestInterceptor interface {
    BeforeRequest(ctx context.Context, req *tikvrpc.Request) error
    AfterRequest(ctx context.Context, req *tikvrpc.Request, resp *tikvrpc.Response, err error)
}
```

## 6. Expression Framework

### 6.1 Custom Expression Types

#### Implementing Custom Expressions
```go
// Custom expression node
type CustomExpression struct {
    expression.BaseExpr
    
    operands []expression.Expression
    opType   CustomOpType
}

func (c *CustomExpression) Eval(ctx chunk.Row) (types.Datum, error) {
    // Evaluate operands
    args := make([]types.Datum, len(c.operands))
    for i, operand := range c.operands {
        val, err := operand.Eval(ctx)
        if err != nil {
            return types.Datum{}, err
        }
        args[i] = val
    }
    
    // Apply custom operation
    result, err := c.applyOperation(args)
    if err != nil {
        return types.Datum{}, err
    }
    
    return result, nil
}

func (c *CustomExpression) Clone() expression.Expression {
    clonedOperands := make([]expression.Expression, len(c.operands))
    for i, operand := range c.operands {
        clonedOperands[i] = operand.Clone()
    }
    
    return &CustomExpression{
        BaseExpr: c.BaseExpr,
        operands: clonedOperands,
        opType:   c.opType,
    }
}

// Expression factory
func NewCustomExpression(opType CustomOpType, operands ...expression.Expression) *CustomExpression {
    return &CustomExpression{
        BaseExpr: expression.NewBaseExpr(determineType(opType, operands), mysql.BinaryFlag),
        operands: operands,
        opType:   opType,
    }
}
```

## 7. Statistics and Cost Model

### 7.1 Custom Cost Model

#### Implementing Cost Models
```go
// Custom cost model
type CustomCostModel struct {
    config CostConfig
    
    // Cost factors
    cpuFactor     float64
    memoryFactor  float64
    diskFactor    float64
    networkFactor float64
}

func (c *CustomCostModel) GetOperatorCost(op physicalop.PhysicalPlan, children []float64) float64 {
    baseCost := sum(children)
    
    switch op := op.(type) {
    case *physicalop.PhysicalTableScan:
        return baseCost + c.getTableScanCost(op)
    case *physicalop.PhysicalIndexScan:
        return baseCost + c.getIndexScanCost(op)
    case *physicalop.PhysicalHashJoin:
        return baseCost + c.getHashJoinCost(op)
    default:
        return baseCost + c.getDefaultOperatorCost(op)
    }
}

func (c *CustomCostModel) getHashJoinCost(join *physicalop.PhysicalHashJoin) float64 {
    leftRows := join.Children()[0].Stats().RowCount
    rightRows := join.Children()[1].Stats().RowCount
    
    // Build cost: iterate through build side to create hash table
    buildCost := rightRows * c.cpuFactor
    
    // Probe cost: iterate through probe side and lookup in hash table
    probeCost := leftRows * c.cpuFactor * 0.5 // Hash lookup is cheaper than scan
    
    // Memory cost for hash table
    memoryCost := rightRows * estimateRowSize(join.Children()[1]) * c.memoryFactor
    
    return buildCost + probeCost + memoryCost
}
```

### 7.2 Custom Statistics

#### Statistics Collection Interface
```go
// Custom statistics collector
type CustomStatsCollector struct {
    sampleRate float64
    maxSamples int
}

func (c *CustomStatsCollector) CollectColumnStats(table *model.TableInfo, column *model.ColumnInfo, data []types.Datum) (*statistics.Histogram, error) {
    // Sample data if too large
    if len(data) > c.maxSamples {
        data = c.sampleData(data)
    }
    
    // Sort data for histogram construction
    slices.SortFunc(data, func(a, b types.Datum) int {
        return a.Compare(sessionctx.Background().GetSessionVars().StmtCtx, &b, collate.GetBinaryCollator())
    })
    
    // Build histogram with custom bucket strategy
    buckets := c.buildBuckets(data, column.FieldType)
    
    histogram := &statistics.Histogram{
        ID:       column.ID,
        NDV:      int64(c.estimateNDV(data)),
        Buckets:  buckets,
        Tp:       &column.FieldType,
    }
    
    return histogram, nil
}

func (c *CustomStatsCollector) buildBuckets(data []types.Datum, tp types.FieldType) []statistics.Bucket {
    bucketCount := min(len(data)/100, 254) // Max 254 buckets
    if bucketCount < 1 {
        bucketCount = 1
    }
    
    buckets := make([]statistics.Bucket, 0, bucketCount)
    itemsPerBucket := len(data) / bucketCount
    
    for i := 0; i < bucketCount; i++ {
        start := i * itemsPerBucket
        end := min((i+1)*itemsPerBucket, len(data))
        
        if i == bucketCount-1 {
            end = len(data) // Last bucket gets remaining items
        }
        
        bucket := statistics.Bucket{
            Count:  int64(end),
            Repeat: c.countRepeats(data[start:end]),
        }
        
        // Set bucket bounds
        if start < len(data) {
            bucket.LowerBound = data[start]
        }
        if end > 0 && end <= len(data) {
            bucket.UpperBound = data[end-1]
        }
        
        buckets = append(buckets, bucket)
    }
    
    return buckets
}
```

## 8. Custom Functions

### 8.1 Built-in Function Extension

#### Function Registration
```go
// Register custom function during initialization
func init() {
    // Register the function class
    expression.RegisterFunction("JSON_EXTRACT_PATH", &jsonExtractPathClass{})
}

// Function class implementation
type jsonExtractPathClass struct {
    baseFunctionClass
}

func (c *jsonExtractPathClass) getFunction(ctx sessionctx.Context, args []expression.Expression) (builtinFunc, error) {
    if err := c.verifyArgs(args); err != nil {
        return nil, err
    }
    
    argTypes := make([]types.EvalType, len(args))
    for i := range args {
        argTypes[i] = types.ETString
    }
    
    sig := &builtinJSONExtractPathSig{
        baseBuiltinFunc: newBaseBuiltinFuncWithTp(ctx, c.funcName, args, types.ETString, argTypes...),
    }
    
    sig.tp.Flen = mysql.MaxBlobWidth
    return sig, nil
}

// Function signature implementation
type builtinJSONExtractPathSig struct {
    baseBuiltinFunc
}

func (s *builtinJSONExtractPathSig) evalString(row chunk.Row) (string, bool, error) {
    // Get JSON document
    jsonDoc, isNull, err := s.args[0].EvalString(s.ctx, row)
    if isNull || err != nil {
        return "", isNull, err
    }
    
    // Get path expression
    path, isNull, err := s.args[1].EvalString(s.ctx, row)
    if isNull || err != nil {
        return "", isNull, err
    }
    
    // Extract value using custom logic
    result, err := extractJSONPath(jsonDoc, path)
    if err != nil {
        return "", false, err
    }
    
    return result, false, nil
}

func (s *builtinJSONExtractPathSig) Clone() builtinFunc {
    newSig := &builtinJSONExtractPathSig{}
    newSig.cloneFrom(&s.baseBuiltinFunc)
    return newSig
}
```

---

This API reference provides the foundation for extending TiDB's functionality. For more detailed examples and implementation guidance, refer to:

- [Architecture Analysis](architecture_analysis.md) - Understanding the overall system design
- [Best Practices](best_practices.md) - Development standards and guidelines
- [Getting Started](getting_started.md) - Setting up your development environment

For questions about specific APIs or extension development, visit the [TiDB GitHub Discussions](https://github.com/pingcap/tidb/discussions) or consult the [official TiDB documentation](https://docs.pingcap.com/tidb/stable).