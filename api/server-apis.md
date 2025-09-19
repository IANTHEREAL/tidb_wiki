# TiDB Server APIs

TiDB provides several APIs for monitoring, administration, and integration. This document covers the HTTP REST APIs, status endpoints, and administrative interfaces.

## 🌐 HTTP Status API

TiDB server exposes HTTP endpoints on the status port (default: 10080) for monitoring and administration.

### Base URL
```
http://localhost:10080
```

### Core Endpoints

#### 1. Server Status

**GET /status**
```bash
curl http://localhost:10080/status
```

Response:
```json
{
  "connections": 0,
  "version": "5.7.25-TiDB-v7.5.0",
  "git_hash": "abcd1234"
}
```

#### 2. Server Information

**GET /info**
```bash
curl http://localhost:10080/info
```

Response:
```json
{
  "type": "tidb",
  "version": {
    "version": "7.5.0",
    "git_hash": "abcd1234",
    "build_time": "2024-01-15 10:30:00"
  },
  "status_port": 10080,
  "lease": "45s"
}
```

#### 3. Health Check

**GET /info/all**
```bash
curl http://localhost:10080/info/all
```

Response includes detailed server health information.

### Metrics and Monitoring

#### 4. Prometheus Metrics

**GET /metrics**
```bash
curl http://localhost:10080/metrics
```

Returns Prometheus-compatible metrics for monitoring.

#### 5. Profile Endpoints

