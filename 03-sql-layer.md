# TiDB SQL Layer Architecture

## Overview

TiDB's SQL layer is responsible for parsing, planning, and executing SQL statements. It transforms SQL text into optimized execution plans and coordinates with the storage layer to retrieve and manipulate data. The SQL layer consists of several interconnected components that work together to provide MySQL-compatible SQL processing.

## 1. Parser Architecture (`pkg/parser`)

### Core Components

The TiDB parser is a standalone, MySQL-compatible SQL parser built using yacc (Yet Another Compiler Compiler) that generates a bottom-up parser with excellent performance.

#### Key Files:
- **`parser.y`** - Main yacc grammar file (405KB, ~12,000 lines)
- **`parser.go`** - Generated parser implementation (7.6MB)
- **`lexer.go`** - Lexical analyzer for tokenization
- **`hintparser.y`** - Optimizer hint parsing grammar
- **`keywords.go`** - SQL keyword definitions and handling

#### Parser Structure:
```
SQL Text → Lexer → Tokens → Parser → AST → Semantic Analysis
```

### Features:
- **MySQL Compatibility**: Nearly complete MySQL 8.0 syntax support
- **High Performance**: Bottom-up parsing with state machine efficiency
- **Extensible**: Adding new syntax requires minimal yacc and Go code changes
- **Error Recovery**: Robust error handling and reporting
- **Hint Support**: Optimizer hints for query tuning

## 2. SQL Statement Processing Flow

### Processing Pipeline:
```
1. SQL Text Input
2. Lexical Analysis (Tokenization)
3. Syntax Analysis (Parsing to AST)
4. Semantic Analysis (Type checking, name resolution)
5. Logical Plan Building
6. Physical Plan Optimization
7. Execution Plan Generation
8. Statement Execution
```

### Key Components in Processing:

#### Session Layer (`pkg/session`)
- Entry point for SQL execution
- Connection and session management
- Transaction coordination
- Statement preparation and caching

#### Plan Builder (`pkg/planner/core/planbuilder.go`)
- Converts AST to logical plans
- Handles name resolution and type checking
- Implements SQL semantic rules
- Manages query context and metadata

#### Executor (`pkg/executor/`)
- Executes physical plans
- Coordinates with storage layer
- Implements execution operators
- Handles result set generation

## 3. AST Structure and Transformation

### AST Node Hierarchy

The AST is built around a flexible node interface system:

```go
type Node interface {
    Restore(ctx *format.RestoreCtx) error  // SQL regeneration
    Accept(v Visitor) (node Node, ok bool) // Visitor pattern
    Text() string                          // UTF-8 text
    OriginalText() string                  // Original text
    SetText(enc charset.Encoding, text string)
    SetOriginTextPosition(offset int)
    OriginTextPosition() int
}
```

#### Node Categories:

##### Statement Nodes (`ast/`)
- **`StmtNode`** - Base interface for all statements
- **`DDLNode`** - Data Definition Language statements
- **`DMLNode`** - Data Manipulation Language statements

##### Expression Nodes
- **`ExprNode`** - Base interface for expressions
- **Arithmetic, logical, comparison operators**
- **Function calls and subqueries**
- **Constants and column references**

##### Structural Nodes
- **Table and column references**
- **JOIN clauses and conditions**
- **ORDER BY, GROUP BY, HAVING clauses**
- **Subqueries and CTEs**

### AST Transformation Process:

1. **Parsing Phase**: Raw SQL → AST nodes
2. **Validation Phase**: Type checking, name resolution
3. **Optimization Phase**: AST rewriting for optimization
4. **Plan Generation**: AST → Logical plans → Physical plans

### Visitor Pattern Implementation:
```go
type Visitor interface {
    Enter(n Node) (node Node, skipChildren bool)
    Leave(n Node) (node Node, ok bool)
}
```

## 4. DDL (Data Definition Language) Handling

### DDL Statement Types (`pkg/parser/ast/ddl.go`)

#### Database Operations:
- **`CreateDatabaseStmt`** - CREATE DATABASE
- **`DropDatabaseStmt`** - DROP DATABASE  
- **`FlashBackDatabaseStmt`** - FLASHBACK DATABASE

