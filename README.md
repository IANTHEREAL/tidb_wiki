# TiDB Developer Documentation Wiki

Welcome to the comprehensive TiDB development documentation. This wiki provides detailed insights into TiDB's internal architecture, development practices, and optimization strategies for contributors and developers working with TiDB's codebase.

## 📋 Table of Contents

### 🏗️ [01. Architecture Overview](./01-architecture-overview.md)
**Comprehensive system architecture and design principles**

- **High-Level Architecture**: Three-tier distributed architecture
- **Core Architectural Principles**: Separation of concerns, horizontal scalability, strong consistency
- **TiDB Server Internal Architecture**: Request processing pipeline and component overview
- **Storage Architecture**: TiKV integration and key-value abstraction
- **Schema and Metadata Management**: Domain and DDL system
- **Data Flow and Component Interactions**: Query execution and DDL operation flow
- **Configuration and Runtime Management**: Configuration system and monitoring
- **Extensibility and Plugin Architecture**: Plugin system and extension points
- **Performance Considerations**: Optimization strategies and caching layers
- **Error Handling and Recovery**: Error management and failure recovery

**Key Insights**: Understanding TiDB's distributed systems design with clear separation between compute, storage, and coordination layers.

---

### 🔧 [02. Core Components Deep Dive](./02-core-components.md)
**Detailed analysis of TiDB's major internal components**

#### SQL Parser (`/pkg/parser/`)
- **Architecture and Design**: YACC-based parser with MySQL compatibility
- **Key Components**: Lexical analysis, parser implementation, AST node definitions
- **Implementation Highlights**: Expression parsing and MySQL compatibility features
- **Error Handling and Recovery**: Context-aware error reporting
- **Performance Optimizations**: Token caching and memory management

#### Query Planner and Optimizer (`/pkg/planner/`)
- **Architecture Overview**: Two-stage optimization approach
- **Logical Optimization**: Rule-based transformations and optimization rules
- **Physical Optimization**: Cost-based plan selection and physical operators
- **Statistics-Based Optimization**: Histogram and cardinality estimation

#### Execution Engine (`/pkg/executor/`)
- **Execution Architecture**: Volcano model with vectorized processing
- **Executor Framework**: Base executor interface and chunk-based execution
- **Key Executors Implementation**: Table scan, join, and aggregation executors
- **Distributed Execution (DistSQL)**: Coprocessor integration and parallel execution
- **Memory Management**: Memory usage tracking and spill-to-disk strategy

#### Storage Interface (`/pkg/kv/`, `/pkg/store/`)
- **Storage Abstraction Layer**: Key-value interface and TiKV communication
- **Transaction Implementation**: Two-phase commit protocol and snapshot isolation
- **Memory Buffer Interface**: Transaction local storage and union store

#### Expression System (`/pkg/expression/`)
- **Expression Architecture**: Expression evaluation and built-in functions
- **Core Expression Types**: Base interface and scalar functions
- **Vectorized Evaluation**: Batch processing for performance optimization

**Key Insights**: Deep understanding of how SQL queries are parsed, optimized, and executed in a distributed environment.

---

### 🔄 [03. Module Interactions and Data Flow](./03-module-interactions.md)
**Understanding how TiDB components work together**

#### High-Level Data Flow
- **Query Processing Flow**: From SQL text to result sets
- **Connection and Session Management**: Connection establishment and session lifecycle
- **SQL Processing Pipeline**: Parser-to-planner-to-executor integration

#### Transaction and Storage Integration
- **Transaction Lifecycle**: Begin, execute, commit workflow
- **Storage Layer Interactions**: KV interface to TiKV communication
- **DDL Operations and Schema Management**: DDL state machine and schema version management

#### Distributed Execution Coordination
- **DistSQL Processing**: Coprocessor request building and execution
- **Error Handling and Recovery**: Error propagation and recovery strategies

**Key Insights**: How components coordinate to maintain consistency and performance in a distributed environment.

---

### 💻 [04. Development Guide](./04-development-guide.md)
**Complete guide for TiDB development**