**GET /debug/pprof/**
```bash
# CPU profile
curl http://localhost:10080/debug/pprof/profile > cpu.prof

# Memory profile  
curl http://localhost:10080/debug/pprof/heap > mem.prof

# Goroutine dump
curl http://localhost:10080/debug/pprof/goroutine > goroutines.prof
```

### Schema and Statistics

#### 6. Schema Information

**GET /schema**
```bash
curl http://localhost:10080/schema
```

**GET /schema/{db}**
```bash
curl http://localhost:10080/schema/test
```

**GET /schema/{db}/{table}**
```bash
curl http://localhost:10080/schema/test/users
```

#### 7. Table Statistics

**GET /stats/dump/{db}/{table}**
```bash
curl http://localhost:10080/stats/dump/test/users
```

### DDL Operations

#### 8. DDL History

**GET /ddl/history**
```bash
curl http://localhost:10080/ddl/history
```

Response:
```json
{
  "jobs": [
    {
      "id": 123,
      "type": "create table",
      "state": "synced",
      "start_time": "2024-01-15T10:30:00Z"
    }
  ]
}
```

### Settings and Configuration

#### 9. System Variables

**GET /settings**
```bash
curl http://localhost:10080/settings
```

**POST /settings**
```bash
curl -X POST http://localhost:10080/settings \
  -H "Content-Type: application/json" \
  -d '{"tidb_general_log": "1"}'
```

#### 10. Connection Management

**GET /info/all**
```bash
curl http://localhost:10080/info/all
```

Shows active connections and session information.

## 🔌 gRPC Internal APIs

TiDB components communicate via gRPC. These are internal APIs primarily used by TiDB components.

### Coprocessor API

Used for distributed query execution:

```protobuf
service Coprocessor {
  rpc Coprocess(Request) returns (Response);
  rpc CoprocessStream(Request) returns (stream Response);
}

message Request {
  Context context = 1;
  int64 tp = 2;
  bytes data = 3;
  repeated KeyRange ranges = 4;
  bool is_cache_enabled = 5;
}
```

### MPP API (TiFlash Integration)

For analytical query processing:

```protobuf
service MPP {
  rpc EstablishMPPConnection(EstablishMPPConnectionRequest) 
    returns (stream MPPDataPacket);
  rpc DispatchMPPTask(DispatchMPPTaskRequest) 
    returns (DispatchMPPTaskResponse);
}
```

## 🗄️ Information Schema Extensions

TiDB extends MySQL's INFORMATION_SCHEMA with TiDB-specific tables:

### TiDB-Specific Tables

#### CLUSTER_INFO
```sql
SELECT * FROM INFORMATION_SCHEMA.CLUSTER_INFO;
```

Shows cluster topology information.

#### TIDB_INDEXES  
```sql
SELECT * FROM INFORMATION_SCHEMA.TIDB_INDEXES 
WHERE table_schema = 'test';
```

Index usage statistics.

#### STATEMENTS_SUMMARY
```sql
SELECT * FROM INFORMATION_SCHEMA.STATEMENTS_SUMMARY 
ORDER BY sum_latency DESC LIMIT 10;
```

Query performance statistics.

#### SLOW_QUERY
```sql
SELECT * FROM INFORMATION_SCHEMA.SLOW_QUERY 
WHERE time > '2024-01-15 10:00:00' 
ORDER BY time DESC;
```

Slow query logs.

## 🔧 Administrative Commands

### ADMIN Commands

TiDB provides special ADMIN SQL commands:

#### Statistics Management
```sql
-- Analyze table statistics
ANALYZE TABLE test.users;

-- Show table statistics
SHOW STATS_BUCKETS USING test.users;

-- Load statistics
ADMIN LOAD STATS_EXTENDED;
```

#### DDL Management
```sql
-- Show DDL jobs
ADMIN SHOW DDL JOBS;

-- Cancel DDL job
ADMIN CANCEL DDL JOBS 123;

-- Resume DDL job  
ADMIN RESUME DDL JOBS 123;
```

#### Cluster Management
```sql
-- Show cluster configuration
ADMIN SHOW CONFIG;

-- Reload configuration
ADMIN RELOAD CONFIG;

-- Check table consistency
ADMIN CHECK TABLE test.users;
```

## 📊 Monitoring Integration

### Grafana Dashboard API

TiDB integrates with monitoring systems:

#### Metrics Categories

1. **Server Metrics**
   - Connection count
   - QPS (Queries Per Second)
   - Query duration
   - Memory usage

2. **Storage Metrics** 
   - TiKV response time
   - Region count
   - Raft status

3. **Query Metrics**
   - Slow queries
   - Failed queries
   - Plan cache hit rate

### Custom Monitoring

```go
// Custom metrics in TiDB code
import "github.com/prometheus/client_golang/prometheus"

var (
    queryCounter = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "tidb_query_total",
            Help: "Total number of queries",
        },
        []string{"type", "status"},
    )
)

func recordQuery(queryType, status string) {
    queryCounter.WithLabelValues(queryType, status).Inc()
}
```

## 🔐 Security APIs

### Authentication

TiDB supports multiple authentication methods:

#### User Management
```sql
-- Create user
CREATE USER 'app_user'@'%' IDENTIFIED BY 'password';

-- Grant privileges
GRANT SELECT, INSERT ON test.* TO 'app_user'@'%';

-- Show grants
SHOW GRANTS FOR 'app_user'@'%';
```

#### Role-Based Access Control
```sql
-- Create role
CREATE ROLE 'app_role';

-- Grant role to user
GRANT 'app_role' TO 'app_user'@'%';

-- Set default role
SET DEFAULT ROLE 'app_role' TO 'app_user'@'%';
```

### TLS Configuration

Configure TLS for secure connections:

```toml
# config.toml
[security]
ssl-cert = "/path/to/server-cert.pem"
ssl-key = "/path/to/server-key.pem"
ssl-ca = "/path/to/ca-cert.pem"
cluster-ssl-cert = "/path/to/cluster-cert.pem"
cluster-ssl-key = "/path/to/cluster-key.pem"
cluster-ssl-ca = "/path/to/cluster-ca-cert.pem"
```

## 🚀 Client Integration Examples

### Go Client

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/go-sql-driver/mysql"
)

func main() {
    // Connect to TiDB
    db, err := sql.Open("mysql", "root:@tcp(localhost:4000)/test")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    
    // Execute query
    rows, err := db.Query("SELECT id, name FROM users WHERE status = ?", "active")
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()
    
    // Process results
    for rows.Next() {
        var id int
        var name string
        if err := rows.Scan(&id, &name); err != nil {
            log.Fatal(err)
        }
        fmt.Printf("ID: %d, Name: %s\n", id, name)
    }
}
```

### Python Client

```python
import pymysql
import requests

# Database connection
connection = pymysql.connect(
    host='localhost',
    port=4000,
    user='root',
    password='',
    database='test'
)

try:
    with connection.cursor() as cursor:
        # Execute query
        cursor.execute("SELECT id, name FROM users WHERE status = %s", ("active",))
        results = cursor.fetchall()
        
        for row in results:
            print(f"ID: {row[0]}, Name: {row[1]}")
            
finally:
    connection.close()

# Monitor via HTTP API
response = requests.get('http://localhost:10080/status')
status = response.json()
print(f"Active connections: {status['connections']}")
```

### Java Client

```java
import java.sql.*;

public class TiDBExample {
    public static void main(String[] args) {
        String url = "jdbc:mysql://localhost:4000/test";
        String user = "root";
        String password = "";
        
        try (Connection conn = DriverManager.getConnection(url, user, password)) {
            // Prepare statement
            String sql = "SELECT id, name FROM users WHERE status = ?";
            try (PreparedStatement stmt = conn.prepareStatement(sql)) {
                stmt.setString(1, "active");
                
                // Execute query
                try (ResultSet rs = stmt.executeQuery()) {
                    while (rs.next()) {
                        int id = rs.getInt("id");
                        String name = rs.getString("name");
                        System.out.printf("ID: %d, Name: %s%n", id, name);
                    }
                }
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

## 🔍 Debugging and Troubleshooting APIs

### Query Execution Analysis

#### EXPLAIN Commands
```sql
-- Show execution plan
EXPLAIN SELECT * FROM users WHERE age > 25;

-- Show actual execution statistics
EXPLAIN ANALYZE SELECT * FROM users WHERE age > 25;

-- Show optimizer trace
SET SESSION tidb_trace_optimizer_logical = ON;
EXPLAIN SELECT * FROM users WHERE age > 25;
```

#### Plan Management
```sql
-- Show cached plans
SELECT * FROM INFORMATION_SCHEMA.STATEMENTS_SUMMARY;

-- Bind execution plan
CREATE BINDING FOR SELECT * FROM users WHERE age > 25 
USING SELECT /*+ USE_INDEX(users, idx_age) */ * FROM users WHERE age > 25;

-- Show bindings
SHOW BINDINGS;
```

### Performance Analysis

#### Slow Query Analysis
```bash
# Get slow queries via API
curl "http://localhost:10080/info/all" | jq '.slow_query'

# Query slow query table
mysql -h localhost -P 4000 -u root -e "
SELECT query_time, query, time 
FROM INFORMATION_SCHEMA.SLOW_QUERY 
WHERE time > NOW() - INTERVAL 1 HOUR 
ORDER BY query_time DESC 
LIMIT 10;"
```

## 📚 Related Documentation

- [Plugin System](plugins.md) - Extending TiDB functionality
- [Internal APIs](internal.md) - Component communication interfaces  
- [Performance Guidelines](../patterns/performance.md) - Optimization best practices
- [Contributing Guidelines](../guides/contributing.md) - Development workflow