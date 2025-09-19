# TiDB Concurrency Control

## Overview

TiDB implements sophisticated concurrency control mechanisms to ensure data consistency and prevent conflicts in a distributed environment. The system combines lock management, deadlock detection, and conflict resolution to provide ACID guarantees while maintaining high performance.

## 1. Lock Management System

### 1.1 Lock Architecture

TiDB implements a distributed lock management system that operates across multiple TiKV nodes.

**Lock Hierarchy:**
- **Row-level locks**: Fine-grained locking for individual records
- **Range locks**: For gap locking in certain isolation levels
- **Table locks**: Coarse-grained locks for DDL operations
- **Schema locks**: Database-level locking for schema changes

**Lock Storage:**
- Locks are stored in TiKV as special key-value pairs
- Lock information includes transaction ID, timestamp, and lock type
- Primary/secondary lock relationship for distributed transactions

### 1.2 Lock Types and Modes

**Write Locks (Exclusive):**
```go
// Pessimistic write lock acquired during DML operations
type Lock struct {
    Primary    []byte    // Primary lock key
    StartTS    uint64    // Transaction start timestamp
    TTL        uint64    // Time to live
    LockType   LockType  // Lock type (Put, Delete, etc.)
}
```

**Read Locks:**
- Used in pessimistic transactions for `SELECT ... FOR UPDATE`
- Prevent phantom reads in certain scenarios
- Compatible with other read locks but exclusive with write locks

**Lock Compatibility Matrix:**
```
        Read    Write   None
Read    ✓       ✗       ✓
Write   ✗       ✗       ✓
None    ✓       ✓       ✓
```

### 1.3 Fair Locking Implementation

TiDB implements fair locking to prevent lock starvation:

**Key Features:**
- **Lock queues**: Transactions wait in FIFO order
- **Priority inheritance**: High-priority transactions can inherit from waiting transactions
- **Timeout mechanisms**: Configurable lock wait timeouts

**Implementation Details:**
```go
// pkg/kv/kv.go
type FairLockingController interface {
    StartFairLocking() error
    RetryFairLocking(ctx context.Context) error
    CancelFairLocking(ctx context.Context) error
    DoneFairLocking(ctx context.Context) error
    IsInFairLockingMode() bool
}
```

**Fair Locking Process:**
1. Transaction requests lock on occupied resource
2. Transaction enters wait queue in fair order
3. Lock holder releases lock
4. Next transaction in queue acquires lock
5. Process repeats for subsequent waiters

### 1.4 Lock Context Management

**Lock Context Structure:**
- Maintains lock wait information
- Tracks lock acquisition and release
- Handles lock timeout and cancellation

**Code Reference:**
```go
// pkg/kv/kv.go
type LockCtx = tikvstore.LockCtx

// Lock acquisition interface
type Transaction interface {
    LockKeys(ctx context.Context, lockCtx *LockCtx, keys ...Key) error
    LockKeysFunc(ctx context.Context, lockCtx *LockCtx, fn func(), keys ...Key) error
}
```

## 2. Deadlock Detection

### 2.1 Deadlock Detection Architecture

TiDB implements distributed deadlock detection to identify and resolve circular wait conditions.

**Components:**
- **Deadlock Detector**: Centralized component for deadlock analysis
- **Wait-for Graph**: Tracks transaction dependencies
- **Deadlock History**: Maintains historical deadlock information

### 2.2 Deadlock Detection Algorithm

**Wait-for Graph Construction:**
1. Track transaction wait relationships
2. Build directed graph of dependencies
3. Detect cycles in the dependency graph
4. Select victim transaction for rollback

**Detection Process:**
```
Transaction A waits for lock held by Transaction B
Transaction B waits for lock held by Transaction C  
Transaction C waits for lock held by Transaction A
=> Deadlock detected: A -> B -> C -> A
```

### 2.3 Deadlock Resolution

**Victim Selection Criteria:**
- Transaction with lowest priority
- Transaction with least work done
- Transaction that started most recently
- Configurable deadlock resolution policies

**Resolution Actions:**
1. Select victim transaction
2. Rollback victim transaction
3. Release all locks held by victim
4. Allow other transactions to proceed
5. Victim transaction can retry if configured

