# TiDB Architecture Analysis

## Table of Contents

1. [Overall Architecture Overview](#1-overall-architecture-overview)
2. [Core Modules Analysis](#2-core-modules-analysis)
3. [Data Flow - SQL Query Execution](#3-data-flow---sql-query-execution)
4. [Key Design Patterns](#4-key-design-patterns)
5. [Module Dependencies](#5-module-dependencies)
6. [Entry Points](#6-entry-points)
7. [Critical Files](#7-critical-files)
8. [Testing and Development](#8-testing-and-development)
9. [Architectural Strengths](#9-architectural-strengths)
10. [Key Insights for New Developers](#10-key-insights-for-new-developers)

## Overview

TiDB is a distributed, NewSQL, HTAP (Hybrid Transactional/Analytical Processing) database that provides MySQL compatibility with horizontal scalability. This document provides a comprehensive analysis of TiDB's architecture based on exploration of the codebase located at `~/tidb`.

> **Related Documentation**: 
> - [Planner and Optimizer](planner_and_optimizer.md) - Deep dive into query processing
> - [Getting Started](getting_started.md) - Development environment setup
> - [Best Practices](best_practices.md) - Development standards and guidelines

## 1. Overall Architecture Overview

TiDB follows a layered, distributed architecture that separates computation from storage:

### Major Components

1. **TiDB Server** (`pkg/server/`) - SQL computing layer
2. **Parser** (`pkg/parser/`) - SQL parsing and AST generation
3. **Planner** (`pkg/planner/`) - Query optimization and plan generation
4. **Executor** (`pkg/executor/`) - Query execution engine
5. **Storage Layer** (`pkg/store/`, `pkg/kv/`) - Key-value storage abstraction
6. **Session Management** (`pkg/session/`) - Connection and session handling
7. **Domain** (`pkg/domain/`) - Global state and schema management
8. **DDL** (`pkg/ddl/`) - Data Definition Language operations
9. **Transaction** (`pkg/sessiontxn/`) - Transaction management

### Architecture Characteristics

- **Stateless SQL Layer**: TiDB servers are stateless and can be horizontally scaled
- **Shared-Nothing Architecture**: Each component can scale independently
- **MySQL Protocol Compatibility**: Full wire protocol compatibility with MySQL
- **ACID Compliance**: Strong consistency with distributed transactions
- **Pluggable Storage**: Abstracted storage layer supporting multiple backends

## 2. Core Modules Analysis

### 2.1 Server Module (`pkg/server/`)
**Location**: `pkg/server/server.go:1`

**Responsibilities**:
- MySQL wire protocol implementation
- Connection management and authentication
- Request routing and session coordination
- Performance monitoring and metrics collection

**Key Interfaces**:
- `Server` struct for main server functionality
- Connection handling and pool management
- Protocol-level communication with clients

### 2.2 Parser Module (`pkg/parser/`)
**Location**: `pkg/parser/parser.go:1`

**Responsibilities**:
- SQL statement parsing using yacc-generated parser
- AST (Abstract Syntax Tree) construction
- Syntax validation and error reporting
- MySQL dialect compatibility

**Key Features**:
- Generated parser from grammar files
- Comprehensive AST node types
- MySQL syntax compatibility layer

### 2.3 Session Module (`pkg/session/`)
**Location**: `pkg/session/session.go:1`

**Responsibilities**:
- Session context management
- Variable and configuration handling
- Transaction lifecycle management
- Security and privilege enforcement

**Key Components**:
- Session state management
- Variable scope handling (SESSION, GLOBAL, INSTANCE)
- Transaction boundary control

### 2.4 Planner Module (`pkg/planner/`)
**Location**: `pkg/planner/core/optimizer.go:1`

**Responsibilities**:
- Logical plan generation from AST
- Cost-based optimization
- Rule-based transformations
- Physical plan selection

**Key Optimization Rules**:
- Column pruning and projection elimination
- Predicate pushdown
- Join reordering and optimization
- Index selection and access path optimization

### 2.5 Executor Module (`pkg/executor/`)
**Location**: `pkg/executor/adapter.go:1`

**Responsibilities**:
- Physical plan execution
- Data retrieval and processing
- Result set generation
- Performance monitoring and profiling

**Execution Models**:
- Volcano-style iterator model
- Vectorized execution for analytical workloads
- Distributed execution coordination

### 2.6 Storage Abstraction (`pkg/kv/`, `pkg/store/`)
**Location**: `pkg/kv/kv.go:1`, `pkg/store/store.go:1`

**Responsibilities**:
- Storage engine abstraction
- Transaction interface
- Key-value operations
- Storage driver registration

**Storage Backends**:
- TiKV (distributed transactional storage)
- MockTiKV (testing)
- UniStore (embedded storage)

### 2.7 Domain Module (`pkg/domain/`)
**Location**: `pkg/domain/domain.go:1`

**Responsibilities**:
- Global state management
- Schema information caching
- Statistics collection and management
- Cross-component coordination

**Key Functions**:
- Information schema management
- Global configuration synchronization
- Background task coordination

### 2.8 DDL Module (`pkg/ddl/`)
**Location**: `pkg/ddl/ddl.go:1`

**Responsibilities**:
- Schema change operations
- Online DDL execution
- Distributed coordination of schema changes
- Backward compatibility maintenance

**DDL Features**:
- Online schema changes
- Distributed coordination via etcd
- Multi-version schema support

### 2.9 Configuration Module (`pkg/config/`)
**Location**: `pkg/config/config.go:1`

**Responsibilities**:
- Configuration management and validation
- Runtime configuration updates
- Environment-specific settings
- Default value management

### 2.10 Extension System (`pkg/extension/`)
**Responsibilities**:
- Plugin architecture
- Custom function registration
- Feature extension points
- Third-party integration

## 3. Data Flow - SQL Query Execution

The following diagram illustrates how a SQL query flows through TiDB:

```
Client → Server → Parser → Planner → Executor → Storage
   ↑                                               ↓
   └─────────────── Result Set ←──────────────────┘
```

### Detailed Query Flow:

1. **Connection Establishment** (`pkg/server/`)
   - Client connects via MySQL protocol
   - Authentication and session initialization
   - Connection pooling and management

2. **SQL Parsing** (`pkg/parser/`)
   - Raw SQL text parsing
   - AST generation and validation
   - Syntax error detection

3. **Logical Planning** (`pkg/planner/`)
   - AST to logical plan conversion
   - Privilege checking and validation
   - Initial optimization rules application

4. **Physical Planning** (`pkg/planner/`)
   - Cost-based optimization
   - Access path selection
   - Join algorithm selection
   - Distributed execution planning

5. **Execution** (`pkg/executor/`)
   - Physical plan instantiation
   - Data retrieval from storage
   - Result computation and aggregation
   - Transaction coordination

6. **Storage Operations** (`pkg/kv/`, `pkg/store/`)
   - Key-value operations
   - Transaction management
   - Distributed coordination

## 4. Key Design Patterns

### 4.1 Layered Architecture
- Clear separation of concerns between layers
- Abstraction interfaces between components
- Pluggable storage backends

### 4.2 Interface-Based Design
- Heavy use of Go interfaces for abstraction
- Dependency injection patterns
- Mockable components for testing

### 4.3 Factory Pattern
- Storage driver registration and creation
- Executor factory for different plan types
- Configuration object creation

### 4.4 Observer Pattern
- Event-driven DDL notifications
- Statistics update triggers
- Configuration change propagation

### 4.5 Command Pattern
- DDL job execution
- Transaction operation batching
- Distributed task coordination

### 4.6 Strategy Pattern
- Multiple optimization strategies
- Different storage backends
- Execution algorithms selection

## 5. Module Dependencies

### Core Dependency Graph:
```
cmd/tidb-server
    ↓
pkg/server ← pkg/config
    ↓
pkg/session ← pkg/domain ← pkg/ddl
    ↓           ↓
pkg/executor ← pkg/planner ← pkg/parser
    ↓           ↓
pkg/kv ← pkg/store
```

### Key Relationships:

- **Server** depends on **Session**, **Domain**, **Config**
- **Session** depends on **Executor**, **Planner**, **Domain**
- **Executor** depends on **Planner**, **KV**, **Expression**
- **Planner** depends on **Parser**, **Expression**, **Statistics**
- **Domain** coordinates **DDL**, **InfoSchema**, **Statistics**
- **Storage** provides abstraction over **KV** backends

### Module Isolation:
- Parser is standalone with its own go.mod
- Core modules are well-separated with clear interfaces
- Storage layer is completely abstracted

## 6. Entry Points

### 6.1 Main Entry Point
**File**: `cmd/tidb-server/main.go:280`

**Startup Sequence**:
1. Configuration initialization and validation
2. Storage driver registration
3. Metrics and logging setup
4. Domain creation and schema loading
5. Server instantiation and startup
6. Signal handling and graceful shutdown

### 6.2 Server Creation
**File**: `cmd/tidb-server/main.go:985`

**Key Steps**:
- TiDB driver creation with storage backend
- Server configuration and initialization
- Domain association
- Background services startup

### 6.3 Storage Initialization
**File**: `cmd/tidb-server/main.go:478`

**Process**:
- Keyspace management
- Storage backend connection
- DDL owner manager startup
- Session bootstrap

## 7. Critical Files

### 7.1 Core Server Files
- `cmd/tidb-server/main.go` - Main application entry point
- `pkg/server/server.go` - Core server implementation
- `pkg/session/session.go` - Session management core

### 7.2 Query Processing Files
- `pkg/parser/parser.go` - SQL parser implementation
- `pkg/planner/core/optimizer.go` - Query optimization engine
- `pkg/executor/adapter.go` - Execution coordination

### 7.3 Storage Interface Files
- `pkg/kv/kv.go` - Key-value storage interface
- `pkg/store/store.go` - Storage driver management
- `pkg/domain/domain.go` - Global state management

### 7.4 Configuration Files
- `pkg/config/config.go` - Configuration management
- `Makefile` - Build and development commands
- `go.mod` - Go module dependencies

### 7.5 Schema Management Files
- `pkg/ddl/ddl.go` - DDL operation coordination
- `pkg/infoschema/` - Schema information management
- `pkg/meta/` - Metadata operations

## 8. Testing and Development

### Test Structure:
- Unit tests co-located with source files (`*_test.go`)
- Integration tests in `tests/` directory
- Benchmark tests for performance validation

### Development Tools:
- `cmd/importer/` - Data import utilities
- `cmd/benchdb/`, `cmd/benchkv/` - Performance benchmarking
- `tools/` - Development and maintenance utilities

## 9. Architectural Strengths

1. **Modularity**: Clear module boundaries with well-defined interfaces
2. **Scalability**: Stateless design enables horizontal scaling
3. **Extensibility**: Plugin architecture supports feature additions
4. **Testability**: Interface-based design enables comprehensive testing
5. **Compatibility**: Strong MySQL compatibility ensures easy migration
6. **Performance**: Optimized query processing and execution pipelines

## 10. Key Insights for New Developers

1. **Start with**: `cmd/tidb-server/main.go` to understand startup flow
2. **Understand**: The session lifecycle in `pkg/session/session.go`
3. **Learn**: Query flow from parser through executor
4. **Explore**: Storage abstraction in `pkg/kv/` and `pkg/store/`
5. **Study**: Configuration management in `pkg/config/`
6. **Focus on**: Interface definitions before implementation details

This architecture analysis provides a foundation for understanding TiDB's sophisticated distributed database design and can serve as a guide for both newcomers and experienced developers working with the codebase.

---

## Related Documentation

- **[Planner and Optimizer](planner_and_optimizer.md)** - Detailed analysis of query planning and optimization
- **[Getting Started](getting_started.md)** - Set up your development environment 
- **[Best Practices](best_practices.md)** - Code style and development guidelines
- **[API Reference](api_reference.md)** - Key interfaces for extension development
- **[Troubleshooting](troubleshooting.md)** - Common issues and debugging techniques

## Next Steps

1. **For New Contributors**: Start with [Getting Started](getting_started.md) to set up your development environment
2. **For Query Optimization**: Deep dive into [Planner and Optimizer](planner_and_optimizer.md) 
3. **For Extension Development**: Reference [API Reference](api_reference.md) for interfaces
4. **For Code Quality**: Follow [Best Practices](best_practices.md) guidelines
5. **For Issues**: Consult [Troubleshooting](troubleshooting.md) for common problems