#### Development Environment Setup
- **Prerequisites**: System requirements and required tools
- **Setting Up Development Environment**: Repository setup and IDE configuration
- **Environment Variables**: Development configuration and Makefile targets

#### Code Organization and Standards
- **Directory Structure Guidelines**: Package organization and file naming conventions
- **Coding Standards**: Go style guidelines, error handling patterns, interface design principles
- **Documentation Standards**: Function and package documentation examples

#### Building and Testing
- **Build System**: Makefile targets and build configuration
- **Testing Strategy**: Test organization and unit testing patterns
- **Mock and Test Utilities**: Mock frameworks and test data generators

#### Debugging and Profiling
- **Debugging Techniques**: Logging configuration and debugging with Delve
- **Performance Profiling**: CPU profiling, memory profiling, execution plan analysis

#### Contributing Guidelines
- **Development Workflow**: Branch strategy and commit message format
- **Code Review Process**: Review checklist and response examples
- **Performance Considerations**: Memory management, goroutine management, allocation optimization

**Key Insights**: Establishing effective development workflows and maintaining code quality standards.

---

### 🔌 [05. API and Interfaces Documentation](./05-api-interfaces.md)
**Comprehensive documentation of internal APIs and interfaces**

#### Core Interfaces
- **Plan Interface Hierarchy**: Base plan, logical plan, and physical plan interfaces
- **Plan Implementation Examples**: Logical and physical join plans

#### Executor Interfaces
- **Base Executor Framework**: Core executor interface and base implementation
- **Specialized Executor Interfaces**: Join and aggregation executor interfaces

#### Storage Interfaces
- **Key-Value Storage Interface**: Main storage, transaction, and snapshot interfaces
- **Memory Buffer Interface**: Transaction local storage and union store

#### Session and Context Interfaces
- **Session Context Interface**: Main session context and expression context
- **Statement Context Interface**: Statement context for execution tracking

#### Expression System Interfaces
- **Expression Interface Hierarchy**: Base expression and scalar function interfaces
- **Expression Implementation Examples**: Arithmetic functions and column references

#### Schema and Metadata Interfaces
- **Information Schema Interface**: Metadata access and table operations
- **Table and Index Interfaces**: Table metadata and index operations

**Key Insights**: Understanding the interface contracts that enable modular development and testing.

---

### 🧪 [06. Testing Strategies and Framework](./06-testing-strategies.md)
**Comprehensive testing approaches and frameworks**

#### Testing Philosophy
- **Test Pyramid Strategy**: Unit, integration, and end-to-end test distribution
- **Testing Principles**: Fast feedback, deterministic results, comprehensive coverage

#### Test Organization and Structure
- **Directory Structure**: Test file organization and naming conventions
- **Unit Testing Framework**: TestKit framework and mock frameworks
- **Test Patterns and Examples**: Table-driven tests, property-based testing, error injection

#### Integration Testing
- **Integration Test Framework**: Component interaction verification
- **Real TiKV Integration Tests**: Distributed scenario testing

#### End-to-End Testing
- **System-Level Test Scenarios**: Full system workflow testing
- **Performance and Benchmark Testing**: Benchmark framework and regression testing

#### Fault Injection and Chaos Testing
- **Failpoint Framework**: Systematic fault injection
- **Chaos Engineering Tests**: Random failure and network partition testing

**Key Insights**: Multi-layered testing strategy ensuring reliability and correctness across distributed scenarios.

---

### ⚡ [07. Performance Optimization Guide](./07-performance-optimization.md)
**Strategies and techniques for optimizing TiDB performance**

#### Performance Analysis Methodology
- **Performance Monitoring Stack**: Metrics collection and analysis tools
- **Performance Profiling Workflow**: Systematic performance analysis approach
- **Key Performance Metrics**: Query execution and resource usage metrics

#### Query Optimization
- **Execution Plan Optimization**: Understanding TiDB's optimizer and cost model
- **Query Optimization Strategies**: Index optimization, join optimization, subquery optimization
- **Advanced Optimizer Hints**: Forcing specific algorithms and memory management
- **Vectorized Execution Optimization**: Batch processing for improved performance

