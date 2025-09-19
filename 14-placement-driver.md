# TiDB Placement Driver Integration and Metadata Management

## Overview

The Placement Driver (PD) is the central coordinator in TiDB's distributed architecture, responsible for metadata management, timestamp allocation, and cluster coordination. This document analyzes how TiDB integrates with PD and manages distributed metadata.

## 1. PD Client Architecture

### Core Components
- **PD Client Interface**: Abstracts PD communication
- **Region Cache**: Caches region topology information
- **Timestamp Oracle**: Manages global timestamp allocation
- **Metadata Sync**: Synchronizes schema and configuration changes

### Key Integration Points
- **Timestamp Service (TSO)**: Global timestamp allocation for MVCC
- **Metadata Service**: Schema version management and synchronization
- **Region Management**: Region split/merge coordination
- **Configuration Management**: Global configuration distribution

## 2. PD Communication Patterns

### Mock PD Client (`pkg/store/mockstore/unistore/pd.go`)

For development and testing, TiDB provides a mock PD implementation:

```go
type pdClient struct {
    *us.MockPD
    pd.ResourceManagerClient
    globalConfig      map[string]string
    externalTimestamp atomic.Uint64
    addrs             []string
    keyspaceMeta      *keyspacepb.KeyspaceMeta
}
```

### Core PD Operations

#### Timestamp Service
- **TSO Allocation**: `GetTS()` - Allocates globally unique timestamps
- **Batch TSO**: `GetTSAsync()` - Batch timestamp allocation for performance
- **Local TSO**: Regional timestamp allocation for multi-DC deployments

#### Region Management
- **Region Discovery**: `GetRegion()` - Locate regions by key
- **Region Routing**: `GetStore()` - Find stores serving specific regions
- **Region Updates**: Handle region split/merge notifications

#### Configuration Management
- **Global Config**: `LoadGlobalConfig()`, `StoreGlobalConfig()`
- **Config Watch**: `WatchGlobalConfig()` - Real-time configuration updates
- **Keyspace Management**: Multi-tenant keyspace coordination

### Communication Protocols
- **gRPC**: Primary communication protocol with PD
- **HTTP**: Administrative and monitoring interfaces
- **Service Discovery**: Automatic PD endpoint discovery and failover

## 3. Metadata Management Architecture

### InfoSchema System (`pkg/infoschema/`)

TiDB's information schema provides a unified view of database metadata:

```go
type infoSchema struct {
    infoSchemaMisc
    schemaMap           map[string]*schemaTables
    schemaID2Name       map[int64]string
    sortedTablesBuckets []sortedTables
    referredForeignKeyMap map[SchemaAndTableName][]*model.ReferredFKInfo
    r                   autoid.Requirement
}
```

### Schema Version Management
- **Version Tracking**: `schemaMetaVersion` tracks current schema state
- **Change Propagation**: Schema changes propagated via PD
- **Consistency Guarantees**: Ensures all nodes see consistent schema

### Metadata Storage Structure (`pkg/meta/meta.go`)

```
Meta structure:
    NextGlobalID -> int64
    SchemaVersion -> int64
    DBs -> {
        DB:1 -> db meta data []byte
        DB:2 -> db meta data []byte
    }
    DB:1 -> {
        Table:1 -> table meta data []byte
        Table:2 -> table meta data []byte
        TID:1 -> int64
        TID:2 -> int64
    }
```

### Key Metadata Components

#### Database Metadata
- **Schema Information**: Database definitions and properties
- **Table Definitions**: Table structure, columns, indexes
- **Partition Information**: Partitioning schemes and boundaries
- **Placement Rules**: Data placement policies

#### System Metadata
- **Global IDs**: Unique identifier allocation
- **Schema Versions**: Change tracking and synchronization
- **Bootstrap Information**: Cluster initialization state
- **Statistics Metadata**: Query optimization statistics

## 4. Schema Synchronization

### Schema Change Workflow
1. **DDL Initiation**: Schema change request submitted
2. **Version Increment**: New schema version allocated
3. **PD Coordination**: Change distributed via PD
4. **Local Updates**: Each TiDB node updates local schema cache
5. **Consistency Check**: Verify all nodes are synchronized

### InfoSchema Caching
- **Local Cache**: Each TiDB node maintains schema cache
- **Cache Invalidation**: PD notifications trigger cache updates
- **Version Checks**: Queries validate schema version consistency

### Schema Builder (`pkg/infoschema/builder.go`)
Responsible for constructing and updating information schema:

```go
type Builder struct {
    is *infoSchema
    handle *Handle
    store kv.Storage
    factory func() (pools.Resource, error)
}
```

## 5. Global ID Management

### Auto-ID Service (`pkg/meta/autoid/`)

Manages unique identifier allocation across the cluster:

- **Global ID Allocation**: Distributed unique ID generation
- **Range Allocation**: Batch ID allocation for performance
- **Auto-increment Columns**: Support for MySQL-compatible auto-increment
- **Sequence Objects**: SQL standard sequence support

