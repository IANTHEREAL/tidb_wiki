# TiDB Development Wiki

Welcome to the comprehensive TiDB development documentation! This wiki provides in-depth technical documentation for developers working on TiDB, covering architecture, core modules, development workflows, and best practices.

## 🚀 What is TiDB?

TiDB is an open-source, cloud-native, distributed SQL database designed for high availability, horizontal and vertical scalability, strong consistency, and high performance. It provides MySQL compatibility while offering the benefits of a distributed architecture.

### Key Features
- **Distributed Transactions**: ACID compliance with two-phase commit protocol
- **Horizontal Scalability**: Scale computing and storage independently
- **High Availability**: Built-in Raft consensus and automated failover
- **HTAP Support**: Handle both transactional and analytical workloads
- **MySQL Compatibility**: Drop-in replacement with familiar tools and drivers
- **Cloud Native**: Kubernetes-ready with operator support

## 📚 Wiki Structure and Navigation

This wiki is organized into several comprehensive sections:

### Core Architecture Documentation
| Document | Description | Key Topics |
|----------|-------------|------------|
| **[Architecture Analysis](architecture_analysis.md)** | Complete system architecture overview | Components, data flow, design patterns, entry points |
| **[Planner and Optimizer](planner_and_optimizer.md)** | Query planning and optimization engine | Cost-based optimization, statistics, physical plans |

### Development Guides
| Document | Description | Key Topics |
|----------|-------------|------------|
| **[Getting Started](getting_started.md)** | Development environment and setup | Building, testing, debugging, common tasks |
| **[Best Practices](best_practices.md)** | Development guidelines and standards | Code style, testing, performance, security |
| **[API Reference](api_reference.md)** | Key interfaces and extension points | Plugin development, integration guidelines |

### Operational Documentation
| Document | Description | Key Topics |
|----------|-------------|------------|
| **[Troubleshooting](troubleshooting.md)** | Common issues and debugging | Error codes, performance issues, FAQ |

## 🎯 Quick Start Guide for Developers

### 1. Set Up Development Environment
```bash
# Clone the TiDB repository
git clone https://github.com/pingcap/tidb.git
cd tidb

# Install Go (version 1.21 or later)
# See getting_started.md for detailed instructions

# Build TiDB
make
```

### 2. Understand the Architecture
Start with these key files to understand TiDB's structure:
- **Entry Point**: [`cmd/tidb-server/main.go:280`](../tidb/cmd/tidb-server/main.go) - Application startup
- **Server Core**: [`pkg/server/server.go:1`](../tidb/pkg/server/server.go) - Connection handling
- **Session Management**: [`pkg/session/session.go:1`](../tidb/pkg/session/session.go) - SQL execution lifecycle
- **Query Processing**: [`pkg/executor/adapter.go:1`](../tidb/pkg/executor/adapter.go) - Query execution coordination

### 3. Key Development Workflows

#### Running Tests
```bash
# Unit tests
make test

# Integration tests
make test-race

# Specific package tests
go test ./pkg/planner/core/...
```

#### Making Changes
1. **Read**: [Architecture Analysis](architecture_analysis.md) for system overview
2. **Plan**: Understand the module you're modifying using our documentation
3. **Code**: Follow [Best Practices](best_practices.md) for development standards
4. **Test**: Write comprehensive tests for your changes
5. **Debug**: Use [Troubleshooting](troubleshooting.md) for common issues

### 4. Essential Reading Path

For new developers, we recommend this reading order:

1. **[Architecture Analysis](architecture_analysis.md)** - System overview and component relationships
2. **[Getting Started](getting_started.md)** - Set up your development environment
3. **[Planner and Optimizer](planner_and_optimizer.md)** - Deep dive into query processing
4. **[Best Practices](best_practices.md)** - Development standards and guidelines
5. **[API Reference](api_reference.md)** - Key interfaces for extension development

## 🏗️ TiDB Module Overview

### Core Components

