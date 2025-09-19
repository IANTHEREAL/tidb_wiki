# Contributing to TiDB

Welcome to the TiDB development community! This guide will help you make meaningful contributions to the TiDB project, from your first bug fix to major feature development.

## 🚀 Getting Started

### Prerequisites

Before contributing, ensure you have:

1. **Development Environment** - Follow our [Development Setup Guide](../development/setup.md)
2. **Code Understanding** - Read the [Code Walkthrough](../development/walkthrough.md)
3. **Architecture Knowledge** - Understand [TiDB Architecture](../architecture/overview.md)

### Your First Contribution

#### 1. Find an Issue

**Good First Issues:**
- Browse [good first issues](https://github.com/pingcap/tidb/issues?q=is%3Aopen+is%3Aissue+label%3A%22good+first+issue%22)
- Look for [help wanted](https://github.com/pingcap/tidb/issues?q=is%3Aopen+is%3Aissue+label%3A%22help+wanted%22) labels
- Check documentation improvements needed

**Issue Types:**
- 🐛 **Bug Fixes** - Fix incorrect behavior
- 📝 **Documentation** - Improve docs and examples  
- ✨ **Features** - Add new functionality
- 🔧 **Refactoring** - Improve code structure
- 🎯 **Performance** - Optimize existing code

#### 2. Claim the Issue

Comment on the issue:
```
I'd like to work on this issue. Could you assign it to me?
```

Wait for a maintainer to assign it to you before starting work.

### Development Workflow

#### 1. Fork and Clone

```bash
# Fork the repository on GitHub first, then:
git clone https://github.com/YOUR_USERNAME/tidb.git
cd tidb

# Add upstream remote
git remote add upstream https://github.com/pingcap/tidb.git

# Verify remotes
git remote -v
```

#### 2. Create a Feature Branch

```bash
# Update master branch
git checkout master
git pull upstream master

# Create feature branch
git checkout -b feature/your-feature-name

# Or for bug fixes
git checkout -b fix/issue-number-description
```

#### 3. Make Changes

Follow our [coding standards](#coding-standards) and make your changes:

```bash
# Make your changes
vim pkg/executor/your_file.go

# Add tests
vim pkg/executor/your_file_test.go

# Test your changes
make test
go test ./pkg/executor/...

# Check code quality
make check
```

#### 4. Commit Changes

Follow [conventional commit](https://www.conventionalcommits.org/) format:

```bash
# Stage your changes
git add .

# Commit with descriptive message
git commit -m "executor: add support for new SQL function REVERSE

- Add REVERSE function implementation in builtin_string.go
- Add comprehensive tests for various input types
- Update function registry and documentation

Fixes #12345"
```

**Commit Message Format:**
```
<component>: <brief description>

<detailed description>
- List of specific changes
- Why the change was needed
- Any breaking changes

Fixes #issue-number
```

**Components**: `executor`, `planner`, `parser`, `server`, `ddl`, `session`, `store`, etc.

#### 5. Push and Create PR

```bash
# Push to your fork
git push origin feature/your-feature-name

# Create Pull Request on GitHub
# Go to https://github.com/pingcap/tidb and click "New Pull Request"
```

## 📝 Pull Request Guidelines

### PR Title Format

Use the same format as commit messages:
```
executor: add support for new SQL function REVERSE
```

### PR Description Template

```markdown
## What problem does this PR solve?

Issue Number: Close #12345

## What is changed and how it works?

### Summary
Brief description of the changes.

### Implementation Details
- Added REVERSE function in `pkg/expression/builtin_string.go`
- Implemented `builtinReverseSig` with proper string reversal logic
- Added comprehensive test cases covering edge cases
- Updated function registry to include the new function

### Code Changes
- `pkg/expression/builtin_string.go`: Function implementation
- `pkg/expression/builtin_string_test.go`: Test cases
- `pkg/parser/parser.y`: Grammar updates (if needed)

## Test Plan

### Unit Tests
- Added tests for normal string reversal
- Added tests for empty string and null values
- Added tests for unicode characters

### Integration Tests
- Verified function works in SELECT statements
- Tested with complex expressions
- Checked compatibility with existing functions

## Side Effects

None expected. This is a pure addition without modifying existing behavior.

## Release Note

```
New SQL function REVERSE() added for string manipulation.
```

## Documentation

- [ ] Updated function documentation
- [ ] Added examples to user guide
- [ ] Updated SQL reference
```

### PR Checklist

Before submitting, verify:

- [ ] **Code Quality**
  - [ ] Code follows TiDB coding standards
  - [ ] All tests pass locally
  - [ ] No linting errors
  - [ ] Proper error handling

- [ ] **Testing**
  - [ ] Unit tests added/updated
  - [ ] Integration tests pass
  - [ ] Edge cases covered
  - [ ] Performance impact assessed

- [ ] **Documentation**
  - [ ] Code comments added
  - [ ] User documentation updated
  - [ ] API documentation updated

- [ ] **Compatibility**
  - [ ] No breaking changes (or properly documented)
  - [ ] MySQL compatibility maintained
  - [ ] Backward compatibility preserved

## 🎯 Coding Standards

### Code Style

#### General Principles

1. **Readability** - Code should be self-documenting
2. **Simplicity** - Prefer simple solutions over complex ones
3. **Consistency** - Follow existing patterns in the codebase
4. **Performance** - Consider performance implications
5. **Testability** - Write testable code

#### Go Coding Standards

```go
// ✓ Good: Use meaningful names
func buildExecutorTree(plan PhysicalPlan) (Executor, error) {
    switch p := plan.(type) {
    case *PhysicalTableScan:
        return buildTableScanner(p)
    case *PhysicalSelection:
        return buildSelectionExecutor(p)
    default:
        return nil, errors.Errorf("unsupported plan type: %T", plan)
    }
}

// ✗ Bad: Unclear names and no error context
func build(p interface{}) (interface{}, error) {
    switch x := p.(type) {
    case *PhysicalTableScan:
        return buildT(x)
    default:
        return nil, errors.New("error")
    }
}
```

#### Error Handling

```go
// ✓ Good: Wrap errors with context
func (e *TableReaderExecutor) Open(ctx context.Context) error {
    kvReq, err := e.buildKVReq(ctx)
    if err != nil {
        return errors.Trace(err)
    }
    
    result, err := e.SelectResult(ctx, e.ctx, kvReq, retTypes(e))
    if err != nil {
        return errors.Annotatef(err, "table reader failed to select result")
    }
    
    e.resultHandler.open(result)
    return nil
}

// ✗ Bad: Silent failures and lost context
func (e *TableReaderExecutor) Open(ctx context.Context) error {
    kvReq, err := e.buildKVReq(ctx)
    if err != nil {
        return err
    }
    
    result, err := e.SelectResult(ctx, e.ctx, kvReq, retTypes(e))
    if err != nil {
        log.Error("select failed") // Don't log and return error
        return err
    }
    
    e.resultHandler.open(result)
    return nil
}
```

#### Interface Design

```go
// ✓ Good: Small, focused interfaces
type Executor interface {
    Open(ctx context.Context) error
    Next(ctx context.Context, req *chunk.Chunk) error
    Close() error
    Schema() *expression.Schema
}

// ✗ Bad: Large interface with many responsibilities
type ExecutorManager interface {
    Open(ctx context.Context) error
    Next(ctx context.Context, req *chunk.Chunk) error
    Close() error
    Schema() *expression.Schema
    GetStats() *ExecutorStats
    SetMemoryTracker(*memory.Tracker)
    GetMemoryUsage() int64
    EnableParallelism(int)
    // ... many more methods
}
```

#### Comments

```go
// ✓ Good: Explain why, not what
// buildKeyRanges converts ranger.Range to kv.KeyRange for distributed execution.
// It handles infinite ranges and ensures proper key encoding for the storage layer.
func (e *TableReaderExecutor) buildKeyRanges() ([]kv.KeyRange, error) {
    ranges := make([]kv.KeyRange, 0, len(e.ranges))
    
    for _, ran := range e.ranges {
        // Handle infinite range case
        if ran.LowVal[0].Kind() == types.KindMinNotNull {
            // Use table prefix as start key
            startKey = tablecodec.EncodeTablePrefix(e.physicalTableID)
        }
        // ...
    }
    
    return ranges, nil
}

// ✗ Bad: Comments describe what code already shows
// buildKeyRanges builds key ranges
func (e *TableReaderExecutor) buildKeyRanges() ([]kv.KeyRange, error) {
    // Create empty ranges slice
    ranges := make([]kv.KeyRange, 0, len(e.ranges))
    
    // Loop through ranges
    for _, ran := range e.ranges {
        // Check if low value is min not null
        if ran.LowVal[0].Kind() == types.KindMinNotNull {
            // Set start key
            startKey = tablecodec.EncodeTablePrefix(e.physicalTableID)
        }
    }
    
    // Return ranges
    return ranges, nil
}
```

### Testing Standards

#### Unit Test Guidelines

```go
func TestReverseFunction(t *testing.T) {
    // Use table-driven tests
    tests := []struct {
        name     string
        input    interface{}
        expected interface{}
        hasError bool
    }{
        {
            name:     "normal string",
            input:    "hello",
            expected: "olleh",
            hasError: false,
        },
        {
            name:     "empty string",
            input:    "",
            expected: "",
            hasError: false,
        },
        {
            name:     "unicode string",
            input:    "你好",
            expected: "好你",
            hasError: false,
        },
        {
            name:     "null input",
            input:    nil,
            expected: nil,
            hasError: false,
        },
    }
    
    ctx := createContext(t)
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Create function with input
            f, err := newFunctionForTest(ctx, ast.Reverse, 
                primitiveValsToConstants(ctx, []interface{}{tt.input})...)
            require.NoError(t, err)
            
            // Evaluate function
            result, err := evalBuiltinFunc(f, chunk.Row{})
            
            if tt.hasError {
                require.Error(t, err)
                return
            }
            
            require.NoError(t, err)
            if tt.expected == nil {
                require.True(t, result.IsNull())
            } else {
                require.Equal(t, tt.expected, result.GetString())
            }
        })
    }
}
```

#### Integration Test Guidelines

```go
func TestQueryExecution(t *testing.T) {
    store, clean := testkit.CreateMockStore(t)
    defer clean()
    
    tk := testkit.NewTestKit(t, store)
    tk.MustExec("use test")
    tk.MustExec("create table t (id int primary key, name varchar(100))")
    tk.MustExec("insert into t values (1, 'hello'), (2, 'world')")
    
    // Test basic functionality
    result := tk.MustQuery("select id, reverse(name) from t order by id")
    result.Check(testkit.Rows("1 olleh", "2 dlrow"))
    
    // Test with complex expressions
    result = tk.MustQuery("select reverse(concat(name, '_test')) from t where id = 1")
    result.Check(testkit.Rows("tset_olleh"))
    
    // Test edge cases
    tk.MustExec("insert into t values (3, '')")
    result = tk.MustQuery("select reverse(name) from t where id = 3")
    result.Check(testkit.Rows(""))
}
```

## 🔄 Development Process

### Types of Contributions

#### 1. Bug Fixes

**Process:**
1. Reproduce the bug locally
2. Write a failing test that demonstrates the bug
3. Fix the bug with minimal changes
4. Ensure the test passes
5. Add regression tests if needed

**Example:**
```go
// Bug: REVERSE function doesn't handle unicode correctly

// 1. Add failing test
func TestReverseFunctionUnicode(t *testing.T) {
    // This test should fail before the fix
    testReverse(t, "你好", "好你")
}

// 2. Fix the implementation
func (b *builtinReverseSig) evalString(row chunk.Row) (string, bool, error) {
    str, isNull, err := b.args[0].EvalString(b.ctx, row)
    if isNull || err != nil {
        return "", isNull, err
    }
    
    // Fix: Use runes instead of bytes for proper unicode handling
    runes := []rune(str)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    
    return string(runes), false, nil
}
```

#### 2. New Features

**Process:**
1. Create RFC (Request for Comments) for large features
2. Get community feedback and approval
3. Break down into smaller, reviewable PRs
4. Implement incrementally
5. Add comprehensive tests and documentation

**RFC Template:**
```markdown
# RFC: Add REVERSE SQL Function

## Summary
Add REVERSE() function to TiDB for string manipulation, maintaining MySQL compatibility.

## Motivation
Users need string reversal functionality for data processing tasks.

## Design
- Add function to expression package
- Support all string types including unicode
- Return NULL for NULL input
- Maintain MySQL compatibility

## Implementation Plan
1. Add function signature and registration
2. Implement evaluation logic
3. Add comprehensive tests
4. Update documentation

## Compatibility
No breaking changes. Pure addition with MySQL compatibility.
```

#### 3. Performance Improvements

**Process:**
1. Identify bottleneck with profiling
2. Create benchmark tests
3. Implement optimization
4. Verify improvement with benchmarks
5. Ensure no functional regressions

**Example:**
```go
// Benchmark before optimization
func BenchmarkStringReverse(b *testing.B) {
    input := strings.Repeat("hello", 1000)
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        reverse(input)
    }
}

// Optimization: Use strings.Builder for large strings
func reverseOptimized(s string) string {
    runes := []rune(s)
    
    // Use pre-allocated slice for better performance
    result := make([]rune, len(runes))
    for i := range runes {
        result[len(runes)-1-i] = runes[i]
    }
    
    return string(result)
}
```

### Code Review Process

#### Submitting for Review

1. **Self-Review First**
   - Test all changes locally
   - Run the full test suite
   - Check code style and documentation
   - Review your own diff carefully

2. **Request Review**
   - Add appropriate reviewers
   - Provide context in PR description
   - Respond to feedback promptly
   - Make requested changes

#### Reviewing Code

When reviewing others' code:

1. **Be Constructive**
   - Focus on the code, not the person
   - Suggest improvements with examples
   - Explain the reasoning behind feedback

2. **Check for**
   - Correctness and logic errors
   - Performance implications
   - Test coverage
   - Documentation updates
   - Compatibility issues

**Good Review Comments:**
```
Consider using a strings.Builder here for better performance when 
concatenating many strings:

```go
var builder strings.Builder
for _, s := range strings {
    builder.WriteString(s)
}
return builder.String()
```

This avoids creating multiple intermediate string objects.
```

**Poor Review Comments:**
```
This is wrong. Fix it.
```

### Continuous Integration

#### Pre-commit Checks

Run these before pushing:

```bash
# Format code
make fmt

# Run linter
make check

# Run unit tests
make test

# Run integration tests
make integration_test

# Check MySQL compatibility
make mysql_compatibility_test
```

#### CI Pipeline

The CI pipeline runs:

1. **Code Quality Checks**
   - Linting (golangci-lint)
   - Formatting (gofmt)
   - Import ordering (goimports)
   - License headers

2. **Testing**
   - Unit tests with race detection
   - Integration tests
   - MySQL compatibility tests
   - Performance regression tests

3. **Build Verification**
   - Multiple Go versions
   - Multiple platforms
   - Docker image build

## 🏆 Recognition and Rewards

### Contribution Levels

#### First-Time Contributors
- Welcome message and mentorship
- Assignment of good first issues
- Guidance through the contribution process

#### Regular Contributors  
- Recognition in release notes
- Invitation to contributor Slack channels
- Early access to new features for testing

#### Core Contributors
- Commit access (after proven track record)
- Participation in architectural decisions
- Speaking opportunities at conferences

### Swag and Recognition

- **Contribution Swag**: Fill out the [contribution form](https://forms.pingcap.com/f/tidb-contribution-swag) to claim swag
- **Certificates**: Contribution certificates for significant contributions
- **Conference Speaking**: Opportunities to present at PingCAP events

## 🚨 Security Contributions

### Reporting Security Issues

**DO NOT** create public GitHub issues for security vulnerabilities.

Instead:
1. Email security@pingcap.com
2. Include detailed reproduction steps
3. Wait for acknowledgment before disclosure

### Security Development Guidelines

When working on security-related code:

1. **Input Validation**
   - Validate all user inputs
   - Use parameterized queries
   - Escape special characters

2. **Access Control**
   - Implement proper privilege checking
   - Follow principle of least privilege
   - Audit access control changes

3. **Data Protection**
   - Encrypt sensitive data
   - Use secure communication protocols
   - Implement proper key management

## 📞 Getting Help

### Communication Channels

#### Discord
- **General Discussion**: #general
- **Development**: #development  
- **Help**: #help

Join: [TiDB Discord](https://discord.gg/KVRZBR2DrG)

#### Slack
- **English**: [TiDB Community Slack](https://slack.tidb.io/invite?team=tidb-community&channel=everyone&ref=pingcap-tidb)
- **Japanese**: [TiDB Japan Slack](https://slack.tidb.io/invite?team=tidb-community&channel=tidb-japan&ref=github-tidb)

#### GitHub
- **Issues**: Bug reports and feature requests
- **Discussions**: Architecture and design discussions
- **Pull Requests**: Code review and collaboration

#### Forums
- **Chinese**: [AskTUG](https://asktug.com)
- **Stack Overflow**: [tidb tag](https://stackoverflow.com/questions/tagged/tidb)

### Mentorship Program

New contributors can request mentorship:

1. **Join Discord/Slack**
2. **Introduce yourself** in #newcomers
3. **Ask for a mentor** - experienced contributors will help
4. **Start with good first issues** - your mentor will guide you

### Office Hours

Regular office hours for contributors:
- **Time**: Every Wednesday 8:00 PM UTC
- **Where**: Discord voice channel
- **What**: Q&A, code review, architecture discussions

## 🎯 Advanced Contributions

### Becoming a Maintainer

Path to maintainership:

1. **Consistent Contributions** (6+ months)
   - Regular, high-quality PRs
   - Helpful code reviews
   - Community participation

2. **Technical Excellence**
   - Deep understanding of TiDB architecture
   - Ability to guide others
   - Leadership in technical decisions

3. **Community Leadership**
   - Help new contributors
   - Participate in discussions
   - Represent TiDB at events

### Creating RFCs

For major changes, create RFCs:

1. **When to Write RFCs**
   - New major features
   - Breaking changes
   - Architectural changes
   - API modifications

2. **RFC Process**
   - Create GitHub issue with RFC template
   - Gather community feedback
   - Iterate on design
   - Get approval from maintainers
   - Implement in phases

### Leading Special Interest Groups (SIGs)

TiDB has SIGs for different areas:
- **SIG-Planner**: Query optimization
- **SIG-Execution**: Query execution  
- **SIG-DDL**: Schema changes
- **SIG-Transaction**: Transaction processing

Consider leading or joining a SIG based on your interests and expertise.

## 🎉 Conclusion

Contributing to TiDB is a rewarding experience that helps build one of the world's most advanced distributed databases. Whether you're fixing a small bug or implementing a major feature, your contributions make TiDB better for everyone.

**Remember:**
- Start small and build up
- Ask questions when in doubt
- Be patient and persistent
- Help others along the way
- Have fun coding!

Welcome to the TiDB community! 🚀

## 📚 Additional Resources

- [Code Walkthrough](../development/walkthrough.md) - Understand the codebase
- [Architecture Overview](../architecture/overview.md) - System design
- [Design Patterns](../patterns/design-patterns.md) - Code patterns
- [Performance Guidelines](../patterns/performance.md) - Optimization tips
- [Development Setup](../development/setup.md) - Environment setup