#### Index Design and Optimization
- **Index Strategy Guidelines**: Primary key design and secondary index optimization
- **Index Maintenance and Statistics**: Statistics collection and index usage monitoring

#### Storage Layer Optimization
- **TiKV Optimization Strategies**: Region distribution and coprocessor optimization
- **Transaction Optimization**: Two-phase commit optimization and async commit

#### Memory Management
- **Memory Usage Optimization**: Memory tracking and control strategies
- **Spill-to-Disk Strategy**: Handling large operations with limited memory

#### Concurrency and Parallelism
- **Parallel Execution Optimization**: Join and aggregation parallelization
- **Lock Optimization**: Optimistic locking strategy and retry logic

**Key Insights**: Comprehensive performance optimization covering all layers from query planning to storage access.

---

## 🎯 Documentation Goals

This documentation serves multiple purposes:

### For New Contributors
- **Onboarding**: Understand TiDB's architecture and development practices
- **Learning Path**: Structured approach to understanding the codebase
- **Best Practices**: Established patterns and conventions

### For Experienced Developers
- **Deep Dive Reference**: Detailed implementation insights
- **Optimization Guidance**: Performance tuning strategies
- **API Documentation**: Interface contracts and usage patterns

### For System Architects
- **Design Patterns**: Architectural decisions and trade-offs
- **Scalability Insights**: Distributed systems design principles
- **Integration Guidance**: How components interact and extend

## 🔍 How to Use This Documentation

### Sequential Reading
For comprehensive understanding, read documents in order:
1. Start with **Architecture Overview** for system understanding
2. Explore **Core Components** for implementation details
3. Study **Module Interactions** for integration patterns
4. Use **Development Guide** for practical development
5. Reference **API Documentation** for specific interfaces
6. Apply **Testing Strategies** for quality assurance
7. Implement **Performance Optimization** for production readiness

### Reference Usage
For specific topics, jump directly to relevant sections:
- **Debugging Issues**: Development Guide → Debugging and Profiling
- **Performance Problems**: Performance Optimization Guide
- **Component Understanding**: Core Components → Specific Component
- **Interface Usage**: API and Interfaces Documentation
- **Testing New Features**: Testing Strategies and Framework

### Code Examples
Each document includes practical code examples:
- **Implementation patterns** with real TiDB code
- **Best practices** demonstrated through examples
- **Common pitfalls** and how to avoid them
- **Performance optimizations** with measurable improvements

## 🤝 Contributing to Documentation

This documentation is a living resource that grows with TiDB's development. Contributions are welcome:

### Documentation Updates
- **Accuracy**: Keep examples and APIs current
- **Completeness**: Add missing implementation details
- **Clarity**: Improve explanations and examples

### New Sections
- **Emerging Patterns**: Document new architectural patterns
- **Performance Insights**: Share optimization discoveries
- **Tool Integration**: Document development tool usage

### Feedback and Issues
- **Report Inaccuracies**: Help maintain documentation quality
- **Suggest Improvements**: Propose better explanations
- **Request Coverage**: Identify documentation gaps

## 📚 Additional Resources

### External References
- [TiDB Official Documentation](https://docs.pingcap.com/tidb/stable)
- [TiDB GitHub Repository](https://github.com/pingcap/tidb)
- [TiDB Design Documents](https://github.com/pingcap/tidb/tree/master/docs/design)

### Community Resources
- [TiDB Community](https://pingcap.com/community/)
- [TiDB Discord](https://discord.gg/KVRZBR2DrG)
- [TiDB Forum](https://asktug.com/)

### Development Tools
- [TiDB Dashboard](https://docs.pingcap.com/tidb/stable/dashboard-intro)
- [TiUP Playground](https://docs.pingcap.com/tidb/stable/tiup-playground)
- [Go Development Tools](https://golang.org/doc/editors.html)

---

This documentation provides a comprehensive foundation for understanding, developing, and optimizing TiDB. Whether you're contributing to TiDB core, building applications, or operating TiDB in production, these guides will help you navigate the complexity and leverage the full power of this distributed SQL database.

**Happy coding! 🚀**