### ID Types
- **Table IDs**: Unique identifiers for tables
- **Column IDs**: Unique identifiers for columns
- **Index IDs**: Unique identifiers for indexes
- **Partition IDs**: Unique identifiers for partitions

## 6. Region Metadata Management

### Region Information
PD maintains comprehensive region metadata:

- **Region Boundaries**: Key range definitions
- **Replica Placement**: Store assignments for each replica
- **Region State**: Health and availability status
- **Load Statistics**: Access patterns and performance metrics

### Region Cache (`pkg/store/copr/region_cache.go`)
```go
type RegionCache struct {
    *tikv.RegionCache
}
```

#### Cache Operations
- **Region Lookup**: Map keys to regions
- **Cache Updates**: Handle region splits and merges
- **Store Discovery**: Locate stores serving regions
- **Invalidation**: Remove stale region information

### Region Routing
1. **Key Analysis**: Parse table/index keys
2. **Region Lookup**: Find responsible region
3. **Store Selection**: Choose appropriate store replica
4. **Request Routing**: Direct requests to target store

## 7. Configuration Management

### Global Configuration
PD manages cluster-wide configuration:

- **System Variables**: MySQL-compatible system variables
- **Feature Flags**: Enable/disable cluster features
- **Resource Limits**: Memory, CPU, and I/O constraints
- **Security Settings**: Authentication and authorization

### Configuration Distribution
- **Push Model**: PD pushes config changes to nodes
- **Watch Mechanism**: Nodes subscribe to configuration updates
- **Atomic Updates**: Ensures consistent configuration across cluster

### Configuration Hierarchy
1. **Global Defaults**: Cluster-wide default values
2. **Instance Overrides**: Node-specific overrides
3. **Session Settings**: Per-connection settings
4. **Statement Hints**: Query-specific overrides

## 8. Placement Policies

### Placement Rules
PD manages data placement policies:

```go
type ruleBundleMap map[int64]*placement.Bundle
type policyMap map[string]*model.PolicyInfo
```

#### Policy Types
- **Leader Rules**: Specify leader replica placement
- **Follower Rules**: Define follower replica distribution
- **Learner Rules**: Configure learner replica placement
- **Isolation Rules**: Enforce data isolation constraints

### Resource Groups
- **Resource Allocation**: CPU, memory, and I/O quotas
- **Priority Management**: Query execution priorities
- **Tenant Isolation**: Multi-tenant resource isolation

## 9. Monitoring and Observability

### Metrics Collection
PD collects comprehensive cluster metrics:

- **Performance Metrics**: Latency, throughput, error rates
- **Resource Utilization**: CPU, memory, disk, network usage
- **Schema Statistics**: Table sizes, access patterns
- **System Health**: Node status, region distribution

### Alerting and Monitoring
- **Threshold-based Alerts**: Automated anomaly detection
- **Dashboard Integration**: Grafana/Prometheus integration
- **Log Aggregation**: Centralized logging and analysis

## 10. High Availability and Disaster Recovery

### PD Cluster Management
- **Leader Election**: Raft-based PD leader selection
- **Consensus Protocol**: Consistent metadata updates
- **Failover Handling**: Automatic PD failover
- **Split-brain Prevention**: Network partition handling

### Backup and Recovery
- **Metadata Backup**: Regular backup of cluster metadata
- **Point-in-time Recovery**: Restore to specific timestamps
- **Cross-region Replication**: Multi-DC disaster recovery

## 11. Performance Optimizations

### Caching Strategies
- **Multi-level Caching**: Local, distributed, and persistent caches
- **Cache Warming**: Proactive cache population
- **Eviction Policies**: LRU and time-based eviction

### Batch Operations
- **Batch Requests**: Reduce PD communication overhead
- **Async Processing**: Non-blocking metadata operations
- **Request Coalescing**: Combine similar requests

### Network Optimization
- **Connection Pooling**: Reuse connections to PD
- **Compression**: Reduce network bandwidth usage
- **Keep-alive**: Maintain persistent connections

## 12. Security and Access Control

### Authentication
- **Certificate-based Auth**: TLS client certificates
- **Token-based Auth**: JWT tokens for service authentication
- **RBAC Integration**: Role-based access control

### Encryption
- **TLS Encryption**: Encrypted communication with PD
- **At-rest Encryption**: Encrypted metadata storage
- **Key Management**: Secure key distribution and rotation

## Summary

TiDB's integration with Placement Driver demonstrates sophisticated distributed systems design:

1. **Centralized Coordination**: PD provides global coordination while maintaining high availability
2. **Metadata Consistency**: Ensures all nodes have consistent view of cluster state
3. **Scalable Architecture**: Handles growing clusters with millions of regions
4. **Performance Optimization**: Multiple caching layers and batch operations
5. **Operational Excellence**: Comprehensive monitoring, alerting, and disaster recovery

The PD integration is crucial for TiDB's ability to operate as a distributed SQL database, providing the foundation for consistent metadata management, global transaction coordination, and dynamic cluster scaling.