### 2.4 Deadlock History and Monitoring

**Deadlock History Tracking:**
```go
// pkg/util/deadlockhistory/deadlock_history.go
type DeadlockHistory struct {
    // Deadlock record storage
    deadlocks []DeadlockRecord
    mu        sync.RWMutex
}

type DeadlockRecord struct {
    OccurTime     time.Time
    WaitChain     []WaitChainItem
    DetectTime    time.Time
}
```

**Monitoring Features:**
- Deadlock frequency tracking
- Wait chain analysis
- Performance impact assessment
- Historical trend analysis

## 3. Conflict Detection and Resolution

### 3.1 Write-Write Conflicts

**Detection Mechanisms:**

**Optimistic Transactions:**
- Conflicts detected at commit time
- Compare write set with committed transactions
- Use timestamp ordering for conflict resolution

**Pessimistic Transactions:**
- Conflicts detected during execution
- Lock acquisition failures indicate conflicts
- Immediate conflict detection and handling

### 3.2 Write-Read Conflicts

**MVCC-based Resolution:**
- Readers use snapshot isolation
- Writers don't block readers
- Timestamp-based visibility rules

**Visibility Rules:**
```go
// A version is visible to a transaction if:
// 1. Version's commit timestamp <= transaction's start timestamp
// 2. Version is not locked by another active transaction
// 3. Version is the latest committed version for the key
```

### 3.3 Conflict Resolution Strategies

**Automatic Retry:**
```go
// pkg/sessiontxn/interface.go
type StmtErrorAction int

const (
    StmtActionError      StmtErrorAction = iota // Return error
    StmtActionRetryReady                        // Ready for retry
    StmtActionNoIdea                           // Let other components decide
)
```

**Retry Mechanisms:**
- **Exponential backoff**: Increasing delays between retries
- **Jitter**: Random delay variation to prevent thundering herd
- **Circuit breaker**: Stop retrying after persistent failures

**Backoff Algorithm:**
```go
// pkg/kv/txn.go
func BackOff(attempts uint) int {
    upper := int(math.Min(
        float64(retryBackOffCap), 
        float64(retryBackOffBase) * math.Pow(2.0, float64(attempts))
    ))
    sleep := time.Duration(rand.Intn(upper)) * time.Millisecond
    time.Sleep(sleep)
    return int(sleep)
}
```

## 4. Lock Wait Management

### 4.1 Lock Wait Queues

**Queue Implementation:**
- FIFO ordering for fairness
- Priority-based queue jumping for urgent transactions
- Timeout-based queue management

**Wait Queue Operations:**
1. **Enqueue**: Add waiting transaction to queue
2. **Dequeue**: Remove transaction when lock becomes available
3. **Timeout**: Remove transaction after wait timeout
4. **Cancel**: Remove transaction due to external cancellation

### 4.2 Lock Wait Timeout

**Timeout Configuration:**
- Global lock wait timeout settings
- Per-transaction timeout overrides
- Statement-level timeout controls

**Timeout Handling:**
```go
// When lock wait timeout occurs:
// 1. Remove transaction from wait queue
// 2. Return lock wait timeout error
// 3. Transaction can choose to retry or abort
// 4. Update timeout metrics and logs
```

### 4.3 Lock Wait Information

**Administrative Queries:**
```go
// pkg/kv/kv.go
type Storage interface {
    GetLockWaits() ([]*deadlockpb.WaitForEntry, error)
}
```

**Wait Information Content:**
- Waiting transaction ID and timestamp
- Lock key and type being waited for
- Lock holder transaction information
- Wait start time and duration
- Wait queue position

## 5. Table-Level Locking

### 5.1 Table Lock Types

**Lock Types:**
```go
// pkg/parser/ast/ddl.go
type TableLockType int

const (
    TableLockRead      TableLockType = iota // READ lock
    TableLockWrite                          // WRITE lock  
    TableLockWriteLocal                     // WRITE LOCAL lock
    TableLockReadOnly                       // READ ONLY lock
)
```

### 5.2 Table Lock Management

