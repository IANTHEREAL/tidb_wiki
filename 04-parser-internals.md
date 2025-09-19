# TiDB Parser Internals

## Overview

TiDB's parser is a sophisticated, high-performance SQL parser built using yacc (Yet Another Compiler Compiler) that generates a LALR(1) bottom-up parser. It provides MySQL-compatible SQL parsing with excellent performance characteristics and extensibility. This document explores the internal implementation details of the parser components.

## Architecture Overview

```
SQL Text → Lexer (Scanner) → Tokens → Parser (yacc-generated) → AST
                ↓
        Character Stream → Token Stream → Parse Tree → Abstract Syntax Tree
```

### Key Components:
- **Lexer (`lexer.go`)**: Tokenizes SQL text into discrete tokens
- **Parser (`parser.y` → `parser.go`)**: Yacc grammar defining SQL syntax rules
- **Hint Parser (`hintparser.y`)**: Specialized parser for optimizer hints
- **Error Handling (`yy_parser.go`)**: Error recovery and reporting
- **Keywords (`keywords.go`)**: SQL keyword definitions and classification

## Lexical Analysis (Lexer)

### Scanner Structure (`lexer.go`)

```go
type Scanner struct {
    r   reader              // Character stream reader
    buf bytes.Buffer        // Token buffer for building strings
    
    client     charset.Encoding  // Client character encoding
    connection charset.Encoding  // Connection character encoding
    
    errs         []error    // Accumulated errors
    warns        []error    // Accumulated warnings
    stmtStartPos int        // Statement start position
    
    inBangComment bool      // Inside /*! ... */ block
    lastScanOffset int      // Last scan position
    // ... additional fields
}
```

#### Key Lexer Functions:

##### Main Tokenization (`Lex`)
```go
func (s *Scanner) Lex(v *yySymType) int {
    // Main entry point for token generation
    // Returns token type and sets token value in v
}
```

