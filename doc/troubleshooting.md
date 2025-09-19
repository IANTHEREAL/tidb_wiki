# TiDB Development Troubleshooting Guide

This guide helps you diagnose and resolve common issues encountered during TiDB development, from build problems to performance debugging.

## Table of Contents

1. [Build and Compilation Issues](#1-build-and-compilation-issues)
2. [Runtime and Testing Problems](#2-runtime-and-testing-problems)
3. [Performance Debugging](#3-performance-debugging)
4. [Error Codes and Meanings](#4-error-codes-and-meanings)
5. [Development Environment Issues](#5-development-environment-issues)
6. [Database Connectivity Problems](#6-database-connectivity-problems)
7. [Debugging Tools and Techniques](#7-debugging-tools-and-techniques)
8. [FAQ](#8-faq)

## 1. Build and Compilation Issues

### 1.1 Go Version Problems

#### Issue: `go version too old`
```bash
Error: This project requires Go 1.21 or later
```

**Solution**:
```bash
# Check current version
go version

# Update Go (Ubuntu/Debian)
wget https://go.dev/dl/go1.21.0.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz

# Update PATH
export PATH=/usr/local/go/bin:$PATH
```

#### Issue: `GOPATH not set correctly`
```bash
Error: cannot find package "github.com/pingcap/tidb/pkg/..."
```

**Solution**:
```bash
# Check Go environment
go env GOPATH
go env GOROOT

# Set correct GOPATH
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin

# Enable Go modules (recommended)
go env -w GO111MODULE=on
```

### 1.2 Parser Generation Issues

#### Issue: `yacc: command not found`
```bash
make: yacc: No such file or directory
make: *** [parser] Error 127
```

**Solution**:
```bash
# Install yacc parser generator
# Ubuntu/Debian
sudo apt-get install byacc

# macOS
brew install byacc

# Regenerate parser
make clean
make
```

#### Issue: `parser.go generation failed`
```bash
Error: failed to generate parser.go
```

**Solution**:
```bash
# Clean and regenerate parser files
cd pkg/parser
rm -f parser.go y.output
cd ../..
make clean
make
```

### 1.3 Dependency Issues

#### Issue: `module not found`
```bash
go: module github.com/pingcap/tidb/pkg/xyz not found
```

**Solution**:
```bash
# Update dependencies
go mod download
go mod tidy

# Clear module cache if corrupted
go clean -modcache
go mod download
```

#### Issue: `replace directive conflicts`
```bash
Error: replace directive conflicts with existing module
```

**Solution**:
```bash
# Check go.mod for conflicts
cat go.mod | grep replace

# Remove conflicting replace directives
go mod edit -dropreplace github.com/conflicting/module
go mod tidy
```

### 1.4 Build Performance Issues

#### Issue: `build takes too long`

**Solutions**:
```bash
# Use build cache
export GOCACHE=/tmp/go-cache

# Increase build parallelism
make -j$(nproc)

# Use specific targets
make server  # Build only server binary
```

## 2. Runtime and Testing Problems

### 2.1 Test Failures

#### Issue: `test timeout`
```bash
panic: test timed out after 10m0s
```

**Solution**:
```bash
# Increase test timeout
go test -timeout 30m ./pkg/your_package/

# Run tests with less parallelism
go test -parallel 1 ./pkg/your_package/

# Run specific test
go test -run TestSpecificFunction ./pkg/your_package/
```

#### Issue: `race condition detected`
```bash
WARNING: DATA RACE
Write at 0x00c000xxx by goroutine X:
```

**Solution**:
```bash
# Run with race detector for debugging
go test -race ./pkg/your_package/

# Fix by adding proper synchronization
// Add mutex, channels, or atomic operations

# Example fix:
var mu sync.RWMutex
func (s *struct) SafeRead() {
    mu.RLock()
    defer mu.RUnlock()
    // read operation
}
```

#### Issue: `flaky tests`
```bash
Test sometimes passes, sometimes fails
```

**Solution**:
```bash
# Run test multiple times to confirm flakiness
go test -count=100 -run TestFlakyFunction ./pkg/your_package/

# Common causes and fixes:
# 1. Timing issues - add proper synchronization
# 2. Cleanup issues - ensure proper test teardown
# 3. Global state - isolate test state
```

### 2.2 Memory Issues

#### Issue: `out of memory during tests`
```bash
fatal error: runtime: out of memory
```

**Solution**:
```bash
# Increase available memory
export GOMEMLIMIT=8GB

# Run tests individually
go test -run TestSpecific ./pkg/your_package/

# Check for memory leaks
go test -memprofile=mem.prof ./pkg/your_package/
go tool pprof mem.prof
```

### 2.3 Startup Issues

#### Issue: `server fails to start`
```bash
Error: failed to create server
```

**Solution**:
```bash
# Check configuration
./bin/tidb-server --config-check

# Use minimal configuration
./bin/tidb-server --store=mocktikv --host=127.0.0.1 --port=4000

# Check port availability
netstat -tlnp | grep 4000
```

## 3. Performance Debugging

### 3.1 Slow Query Performance

#### Issue: `query execution is slow`

**Debugging Steps**:
```bash
# Enable slow query log
SET SESSION tidb_slow_log_threshold = 1000; -- 1 second

# Analyze query plan
EXPLAIN FORMAT='verbose' SELECT * FROM table WHERE condition;

# Check statistics
SHOW STATS_META WHERE table_name = 'your_table';
SHOW STATS_BUCKETS WHERE table_name = 'your_table';

# Update statistics if stale
ANALYZE TABLE your_table;
```

#### Performance Analysis Tools:
```bash
# CPU profiling
go test -cpuprofile=cpu.prof -bench=BenchmarkYourFunction ./pkg/your_package/
go tool pprof cpu.prof

# Memory profiling
go test -memprofile=mem.prof -bench=BenchmarkYourFunction ./pkg/your_package/
go tool pprof mem.prof

# Trace analysis
go test -trace=trace.out -bench=BenchmarkYourFunction ./pkg/your_package/
go tool trace trace.out
```

### 3.2 Optimizer Issues

#### Issue: `poor query plan selection`

**Debugging**:
```sql
-- Enable optimizer trace
SET SESSION tidb_optimizer_trace_level = 1;

-- Run query and check trace
EXPLAIN FORMAT='trace' SELECT * FROM table WHERE condition;

-- Check cardinality estimates
EXPLAIN FORMAT='verbose' SELECT * FROM table WHERE condition;
```

**Common Solutions**:
```sql
-- Force specific index
SELECT /*+ USE_INDEX(table, index_name) */ * FROM table WHERE condition;

-- Force join order
SELECT /*+ LEADING(t1, t2) */ * FROM t1 JOIN t2 ON condition;

-- Force join algorithm
SELECT /*+ HASH_JOIN(t1, t2) */ * FROM t1 JOIN t2 ON condition;
```

### 3.3 Memory Usage Issues

#### Issue: `high memory consumption`

**Analysis**:
```bash
# Monitor memory usage
go tool pprof http://localhost:10080/debug/pprof/heap

# Check for memory leaks
go tool pprof http://localhost:10080/debug/pprof/allocs

# Analyze garbage collection
GODEBUG=gctrace=1 ./bin/tidb-server
```

## 4. Error Codes and Meanings

### 4.1 Common Error Codes

| Error Code | Name | Description | Solution |
|------------|------|-------------|----------|
| **1046** | `ER_NO_DB_ERROR` | No database selected | `USE database_name` |
| **1054** | `ER_BAD_FIELD_ERROR` | Unknown column | Check column name and table schema |
| **1062** | `ER_DUP_ENTRY` | Duplicate entry for key | Handle unique constraint violation |
| **1064** | `ER_PARSE_ERROR` | SQL syntax error | Check SQL syntax |
| **1146** | `ER_NO_SUCH_TABLE` | Table doesn't exist | Create table or check name |
| **1213** | `ER_LOCK_DEADLOCK` | Deadlock found | Retry transaction with backoff |
| **2013** | `CR_SERVER_LOST` | Lost connection | Check network and server status |

### 4.2 TiDB-Specific Errors

| Error Code | Description | Solution |
|------------|-------------|----------|
| **8001** | Memory quota exceeded | Increase `tidb_mem_quota_query` |
| **8002** | Transaction too large | Split into smaller transactions |
| **8003** | Admin check table failed | Run `ADMIN CHECK TABLE` for details |
| **8004** | Invalid auto_random value | Check auto_random configuration |
| **8005** | WriteConflict in tikv | Retry transaction or reduce concurrency |

### 4.3 Parser Errors

#### Issue: `syntax error near 'keyword'`
```sql
Error 1064: You have an error in your SQL syntax
```

**Common Causes**:
1. **Reserved Keywords**: Use backticks around reserved words
   ```sql
   -- Wrong
   CREATE TABLE order (id INT);
   
   -- Correct
   CREATE TABLE `order` (id INT);
   ```

2. **Missing Commas**: Check column definitions
   ```sql
   -- Wrong
   CREATE TABLE t (id INT name VARCHAR(50));
   
   -- Correct
   CREATE TABLE t (id INT, name VARCHAR(50));
   ```

## 5. Development Environment Issues

### 5.1 IDE Configuration Problems

#### Issue: `Go language server not working`

**VS Code Solution**:
```bash
# Reinstall Go tools
Ctrl+Shift+P -> "Go: Install/Update Tools" -> Select All

# Check settings.json
{
    "go.useLanguageServer": true,
    "go.toolsManagement.autoUpdate": true
}
```

#### Issue: `import paths not resolving`

**Solution**:
```bash
# Ensure you're in the correct directory
cd /path/to/tidb

# Check go.mod file exists
ls go.mod

# Refresh IDE cache
# VS Code: Reload window
# GoLand: File -> Invalidate Caches and Restart
```

### 5.2 Git and Version Control Issues

#### Issue: `large file checkout problems`
```bash
error: unable to create file pkg/parser/parser.go: File too large
```

**Solution**:
```bash
# Enable Git LFS if needed
git lfs install
git lfs track "*.go"

# Or increase Git buffer
git config http.postBuffer 524288000
```

## 6. Database Connectivity Problems

### 6.1 Connection Issues

#### Issue: `connection refused`
```bash
Error 2003: Can't connect to MySQL server on 'localhost:4000'
```

**Solutions**:
```bash
# Check if TiDB is running
ps aux | grep tidb-server

# Check port binding
netstat -tlnp | grep 4000

# Start TiDB with correct parameters
./bin/tidb-server --host=0.0.0.0 --port=4000 --store=mocktikv
```

#### Issue: `authentication failed`
```bash
Error 1045: Access denied for user 'root'@'localhost'
```

**Solution**:
```sql
-- Connect without authentication first
mysql -h 127.0.0.1 -P 4000 -u root

-- Set password if needed
SET PASSWORD FOR 'root'@'%' = 'your_password';
```

### 6.2 Transaction Issues

#### Issue: `lock wait timeout exceeded`
```bash
Error 1205: Lock wait timeout exceeded; try restarting transaction
```

**Solution**:
```sql
-- Increase lock wait timeout
SET SESSION innodb_lock_wait_timeout = 120;

-- Check for blocking transactions
SELECT * FROM INFORMATION_SCHEMA.PROCESSLIST WHERE state = 'Waiting';

-- Kill blocking transaction if necessary
KILL CONNECTION_ID;
```

## 7. Debugging Tools and Techniques

### 7.1 Logging and Tracing

#### Enable Debug Logging:
```bash
# Set log level
export TIDB_LOG_LEVEL=debug

# Enable slow query logging
SET GLOBAL tidb_slow_log_threshold = 0;

# Enable optimizer tracing
SET SESSION tidb_optimizer_trace_level = 1;
```

#### Trace Specific Operations:
```sql
-- Trace query optimization
SET SESSION tidb_enable_optimizer_trace = 1;
TRACE SELECT * FROM table WHERE condition;
SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;

-- Trace execution
SET SESSION tidb_enable_exec_trace = 1;
```

### 7.2 Profiling Tools

#### CPU Profiling:
```bash
# Profile running server
go tool pprof http://localhost:10080/debug/pprof/profile?seconds=30

# Profile specific test
go test -cpuprofile=cpu.prof -run TestFunction ./pkg/your_package/
go tool pprof cpu.prof
```

#### Memory Profiling:
```bash
# Heap profile
go tool pprof http://localhost:10080/debug/pprof/heap

# Allocation profile
go tool pprof http://localhost:10080/debug/pprof/allocs
```

### 7.3 Debugging with Delve

```bash
# Debug server startup
dlv exec ./bin/tidb-server -- --store=mocktikv

# Debug specific test
dlv test ./pkg/session/ -- -test.run TestSessionBasic

# Common Delve commands
(dlv) break main.main
(dlv) continue
(dlv) print variable_name
(dlv) goroutines
(dlv) stack
```

## 8. FAQ

### Q: How do I run only a specific test?
```bash
go test -run TestSpecificFunction ./pkg/your_package/
```

### Q: How do I check if my changes broke anything?
```bash
# Run relevant tests
go test ./pkg/your_package/

# Run full test suite
make test

# Check for race conditions
make test-race
```

### Q: How do I debug SQL parsing issues?
```bash
# Test SQL parsing directly
cd pkg/parser
go test -run TestParser

# Debug with custom SQL
echo "YOUR_SQL_HERE" | go run debug_parser.go
```

### Q: How do I profile memory usage during tests?
```bash
go test -memprofile=mem.prof ./pkg/your_package/
go tool pprof mem.prof
```

### Q: How do I find memory leaks?
```bash
# Run with memory profiling
go test -memprofile=mem.prof -memprofilerate=1 ./pkg/your_package/

# Compare heap snapshots
go tool pprof -diff_base=baseline.prof current.prof
```

### Q: How do I debug deadlocks?
```bash
# Enable deadlock detection
export GODEBUG=deadlock=1

# Get goroutine dump
kill -SIGQUIT <tidb_pid>

# Or via HTTP endpoint
curl http://localhost:10080/debug/pprof/goroutine?debug=1
```

### Q: How do I test with different storage engines?
```bash
# MockTiKV (for unit tests)
./bin/tidb-server --store=mocktikv

# UniStore (embedded)
./bin/tidb-server --store=unistore --path=/tmp/unistore

# TiKV (distributed)
./bin/tidb-server --store=tikv --path=127.0.0.1:2379
```

### Q: How do I benchmark my changes?
```bash
# Run benchmarks
go test -bench=. ./pkg/your_package/

# Compare with baseline
go test -bench=. -count=5 ./pkg/your_package/ > new.txt
# (run baseline)
go test -bench=. -count=5 ./pkg/your_package/ > old.txt

# Use benchcmp tool
benchcmp old.txt new.txt
```

---

**Still having issues?** 
- Check our [Architecture Analysis](architecture_analysis.md) for system understanding
- Review [Best Practices](best_practices.md) for development guidelines
- Ask questions in [GitHub Discussions](https://github.com/pingcap/tidb/discussions)
- Join the [TiDB Community Slack](https://slack.tidb.io/invite?team=tidb-community&channel=everyone)