**Lock Acquisition:**
- Table locks acquired through `LOCK TABLES` statement
- Prevents concurrent DDL operations
- Session-scoped lock ownership

**Lock Validation:**
```go
// pkg/lock/lock.go
type Checker struct {
    ctx context.TableLockReadContext
    is  infoschema.InfoSchema
}

func (c *Checker) CheckTableLock(db, table string, privilege mysql.PrivilegeType, alterWriteable bool) error {
    // Validate table lock permissions
}
```

### 5.3 Lock Compatibility

**Table Lock Compatibility Matrix:**
```
Operation      READ    WRITE   WRITE_LOCAL   READ_ONLY
SELECT         ✓       ✗       ✓             ✓
INSERT         ✗       ✓       ✓             ✗
UPDATE         ✗       ✓       ✓             ✗
DELETE         ✗       ✓       ✓             ✗
DDL            ✗       ✗       ✗             ✗
```

## 6. Memory Buffer Management

### 6.1 Transaction Memory Buffer

**Buffer Architecture:**
```go
// pkg/kv/kv.go
type MemBuffer interface {
    RetrieverMutator
    
    // Staging support for nested transactions
    Staging() StagingHandle
    Release(StagingHandle)
    Cleanup(StagingHandle)
    
    // Snapshot support for consistent reads
    SnapshotGetter() Getter
    SnapshotIter(k, upperbound Key) Iterator
}
```

**Staging Mechanism:**
- **Staging Handle**: Reference to staged changes
- **Release**: Commit staged changes to buffer
- **Cleanup**: Discard staged changes
- **Nested staging**: Support for savepoints

### 6.2 Memory Management

**Buffer Size Limits:**
```go
// pkg/kv/kv.go
var (
    TxnEntrySizeLimit = atomic.NewUint64(config.DefTxnEntrySizeLimit)  // 6MB default
    TxnTotalSizeLimit = atomic.NewUint64(config.DefTxnTotalSizeLimit)  // 100MB default
)
```

**Memory Tracking:**
- Per-transaction memory usage monitoring
- Automatic buffer cleanup on transaction end
- Memory pressure handling and limits

### 6.3 Write Buffering

**Write Operations:**
- All writes buffered in memory during transaction
- Efficient key-value storage with flags
- Support for delete markers and update flags

**Flush Mechanisms:**
- Automatic flush for large transactions
- Pipelined DML support for bulk operations
- Memory pressure-triggered flushing

## 7. Concurrency Control Optimizations

### 7.1 Lock-Free Operations

**Read-Only Transactions:**
- No locks required for snapshot reads
- MVCC provides consistent view without blocking
- Parallel execution with write transactions

**Atomic Operations:**
- Compare-and-swap for certain updates
- Lock-free increment/decrement operations
- Optimistic updates with validation

### 7.2 Batch Processing

**Lock Batching:**
- Multiple keys locked in single operation
- Reduced network round trips
- Atomic acquisition of multiple locks

**Commit Batching:**
- Batch commit of multiple transactions
- Reduced TSO pressure
- Improved overall throughput

### 7.3 Adaptive Algorithms

**Dynamic Lock Granularity:**
- Escalation from row locks to page/table locks
- Workload-based lock granularity decisions
- Memory usage optimization

**Smart Conflict Avoidance:**
- Hot key detection and handling
- Load balancing for heavily contended resources
- Predictive conflict detection

## 8. Performance Monitoring

### 8.1 Lock Metrics

**Key Performance Indicators:**
- Lock acquisition latency
- Lock hold duration
- Lock wait queue length
- Deadlock frequency and resolution time

### 8.2 Conflict Analysis

**Conflict Metrics:**
- Write-write conflict rate
- Transaction retry frequency
- Conflict resolution latency
- Success rate after retries

### 8.3 Resource Utilization

**Memory Usage:**
- Transaction buffer utilization
- Lock storage overhead
- Memory fragmentation analysis

**CPU Usage:**
- Deadlock detection overhead
- Lock management CPU consumption
- Conflict resolution processing time

This comprehensive concurrency control system enables TiDB to maintain data consistency while providing high concurrency and performance in distributed transaction processing environments.