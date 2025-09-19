# TiDB Transaction System

## Overview

TiDB implements a distributed transaction system that supports both optimistic and pessimistic transaction models with ACID guarantees. The system is built on a sophisticated Multi-Version Concurrency Control (MVCC) foundation that provides snapshot isolation and other isolation levels across a distributed cluster.

## 1. Transaction Models

### 1.1 Pessimistic Transactions

TiDB's default transaction model that provides better compatibility with MySQL. Pessimistic transactions acquire locks during execution to prevent conflicts.

**Key Characteristics:**
- Locks are acquired immediately when data is accessed
- Provides better compatibility with existing applications
- Reduces transaction retry overhead
- Implements fair locking to prevent starvation

**Implementation Details:**
- Located in `pkg/sessiontxn/isolation/` directory
- Uses `PessimisticTxnContextProvider` for transaction context management
- Implements write conflict detection at statement level
- Supports statement-level rollback for better error handling

**Code Reference:**
```go
// pkg/sessiontxn/isolation/repeatable_read.go
type PessimisticRRTxnContextProvider struct {
    baseTxnContextProvider
    // Pessimistic specific fields
}
```

### 1.2 Optimistic Transactions

The original transaction model that delays conflict detection until commit time.

**Key Characteristics:**
- No locks acquired during transaction execution
- Conflict detection happens at commit time
- Higher concurrency but may require retries
- Better for workloads with low conflict probability

**Implementation Details:**
- Uses `OptimisticTxnContextProvider` for context management
- Implements automatic retry mechanism with configurable limits
- Conflicts detected through timestamp validation at commit

**Code Reference:**
```go
// pkg/sessiontxn/isolation/optimistic.go
type OptimisticTxnContextProvider struct {
    baseTxnContextProvider
    optimizeWithMaxTS bool
}
```

## 2. MVCC Implementation

### 2.1 Core Concepts

TiDB's MVCC system provides multiple consistent views of data through timestamp-based versioning.

**Key Components:**
- **Start Timestamp (StartTS)**: When transaction begins reading
- **Commit Timestamp (CommitTS)**: When transaction commits
- **Primary/Secondary Lock Model**: Distributed two-phase commit
- **Async Commit**: Optimization for reducing commit latency

### 2.2 Version Storage

**Implementation Details:**
- Each key-value pair has multiple versions tagged with timestamps
- Garbage collection removes old versions based on safe points
- Lock information stored separately from data versions

**Code Reference:**
```go
// pkg/kv/kv.go
type Transaction interface {
    StartTS() uint64
    CommitTS() uint64
    Commit(context.Context) error
    Rollback() error
    // ... other methods
}
```

### 2.3 Lock Management

**Primary/Secondary Lock Model:**
- One primary lock per transaction coordinates commit
- Secondary locks reference the primary lock
- Atomic commit through primary lock resolution

**Lock Types:**
- **Write Locks**: Exclusive locks for modifications
- **Read Locks**: For pessimistic read operations
- **Fair Locks**: Prevention of lock starvation

## 3. Timestamp Oracle (TSO)

### 3.1 Overview

TiDB uses PD (Placement Driver) as the Timestamp Oracle to provide globally unique, monotonically increasing timestamps.

**Key Functions:**
- Provides transaction start timestamps
- Ensures global ordering of transactions
- Implements hybrid logical clocks for high performance

### 3.2 TSO Integration

**Implementation Details:**
- TSO requests are batched for performance
- Future timestamp fetching for optimization
- Causal consistency guarantees across regions

**Code Reference:**
```go
// pkg/kv/kv.go (TSO usage)
type Storage interface {
    GetOracle() oracle.Oracle
    CurrentVersion(txnScope string) (Version, error)
    // ... other methods
}
```

### 3.3 Timestamp Allocation

**Allocation Strategy:**
- Batch allocation reduces PD pressure
- Adaptive batching based on workload
- Region-aware timestamp allocation for geo-distributed deployments

## 4. Conflict Detection and Resolution

### 4.1 Write Conflicts

**Detection Mechanisms:**
- **Optimistic**: Detected at commit time through timestamp validation
- **Pessimistic**: Detected during execution through lock acquisition

**Resolution Strategies:**
- Automatic retry for optimistic transactions
- Statement-level retry for pessimistic transactions
- Configurable retry limits and backoff strategies

### 4.2 Read-Write Conflicts

**Snapshot Isolation:**
- Readers never block writers
- Writers never block readers
- Consistent snapshot based on start timestamp

**Implementation:**
- MVCC ensures readers see consistent snapshot
- Write operations check for newer committed versions
- Timestamp-based conflict detection