```
TiDB Architecture
├── SQL Layer (Stateless)
│   ├── Protocol Layer      # MySQL wire protocol
│   ├── Parser             # SQL parsing and AST generation
│   ├── Planner            # Query optimization and planning
│   ├── Executor           # Query execution engine
│   └── Session Management # Connection and transaction handling
│
├── Storage Layer (Distributed)
│   ├── TiKV               # Distributed transactional storage
│   ├── TiFlash            # Columnar analytical storage
│   └── Placement Driver   # Metadata and scheduling
│
└── Ecosystem Tools
    ├── TiDB Lightning     # Data import tool
    ├── TiDB Binlog        # Data replication
    └── Backup & Restore   # Data backup solution
```

### Key Directories in Codebase

| Directory | Purpose | Key Files |
|-----------|---------|-----------|
| `cmd/tidb-server/` | Main application entry | `main.go` - Server startup |
| `pkg/server/` | Connection and protocol handling | `server.go` - Core server logic |
| `pkg/session/` | Session and transaction management | `session.go` - Session lifecycle |
| `pkg/parser/` | SQL parsing (separate module) | `parser.go` - Grammar implementation |
| `pkg/planner/` | Query optimization | `optimize.go`, `core/` - Planning logic |
| `pkg/executor/` | Query execution | `adapter.go` - Execution coordination |
| `pkg/store/` | Storage abstraction | `store.go` - Storage interface |
| `pkg/kv/` | Key-value operations | `kv.go` - Transaction interface |
| `pkg/ddl/` | Schema change operations | `ddl.go` - DDL coordination |
| `pkg/domain/` | Global state management | `domain.go` - Cluster coordination |

## 🤝 Contributing to TiDB

### Before You Start
1. **Read the Documentation**: Familiarize yourself with the architecture and modules
2. **Set Up Environment**: Follow our [Getting Started](getting_started.md) guide
3. **Find an Issue**: Look for [good first issues](https://github.com/pingcap/tidb/issues?q=is%3Aopen+is%3Aissue+label%3A%22good+first+issue%22) or [help wanted](https://github.com/pingcap/tidb/issues?q=is%3Aopen+is%3Aissue+label%3A%22help+wanted%22)

### Development Workflow
1. **Fork and Clone**: Create your development environment
2. **Create Branch**: Use descriptive branch names (`feature/add-xxx`, `fix/issue-xxx`)
3. **Follow Standards**: Adhere to our [Best Practices](best_practices.md)
4. **Write Tests**: Ensure comprehensive test coverage
5. **Submit PR**: Include clear description and link related issues

### Code Style Guidelines
- **Go Standards**: Follow standard Go conventions and formatting
- **Documentation**: Document public APIs and complex logic
- **Error Handling**: Use structured error handling with context
- **Testing**: Write table-driven tests with good coverage
- **Performance**: Consider performance implications of changes

### Getting Help
- **GitHub Issues**: Report bugs and request features
- **GitHub Discussions**: Ask questions and discuss ideas
- **Community Slack**: Join the TiDB community for real-time help
- **Documentation**: Use this wiki for technical reference

## 📖 Additional Resources

### Official Documentation
- [TiDB Official Documentation](https://docs.pingcap.com/tidb/stable)
- [TiDB Architecture Guide](https://docs.pingcap.com/tidb/stable/tidb-architecture)
- [TiDB Development Guide](https://pingcap.github.io/tidb-dev-guide/)

### Community Resources
- [TiDB GitHub Repository](https://github.com/pingcap/tidb)
- [TiDB Community](https://github.com/pingcap/community)
- [Contribution Map](https://github.com/pingcap/tidb-map/blob/master/maps/contribution-map.md)

### Learning Path
- **Beginners**: Start with [Architecture Analysis](architecture_analysis.md) and [Getting Started](getting_started.md)
- **Contributors**: Focus on [Best Practices](best_practices.md) and [API Reference](api_reference.md)
- **Query Optimization**: Deep dive into [Planner and Optimizer](planner_and_optimizer.md)
- **Troubleshooting**: Reference [Troubleshooting](troubleshooting.md) for common issues

## 📝 Wiki Maintenance

This wiki is maintained by the TiDB development community. To contribute to the documentation:

1. **Updates**: Submit PRs for corrections or improvements
2. **New Content**: Propose new documentation in GitHub issues
3. **Questions**: Use GitHub discussions for clarification
4. **Structure**: Follow the established documentation patterns

---

**Ready to contribute to TiDB?** Start with our [Getting Started Guide](getting_started.md) and join the vibrant TiDB development community!