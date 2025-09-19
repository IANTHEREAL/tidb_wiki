# TiDB Architecture Overview

TiDB is a distributed SQL database that provides HTAP (Hybrid Transactional/Analytical Processing) capabilities. This document provides a comprehensive overview of TiDB's architecture and how its components work together.

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Client Applications                    │
└─────────────────────┬───────────────────────────────────┘
                      │ MySQL Protocol
┌─────────────────────▼───────────────────────────────────┐
│                     TiDB Server                         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │
│  │SQL Parser   │ │Query Planner│ │  Executor   │       │
│  └─────────────┘ └─────────────┘ └─────────────┘       │
└─────────────────────┬───────────────────────────────────┘
                      │ gRPC/HTTP
        ┌─────────────▼─────────────┐
        │        Placement Driver    │
        │         (PD Cluster)      │
        └─────────────┬─────────────┘
                      │ Metadata & Scheduling
    ┌─────────────────▼─────────────────┐
    │           Storage Layer           │
    │  ┌───────────┐    ┌────────────┐  │
    │  │   TiKV    │    │  TiFlash   │  │
    │  │(Row Store)│    │(Column Store)│ │
    │  └───────────┘    └────────────┘  │
    └───────────────────────────────────┘
```

## 🔧 Core Components

### 1. TiDB Server (Compute Layer)

The stateless compute layer that handles SQL processing:

- **Location**: `pkg/server/`, `pkg/session/`, `pkg/executor/`
- **Responsibilities**:
  - MySQL protocol compatibility
  - SQL parsing and compilation
  - Query optimization
  - Distributed execution coordination
  - Session management

**Key Modules**:
- **Parser** (`pkg/parser/`): Converts SQL text to AST
- **Planner** (`pkg/planner/`): Generates and optimizes execution plans
- **Executor** (`pkg/executor/`): Executes queries across storage nodes
- **Session** (`pkg/session/`): Manages client connections and transactions

### 2. Placement Driver (PD)

The metadata management and cluster coordination service:

- **Repository**: [tikv/pd](https://github.com/tikv/pd)
- **Responsibilities**:
  - Metadata storage (schema, table definitions)
  - Timestamp allocation (TSO)
  - Region scheduling and load balancing
  - Cluster configuration management

### 3. TiKV (Row-Based Storage)

The distributed key-value storage engine for OLTP workloads:

- **Repository**: [tikv/tikv](https://github.com/tikv/tikv)
- **Responsibilities**:
  - Distributed transactions (2PC)
  - Raft consensus for replication
  - MVCC (Multi-Version Concurrency Control)
  - Coprocessor for computation pushdown

### 4. TiFlash (Columnar Storage)

The columnar storage engine for OLAP workloads:

- **Repository**: [pingcap/tiflash](https://github.com/pingcap/tiflash)
- **Responsibilities**:
  - Real-time data replication from TiKV
  - Columnar data storage
  - MPP (Massively Parallel Processing) execution
  - OLAP query acceleration

## 🔄 Request Processing Flow

### OLTP Query Flow

```
1. Client → TiDB Server (MySQL Protocol)
2. TiDB Parser → SQL to AST
3. TiDB Planner → Optimization & Plan Generation
4. TiDB Executor → Plan Execution
5. TiDB → TiKV (via gRPC)
6. TiKV → Data Processing & Return
7. TiDB → Result Aggregation
8. TiDB → Client (Result Set)
```

### OLAP Query Flow

```
1. Client → TiDB Server (MySQL Protocol)
2. TiDB Parser → SQL to AST
3. TiDB Planner → MPP Plan Generation
4. TiDB → TiFlash Nodes (MPP Execution)
5. TiFlash → Parallel Processing
6. TiFlash → TiDB (Partial Results)
7. TiDB → Final Aggregation
8. TiDB → Client (Result Set)
```

### Transaction Flow

```
1. BEGIN → TiDB allocates transaction ID from PD
2. SQL Execution → Read/Write operations on TiKV
3. COMMIT → Two-Phase Commit (2PC) protocol
   - Phase 1: Prewrite (prepare phase)
   - Phase 2: Commit (commit phase)