##### Token Type Detection
- **Identifiers**: Unquoted names (tables, columns, variables)
- **Quoted Identifiers**: Backtick-quoted names (\`table_name\`)
- **String Literals**: Single/double quoted strings
- **Numeric Literals**: Integers, decimals, floats, hex, binary
- **Keywords**: Reserved words and functions
- **Operators**: Arithmetic, comparison, logical operators
- **Delimiters**: Parentheses, commas, semicolons

##### String Scanning (`scanString`)
```go
func (s *Scanner) scanString() (tok int, pos Pos, lit string) {
    // Handles string literals with escape sequences
    // Supports both single (') and double (") quotes
    // Processes escape sequences like \n, \t, \\, \'
}
```

##### Quoted Identifier Scanning (`scanQuotedIdent`)
```go
func scanQuotedIdent(s *Scanner) (tok int, pos Pos, lit string) {
    // Processes backtick-quoted identifiers
    // Handles escaped backticks (``)
    // Returns quotedIdentifier token type
}
```

### Character Set Handling

The lexer supports multiple character encodings:
- **Client Encoding**: Character set used by client application
- **Connection Encoding**: Character set for the connection
- **Conversion Functions**: `convert2System`, `convert2Connection`

### Position Tracking
```go
type Pos struct {
    Line   int  // Line number (1-based)
    Col    int  // Column number (1-based) 
    Offset int  // Byte offset from start
}
```

## Syntax Analysis (Parser)

### Yacc Grammar (`parser.y`)

The parser uses a massive yacc grammar file (17,046 lines) defining the complete MySQL SQL syntax.

#### Grammar Structure:

##### Token Definitions
```yacc
%token <ident>
    identifier "identifier"
    quotedIdentifier "quoted identifier" 
    singleAtIdentifier "@identifier"
    doubleAtIdentifier "@@identifier"

%token
    add        "ADD"
    all        "ALL"
    alter      "ALTER"
    and        "AND"
    // ... 200+ more keywords
```

##### Union Type Definition
```yacc
%union {
    offset int              // Token position offset
    item interface{}        // Generic item storage
    ident string           // Identifier strings
    expr ast.ExprNode      // Expression nodes
    statement ast.StmtNode // Statement nodes
    // ... many more typed fields
}
```

##### Precedence Rules
```yacc
%left   or "OR"
%left   xor "XOR" 
%left   and "AND"
%right  not "NOT"
%left   between "BETWEEN"
%left   eq ne "=" "!=" "<>" "<=>" lt le gt ge "LIKE" "REGEXP"
// ... full operator precedence hierarchy
```

### Grammar Rules Structure

#### Top-Level Rule
```yacc
StatementList:
    Statement
    {
        if $1 != nil {
            s := $1
            if lexer, ok := yylex.(stmtTexter); ok {
                s.SetText(parser.lexer.client, lexer.stmtText())
            }
            parser.result = append(parser.result, s)
        }
    }
|   StatementList ';' Statement
    {
        if $3 != nil {
            s := $3
            if lexer, ok := yylex.(stmtTexter); ok {
                s.SetText(parser.lexer.client, lexer.stmtText())
            }
            parser.result = append(parser.result, s)
        }
    }
```

#### Statement Categories
- **DDL Statements**: CREATE, ALTER, DROP for schema operations
- **DML Statements**: SELECT, INSERT, UPDATE, DELETE for data operations
- **Utility Statements**: SHOW, DESCRIBE, EXPLAIN for metadata
- **Transaction Statements**: BEGIN, COMMIT, ROLLBACK
- **Administrative Statements**: SET, USE, KILL

#### Expression Grammar
```yacc
Expression:
    singleAtIdentifier
|   Expression '+' Expression 
|   Expression '-' Expression
|   Expression '*' Expression
|   Expression '/' Expression
|   Expression '%' Expression
|   Expression 'DIV' Expression
|   Expression 'MOD' Expression
// ... comprehensive expression rules
```

### Parser Generation Process

#### Build Process (`Makefile`)
```makefile
parser.go: parser.y
    goyacc -o parser.go -p yy parser.y
    @sed -i 's/type yySymType/type yySymType/g' parser.go
```

#### Generated Parser Components:
- **Parse Tables**: LALR(1) parsing tables with states and transitions
- **Action Functions**: Code generation for grammar rule reductions
- **Error Recovery**: Automatic error recovery mechanisms
- **Debug Support**: Optional debugging output for parser development

## Optimizer Hints Parser (`hintparser.y`)

### Specialized Hint Grammar

TiDB supports MySQL-style optimizer hints in comments:
```sql
SELECT /*+ USE_INDEX(t1, idx1) IGNORE_INDEX(t2, idx2) */ * FROM t1, t2;
```

#### Hint Types Supported:
- **Index Hints**: `USE_INDEX`, `IGNORE_INDEX`, `FORCE_INDEX`
- **Join Hints**: `USE_NL`, `USE_HASH`, `USE_MERGE`
- **Scan Hints**: `USE_TOKUDB`, `USE_INDEX_MERGE`
- **Memory Hints**: `MEMORY_QUOTA`, `MAX_EXECUTION_TIME`

#### Hint Grammar Structure:
```yacc
HintTableList:
    HintTable
|   HintTableList ',' HintTable

HintTable:
    hintIdentifier
    {
        $$ = ast.HintTable{TableName: ast.NewCIStr($1)}
    }
|   hintIdentifier '.' hintIdentifier  
    {
        $$ = ast.HintTable{
            DBName: ast.NewCIStr($1),
            TableName: ast.NewCIStr($3),
        }
    }
```

## Keyword Management (`keywords.go`)

### Keyword Classification

Keywords are classified into categories for parsing precedence:

```go
var (
    // Reserved keywords that cannot be used as identifiers
    reservedKeywords = []string{
        "ADD", "ALL", "ALTER", "AND", "AS", "ASC", "BETWEEN", "BY",
        "CASE", "CREATE", "DELETE", "DESC", "DISTINCT", "DROP", "ELSE",
        "FROM", "GROUP", "HAVING", "IF", "IN", "INSERT", "INTO", "IS",
        "JOIN", "KEY", "LIMIT", "NOT", "NULL", "ON", "OR", "ORDER",
        "SELECT", "SET", "TABLE", "THEN", "TO", "UNION", "UPDATE",
        "VALUES", "WHEN", "WHERE", "WITH",
    }
    
    // Non-reserved keywords that can be used as identifiers in some contexts
    nonReservedKeywords = []string{
        "ACTION", "ADMIN", "AFTER", "ALGORITHM", "AUTO_INCREMENT",
        "BEGIN", "BIGINT", "BINARY", "BIT", "BOOL", "BOOLEAN",
        // ... many more
    }
    
    // Function names that can be used as identifiers
    functionKeywords = []string{
        "ABS", "ACOS", "ADDDATE", "ADDTIME", "AES_DECRYPT", "AES_ENCRYPT",
        "ASCII", "ASIN", "ATAN", "ATAN2", "AVG", "BENCHMARK",
        // ... comprehensive function list
    }
)
```

### Keyword Token Generation
```go
func (s *Scanner) handleIdent(lit string) int {
    // Check if identifier is a keyword
    if tok := s.keywordToken(lit); tok != 0 {
        return tok
    }
    return identifier
}
```

## Error Handling and Recovery

### Error Types (`yy_parser.go`)

```go
var (
    // Parse-time errors
    ErrSyntax = terror.ClassParser.NewStd(mysql.ErrSyntax)
    ErrParse = terror.ClassParser.NewStd(mysql.ErrParse)
    
    // Semantic errors
    ErrUnknownCharacterSet = terror.ClassParser.NewStd(mysql.ErrUnknownCharacterSet)
    ErrInvalidYearColumnLength = terror.ClassParser.NewStd(mysql.ErrInvalidYearColumnLength)
    ErrWrongArguments = terror.ClassParser.NewStd(mysql.ErrWrongArguments)
)
```

### Error Recovery Mechanisms:
1. **Panic Mode Recovery**: Skip tokens until synchronization point
2. **Phrase Level Recovery**: Local error correction
3. **Error Productions**: Grammar rules specifically for error handling
4. **Context-Sensitive Messages**: Meaningful error descriptions

### Warning System:
- **Deprecation Warnings**: For obsolete syntax features
- **Compatibility Warnings**: For MySQL version differences
- **Best Practice Warnings**: For non-optimal query patterns

## Performance Characteristics

### Lexer Performance:
- **O(n) Time Complexity**: Linear scan of input
- **Minimal Backtracking**: Deterministic finite automaton
- **Efficient Buffer Management**: Reused byte buffers
- **Zero-Copy Operations**: Where possible, avoid string copying

### Parser Performance:
- **LALR(1) Parsing**: O(n) time complexity
- **State Machine**: Highly optimized state transitions
- **Memory Efficient**: Minimal stack usage for recursion
- **Incremental Parsing**: Support for parsing fragments

### Memory Management:
- **Object Pooling**: Reuse of parser objects
- **Minimal Allocations**: Careful memory allocation patterns
- **AST Node Reuse**: Where semantically safe
- **Garbage Collection Friendly**: Minimal long-lived references

## Integration and Usage

### Parser Interface:
```go
type Parser struct {
    lexer Scanner
    result []ast.StmtNode
    // ... internal state
}

func New() *Parser {
    // Create new parser instance
}

func (parser *Parser) Parse(sql, charset, collation string) ([]ast.StmtNode, []error, error) {
    // Main parsing entry point
}
```

### Integration Points:
- **Session Layer**: Called from session execution
- **Prepared Statements**: Cached parse results
- **EXPLAIN**: Parse without execution
- **Syntax Validation**: Parse-only mode

### Thread Safety:
- **Parser Instances**: Not thread-safe, use per-session
- **AST Nodes**: Immutable after creation
- **Shared Data**: Read-only keyword tables and grammar

## Extensibility

### Adding New Syntax:
1. **Update Grammar**: Add rules to `parser.y`
2. **Add Keywords**: Update `keywords.go` if needed
3. **Create AST Nodes**: Define new node types in `ast/`
4. **Implement Semantics**: Add execution logic
5. **Add Tests**: Comprehensive test coverage

### Example Extension Process:
```yacc
// Add new statement type
NewStmt:
    "MYNEW" "STATEMENT" Identifier
    {
        $$ = &ast.MyNewStmt{Name: $3}
    }
```

## Summary

TiDB's parser represents a sophisticated implementation of SQL parsing that achieves:

- **High Performance**: LALR(1) parsing with O(n) complexity
- **MySQL Compatibility**: Comprehensive syntax support
- **Extensibility**: Clean architecture for adding new features  
- **Robustness**: Excellent error handling and recovery
- **Maintainability**: Clear separation of lexical and syntactic analysis

The combination of yacc-generated parsing tables, hand-optimized lexer, and comprehensive test coverage makes this one of the most capable SQL parsers in the open-source ecosystem.