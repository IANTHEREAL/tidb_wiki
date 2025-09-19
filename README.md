# TiDB Development Wiki

Welcome to the comprehensive TiDB development wiki! This documentation is designed to help developers understand, contribute to, and extend the TiDB distributed SQL database.

## 📋 Table of Contents

### 🏗️ Architecture
- [Architecture Overview](architecture/overview.md) - High-level system architecture
- [Component Interactions](architecture/interactions.md) - How components work together
- [Data Flow](architecture/data-flow.md) - Request processing and data movement

### 🧩 Core Modules
- [SQL Layer](modules/sql-layer.md) - Query parsing, planning, and execution
- [Storage Engine](modules/storage-engine.md) - TiKV and TiFlash integration
- [Distributed Execution](modules/distributed-execution.md) - Distributed query processing
- [Query Optimizer](modules/optimizer.md) - Cost-based optimization and statistics
- [DDL System](modules/ddl.md) - Schema changes and metadata management
- [Transaction System](modules/transactions.md) - ACID transactions and concurrency control

### 💻 Development
- [Setup Guide](development/setup.md) - Environment setup and build instructions
- [Code Walkthrough](development/walkthrough.md) - Guided tour through the codebase
- [Testing Guide](development/testing.md) - Running and writing tests
- [Debugging Tips](development/debugging.md) - Troubleshooting and profiling

### 🔌 APIs & Interfaces
- [Server APIs](api/server-apis.md) - HTTP and gRPC endpoints
- [Plugin System](api/plugins.md) - Extension points and plugin development
- [Internal APIs](api/internal.md) - Inter-component communication

### 🎯 Patterns & Best Practices
- [Design Patterns](patterns/design-patterns.md) - Common patterns used in TiDB
- [Code Style](patterns/code-style.md) - Coding conventions and standards
- [Performance Guidelines](patterns/performance.md) - Writing efficient code

### 📚 Guides
- [Contributing](guides/contributing.md) - How to contribute to TiDB
- [Feature Development](guides/feature-development.md) - Adding new features
- [Bug Fixing](guides/bug-fixing.md) - Identifying and fixing issues

## 🚀 Quick Start for Developers

1. **Setup Development Environment**: Follow the [Setup Guide](development/setup.md)
2. **Understand the Architecture**: Read the [Architecture Overview](architecture/overview.md)
3. **Explore a Module**: Start with the [SQL Layer](modules/sql-layer.md) documentation
4. **Walk Through Code**: Use the [Code Walkthrough](development/walkthrough.md) guide
5. **Make Your First Contribution**: See [Contributing Guide](guides/contributing.md)

## 🎯 Key Learning Paths

### New to TiDB
```
Architecture Overview → SQL Layer → Development Setup → Code Walkthrough
```

### Database Engine Developer
```
Storage Engine → Transactions → Optimizer → Distributed Execution
```

### Performance Engineer
```
Performance Guidelines → Optimizer → Storage Engine → Debugging Tips
```

### Plugin Developer
```
Plugin System → APIs → Design Patterns → Feature Development
```

## 📖 Additional Resources

- [Official TiDB Documentation](https://docs.pingcap.com/tidb/stable)
- [TiDB Design Documents](../docs/design/)
- [Community Forums](https://asktug.com)
- [GitHub Issues](https://github.com/pingcap/tidb/issues)

## 🤝 Community

Join the TiDB developer community:
- [Discord](https://discord.gg/KVRZBR2DrG)
- [Slack](https://slack.tidb.io/invite?team=tidb-community&channel=everyone&ref=pingcap-tidb)
- [Discussions](https://github.com/orgs/pingcap/discussions)

---

*This wiki is maintained by the TiDB community. Contributions and improvements are welcome!*