### 4.3 Error Handling

**Retryable Errors:**
- Write conflicts in optimistic mode
- Lock wait timeout in pessimistic mode
- Region unavailable errors

**Code Reference:**
```go
// pkg/sessiontxn/interface.go
type StmtErrorAction int
const (
    StmtActionError StmtErrorAction = iota
    StmtActionRetryReady
    StmtActionNoIdea
)
```

## 5. Isolation Levels

### 5.1 Snapshot Isolation (SI)

**Default Isolation Level:**
- Transactions see consistent snapshot at start time
- Prevents dirty reads, non-repeatable reads, and phantom reads
- Allows some write skew anomalies

**Implementation:**
- Each transaction reads from snapshot at StartTS
- Writes are validated against concurrent commits
- Uses timestamp ordering for consistency

### 5.2 Read Committed (RC)

**Statement-Level Consistency:**
- Each statement sees latest committed data
- Allows non-repeatable reads and phantom reads
- Better performance for certain workloads

**Implementation Details:**
- New read timestamp for each statement
- Reduced isolation guarantees for better concurrency
- Statement-level snapshot creation

### 5.3 Read Committed with Timestamp Check (RCCheckTS)

**Enhanced Read Committed:**
- Adds timestamp validation for better consistency
- Balances performance and consistency requirements
- Used for specific workload optimizations

**Code Reference:**
```go
// pkg/kv/kv.go
type IsoLevel int
const (
    SI IsoLevel = iota  // Snapshot Isolation
    RC                  // Read Committed
    RCCheckTS          // Read Committed with TS Check
)
```

### 5.4 Isolation Level Implementation

**Context Providers:**
- `RepeatableReadTxnContextProvider`: Implements snapshot isolation
- `ReadCommittedTxnContextProvider`: Implements read committed
- Specialized providers for each isolation level

**Dynamic Switching:**
- Isolation level can be changed per transaction
- Session-level and statement-level controls
- Optimizations based on isolation requirements

## 6. Transaction Lifecycle

### 6.1 Transaction Begin

**Process:**
1. Allocate start timestamp from TSO
2. Create transaction context with appropriate provider
3. Initialize snapshot for reading
4. Set up conflict detection mechanisms

### 6.2 Statement Execution

**Read Operations:**
- Use snapshot at appropriate timestamp
- MVCC version selection based on visibility
- Lock acquisition for pessimistic reads

**Write Operations:**
- Buffer writes in transaction memory
- Acquire locks for pessimistic mode
- Validate conflicts for optimistic mode

### 6.3 Transaction Commit

**Two-Phase Commit:**
1. **Prewrite Phase**: Acquire locks and validate
2. **Commit Phase**: Atomically commit or rollback

**Optimizations:**
- Async commit for single-region transactions
- One-phase commit for read-only transactions
- Parallel prewrite for multiple regions

### 6.4 Error Recovery

**Retry Mechanisms:**
- Automatic retry for transient errors
- Exponential backoff for retry timing
- Circuit breaker for persistent failures

**Rollback Handling:**
- Automatic rollback on commit failure
- Lock cleanup for failed transactions
- Memory buffer cleanup

## 7. Performance Optimizations

### 7.1 Timestamp Optimizations

**Batch TSO Requests:**
- Reduce PD round trips
- Adaptive batching based on load
- Future timestamp allocation

### 7.2 Lock Optimizations

**Fair Locking:**
- Prevents lock starvation
- Queue-based lock waiting
- Priority-based lock allocation

### 7.3 Memory Management

**Transaction Buffers:**
- Efficient memory buffer for writes
- Staged changes with rollback support
- Memory usage tracking and limits

**Code Reference:**
```go
// pkg/kv/kv.go
type MemBuffer interface {
    RetrieverMutator
    Staging() StagingHandle
    Release(StagingHandle)
    Cleanup(StagingHandle)
    // ... other methods
}
```

## 8. Monitoring and Observability

### 8.1 Transaction Metrics

**Key Metrics:**
- Transaction duration and throughput
- Conflict rates and retry counts
- Lock wait times and deadlock frequency
- TSO allocation latency

### 8.2 Debugging Support

**Transaction Tracing:**
- Transaction lifecycle tracking
- Lock acquisition and release events
- Conflict detection and resolution

**Administrative Commands:**
- Lock wait information viewing
- Transaction status inspection
- Deadlock history analysis

This comprehensive transaction system enables TiDB to provide ACID guarantees in a distributed environment while maintaining high performance and scalability through sophisticated concurrency control mechanisms.