4. Transaction completion
```

## 🧱 Key Architectural Patterns

### 1. Separation of Compute and Storage

- **Compute** (TiDB): Stateless, horizontally scalable
- **Storage** (TiKV/TiFlash): Persistent, distributed with replication

### 2. Multi-Raft Architecture

- Data partitioned into **Regions** (96MB by default)
- Each Region replicated using **Raft consensus**
- **Leader/Follower** model for read/write operations

### 3. HTAP Unified Processing

- **TiKV**: Optimized for transactional workloads
- **TiFlash**: Optimized for analytical workloads
- **Automatic data sync** between row and column stores

### 4. Distributed Transaction Model

- **Optimistic transactions** with conflict detection
- **Two-Phase Commit** protocol
- **MVCC** for snapshot isolation
- **Global timestamp ordering** via PD

## 📊 Data Distribution and Sharding

### Region-Based Sharding

```go
// Regions are key ranges with replication
type Region struct {
    ID       uint64
    StartKey []byte
    EndKey   []byte
    Peers    []*Peer  // 3 replicas by default
}
```

### Key Design

TiDB uses an ordered key-value model:

```
Table Data:    t{tableID}_r{rowID} → row data
Index Data:    t{tableID}_i{indexID}_{indexValues} → rowID
Meta Data:     m{metaKey} → schema information
```

## 🔧 Component Integration Points

### TiDB ↔ PD Integration

- **Location**: `pkg/domain/`, `pkg/store/helper/`
- **Functions**:
  - Schema management
  - Timestamp allocation
  - Region routing information

### TiDB ↔ TiKV Integration

- **Location**: `pkg/store/driver/`, `pkg/store/copr/`
- **Functions**:
  - Key-value operations
  - Coprocessor requests
  - Transaction coordination

### TiDB ↔ TiFlash Integration

- **Location**: `pkg/executor/`, `pkg/store/`
- **Functions**:
  - MPP task distribution
  - Columnar scan operations
  - Result collection

## 🏃‍♂️ Performance Considerations

### Query Processing Optimization

1. **Predicate Pushdown**: Push filters to storage layer
2. **Projection Pushdown**: Select only needed columns
3. **Aggregation Pushdown**: Perform aggregations at storage
4. **Join Optimization**: Hash joins, merge joins, index joins

### Storage Optimization

1. **Region Splitting**: Automatic data distribution
2. **Load Balancing**: Even distribution across nodes
3. **Compaction**: Background data organization
4. **Bloom Filters**: Efficient key existence checks

## 📈 Scalability Characteristics

### Horizontal Scalability

- **TiDB Servers**: Add compute capacity
- **TiKV Nodes**: Add storage capacity
- **TiFlash Nodes**: Add analytical processing power

### Performance Scaling

- **Read Performance**: Scales with TiKV/TiFlash nodes
- **Write Performance**: Scales with TiKV nodes and region distribution
- **Analytical Performance**: Scales with TiFlash MPP parallelism

## 🔍 Monitoring and Observability

### Key Metrics

- **QPS**: Queries per second across components
- **Latency**: P99, P95, P50 response times
- **Resource Usage**: CPU, memory, disk, network
- **Region Health**: Split, merge, and balance operations

### Tools

- **Prometheus**: Metrics collection
- **Grafana**: Visualization dashboards
- **Jaeger**: Distributed tracing
- **TiDB Dashboard**: Cluster management UI

## 🚀 Next Steps

- [Component Interactions](interactions.md) - Detailed inter-component communication
- [Data Flow](data-flow.md) - Step-by-step request processing
- [SQL Layer Deep Dive](../modules/sql-layer.md) - Understanding the compute layer
- [Storage Engine Details](../modules/storage-engine.md) - TiKV and TiFlash internals