#### Table Operations:
- **`CreateTableStmt`** - CREATE TABLE with all options
- **`AlterTableStmt`** - ALTER TABLE modifications
- **`DropTableStmt`** - DROP TABLE
- **`TruncateTableStmt`** - TRUNCATE TABLE
- **`RenameTableStmt`** - RENAME TABLE

#### Index Operations:
- **`CreateIndexStmt`** - CREATE INDEX
- **`DropIndexStmt`** - DROP INDEX

#### Advanced Features:
- **`CreateViewStmt`** - CREATE VIEW
- **`CreateSequenceStmt`** - CREATE SEQUENCE
- **`CreatePlacementPolicyStmt`** - Placement policies
- **`CreateResourceGroupStmt`** - Resource groups

### DDL Execution Pipeline (`pkg/executor/ddl.go`, `pkg/ddl/`)

```
DDL AST → DDL Executor → DDL Worker → Schema Change → Storage Layer
```

#### Key Components:
1. **DDL Executor**: Coordinates DDL execution
2. **DDL Worker**: Handles background schema changes
3. **Schema State Management**: Ensures consistency during changes
4. **Online DDL**: Minimizes downtime for schema modifications

## 5. DML (Data Manipulation Language) Processing

### DML Statement Types (`pkg/parser/ast/dml.go`)

#### Core DML Operations:
- **`SelectStmt`** - SELECT queries with all clauses
- **`InsertStmt`** - INSERT statements (VALUES, SELECT)
- **`UpdateStmt`** - UPDATE statements with JOINs
- **`DeleteStmt`** - DELETE statements with JOINs

#### Advanced DML:
- **`LoadDataStmt`** - LOAD DATA INFILE
- **`ImportIntoStmt`** - IMPORT INTO (TiDB extension)
- **`CallStmt`** - Stored procedure calls
- **`ShowStmt`** - SHOW commands
- **`SplitRegionStmt`** - Region management

### DML Execution Architecture

#### Executor Types (`pkg/executor/`):
- **`SelectExecutor`** - SELECT statement execution
- **`InsertExecutor`** - INSERT processing with duplicate handling
- **`UpdateExecutor`** - UPDATE with join and subquery support
- **`DeleteExecutor`** - DELETE with cascading and constraints

#### Execution Operators:
- **Table Readers**: Point get, batch point get, table scan
- **Index Readers**: Index scan, index lookup
- **Join Operators**: Hash join, merge join, nested loop join
- **Aggregation**: Hash aggregation, stream aggregation
- **Sort/Limit**: Top-N, sort, limit operators

### Query Optimization Process:

1. **Logical Optimization**:
   - Predicate pushdown
   - Column pruning
   - Constant folding
   - Join reordering

2. **Physical Optimization**:
   - Access path selection
   - Join algorithm selection
   - Index selection
   - Cost-based optimization

3. **Execution Plan Selection**:
   - Cost comparison
   - Statistics-driven decisions
   - Runtime adaptation

## Integration Points

### Parser Integration:
```
SQL Text → Scanner (Lexer) → Parser → AST → Plan Builder
```

### AST to Execution:
```
AST → Logical Plan → Physical Plan → Execution Tree → Results
```

### Error Handling:
- **Parse Errors**: Syntax and lexical errors
- **Semantic Errors**: Type mismatches, undefined objects
- **Execution Errors**: Runtime constraints, data errors

### Performance Characteristics:
- **Parser**: O(n) complexity, highly optimized state machine
- **AST Operations**: Visitor pattern enables efficient traversals
- **Memory Management**: Careful allocation and reuse patterns
- **Concurrency**: Thread-safe parsing with session isolation

## Summary

TiDB's SQL layer provides a robust, MySQL-compatible SQL processing engine with:

- **High-performance parsing** using yacc-generated parsers
- **Comprehensive AST model** supporting full MySQL syntax
- **Flexible optimization framework** with rule and cost-based optimization
- **Scalable execution engine** with distributed processing capabilities
- **Online DDL capabilities** for production schema changes
- **Advanced DML features** including complex joins and subqueries

This architecture enables TiDB to serve as a drop-in replacement for MySQL while providing horizontal scalability and distributed transaction support.