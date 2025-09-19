# TiDB Development Getting Started Guide

This guide will help you set up a complete development environment for TiDB, from initial setup to running your first tests and making contributions.

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Development Environment Setup](#2-development-environment-setup)
3. [Building TiDB from Source](#3-building-tidb-from-source)
4. [Running Tests](#4-running-tests)
5. [Debugging Tips](#5-debugging-tips)
6. [Common Development Tasks](#6-common-development-tasks)
7. [IDE Setup](#7-ide-setup)
8. [Next Steps](#8-next-steps)

## 1. Prerequisites

### System Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **OS** | Linux/macOS | Ubuntu 20.04+, macOS 12+ |
| **CPU** | 4 cores | 8+ cores |
| **Memory** | 8GB | 16GB+ |
| **Disk** | 20GB free | 50GB+ free |
| **Go Version** | 1.21 | 1.21+ |

### Required Tools

Before starting, ensure you have these tools installed:

```bash
# Git (for source control)
git --version

# Make (for build automation)
make --version

# Go (programming language)
go version
```

## 2. Development Environment Setup

### 2.1 Install Go

#### On Ubuntu/Debian:
```bash
# Download and install Go
wget https://go.dev/dl/go1.21.0.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz

# Add to PATH
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

#### On macOS:
```bash
# Using Homebrew
brew install go

# Or download from https://golang.org/dl/
```

#### Verify Installation:
```bash
go version
# Expected output: go version go1.21.0 linux/amd64
```

### 2.2 Configure Go Environment

```bash
# Set GOPATH and GOROOT
export GOPATH=$HOME/go
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin

# Add to shell profile
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.bashrc
source ~/.bashrc

# Enable Go modules (default in Go 1.16+)
go env -w GO111MODULE=on
```

### 2.3 Install Development Dependencies

```bash
# Install build essentials
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    git \
    curl \
    cmake \
    pkg-config

# Install additional tools for TiDB development
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
go install golang.org/x/tools/cmd/goimports@latest
go install github.com/mgechev/revive@latest
```

## 3. Building TiDB from Source

### 3.1 Clone the Repository

```bash
# Clone TiDB repository
git clone https://github.com/pingcap/tidb.git
cd tidb

# Check repository structure
ls -la
# Expected: cmd/, pkg/, Makefile, go.mod, etc.
```

### 3.2 Understanding the Build System

TiDB uses a Makefile-based build system. Key targets include:

| Target | Purpose | Usage |
|--------|---------|-------|
| `make` | Build TiDB server | Default build |
| `make server` | Build server binary | Explicit server build |
| `make test` | Run unit tests | Testing |
| `make race` | Build with race detection | Debug builds |
| `make clean` | Clean build artifacts | Cleanup |

### 3.3 Initial Build

```bash
# Build TiDB server
make

# This will:
# 1. Download dependencies
# 2. Generate parser files
# 3. Compile the server binary
# 4. Create bin/tidb-server

# Verify build
ls -la bin/
# Expected: tidb-server binary
```

### 3.4 Build Configuration

#### Environment Variables:
```bash
# Enable race detection (for development)
export RACE_ENABLED=1
make

# Build with specific Go version
GO=/usr/local/go/bin/go make

# Build for different architecture
GOOS=darwin GOARCH=amd64 make
```

#### Build Options:
```bash
# Debug build with symbols
make TAGS=debug

# Static build (no external dependencies)
make static

# Build with specific version
make VERSION=v6.5.0
```

## 4. Running Tests

### 4.1 Test Categories

TiDB has several types of tests:

| Test Type | Command | Purpose | Duration |
|-----------|---------|---------|----------|
| **Unit Tests** | `make test` | Package-level testing | 5-15 minutes |
| **Race Tests** | `make test-race` | Concurrency issues | 10-30 minutes |
| **Integration Tests** | `make integration-test` | End-to-end testing | 30+ minutes |
| **Specific Package** | `go test ./pkg/parser/...` | Targeted testing | 1-5 minutes |

### 4.2 Running Unit Tests

```bash
# Run all unit tests
make test

# Run tests for specific package
go test ./pkg/session/
go test ./pkg/executor/
go test ./pkg/planner/core/

# Run tests with verbose output
go test -v ./pkg/parser/

# Run specific test function
go test -run TestSessionBasic ./pkg/session/

# Run tests with coverage
go test -cover ./pkg/planner/core/
```

### 4.3 Running Integration Tests

```bash
# Set up test environment
make integration-test

# Run specific integration test suites
cd tests
go test ./realtikvtest/...
go test ./readtest/...
```

### 4.4 Test Configuration

```bash
# Set test timeout
go test -timeout 30s ./pkg/parser/

# Run tests in parallel
go test -parallel 8 ./pkg/...

# Generate test coverage report
go test -coverprofile=coverage.out ./pkg/planner/core/
go tool cover -html=coverage.out -o coverage.html
```

## 5. Debugging Tips

### 5.1 Debugging with GDB

```bash
# Build with debug symbols
make TAGS=debug

# Run with GDB
gdb ./bin/tidb-server
(gdb) run --store=mocktikv
(gdb) break main.main
(gdb) continue
```

### 5.2 Using Delve (Go Debugger)

```bash
# Install Delve
go install github.com/go-delve/delve/cmd/dlv@latest

# Debug TiDB server
dlv exec ./bin/tidb-server -- --store=mocktikv

# Debug specific test
dlv test ./pkg/session/ -- -test.run TestSessionBasic
```

### 5.3 Debugging Configuration

```bash
# Enable debug logging
export TIDB_LOG_LEVEL=debug

# Enable SQL tracing
export TIDB_ENABLE_SLOW_LOG=true

# Enable optimizer trace
export TIDB_OPTIMIZER_TRACE=true
```

### 5.4 Common Debugging Scenarios

#### Debug Parser Issues:
```bash
# Test SQL parsing
go test -v ./pkg/parser/ -run TestParser

# Debug specific SQL
echo "SELECT * FROM t WHERE id = 1" | dlv test ./pkg/parser/
```

#### Debug Planner Issues:
```bash
# Enable planner debug traces
export TIDB_ENABLE_OPTIMIZER_TRACE=true

# Run with optimizer trace
go test -v ./pkg/planner/core/ -run TestPlanCache
```

#### Debug Executor Issues:
```bash
# Debug execution with verbose logging
go test -v ./pkg/executor/ -run TestSimpleSelect
```

## 6. Common Development Tasks

### 6.1 Adding a New Feature

#### Step 1: Understand the Component
```bash
# Explore the relevant package
find ./pkg -name "*.go" | grep -i "your_feature"

# Read existing tests
find ./pkg -name "*_test.go" | grep -i "your_feature"
```

#### Step 2: Create Development Branch
```bash
git checkout -b feature/add-your-feature
git push -u origin feature/add-your-feature
```

#### Step 3: Implement and Test
```bash
# Make changes
vim pkg/your_package/your_file.go

# Run relevant tests
go test ./pkg/your_package/

# Run full test suite
make test
```

### 6.2 Fixing a Bug

#### Step 1: Reproduce the Issue
```bash
# Create minimal test case
vim pkg/your_package/bug_test.go

# Verify bug exists
go test -run TestBugRepro ./pkg/your_package/
```

#### Step 2: Debug and Fix
```bash
# Use debugger to trace issue
dlv test ./pkg/your_package/ -- -test.run TestBugRepro

# Implement fix
vim pkg/your_package/your_file.go

# Verify fix
go test -run TestBugRepro ./pkg/your_package/
```

### 6.3 Code Quality Checks

```bash
# Format code
gofmt -w ./pkg/your_package/

# Organize imports
goimports -w ./pkg/your_package/

# Run linter
golangci-lint run ./pkg/your_package/

# Check for race conditions
go test -race ./pkg/your_package/
```

### 6.4 Performance Testing

```bash
# Run benchmarks
go test -bench=. ./pkg/your_package/

# Profile CPU usage
go test -cpuprofile=cpu.prof -bench=. ./pkg/your_package/
go tool pprof cpu.prof

# Profile memory usage
go test -memprofile=mem.prof -bench=. ./pkg/your_package/
go tool pprof mem.prof
```

## 7. IDE Setup

### 7.1 VS Code Configuration

Create `.vscode/settings.json`:
```json
{
    "go.toolsManagement.autoUpdate": true,
    "go.useLanguageServer": true,
    "go.lintTool": "golangci-lint",
    "go.formatTool": "goimports",
    "go.generateTestsFlags": [
        "-exported"
    ],
    "files.exclude": {
        "**/bin": true,
        "**/.git": true
    }
}
```

Install recommended extensions:
- Go (Google)
- GitLens
- Go Test Explorer

### 7.2 GoLand/IntelliJ Configuration

1. **Open Project**: File → Open → Select tidb directory
2. **Configure Go**: Settings → Go → GOROOT and GOPATH
3. **Enable Modules**: Settings → Go → Go Modules → Enable
4. **Set Build Tags**: Settings → Go → Build Tags & Vendoring

### 7.3 Vim/Neovim Configuration

Install vim-go and configure:
```vim
" .vimrc or init.vim
Plugin 'fatih/vim-go'

let g:go_fmt_command = "goimports"
let g:go_highlight_functions = 1
let g:go_highlight_methods = 1
let g:go_highlight_types = 1
```

## 8. Next Steps

### 8.1 Understanding the Codebase

Now that your environment is set up, continue with:

1. **[Architecture Analysis](architecture_analysis.md)** - Understand the overall system design
2. **[Planner and Optimizer](planner_and_optimizer.md)** - Deep dive into query processing
3. **[Best Practices](best_practices.md)** - Learn development standards

### 8.2 Contributing Your First Change

1. **Find an Issue**: Look for [good first issues](https://github.com/pingcap/tidb/issues?q=is%3Aopen+is%3Aissue+label%3A%22good+first+issue%22)
2. **Read Documentation**: Understand the relevant module
3. **Make Changes**: Follow our [Best Practices](best_practices.md)
4. **Submit PR**: Include tests and documentation

### 8.3 Staying Updated

```bash
# Keep your fork updated
git remote add upstream https://github.com/pingcap/tidb.git
git fetch upstream
git checkout master
git merge upstream/master

# Update dependencies
go mod download
go mod tidy
```

### 8.4 Development Tools

Install additional helpful tools:
```bash
# Code generation tools
go install github.com/golang/mock/mockgen@latest

# Documentation tools
go install golang.org/x/tools/cmd/godoc@latest

# Profiling tools
go install github.com/google/pprof@latest
```

## Troubleshooting Setup Issues

### Common Build Issues

**Issue**: `go: module github.com/pingcap/tidb: local imports not supported`
**Solution**: Ensure you're in the tidb directory and using Go modules

**Issue**: `make: *** [parser] Error 1`
**Solution**: Install yacc parser generator: `sudo apt-get install byacc`

**Issue**: `fatal error: 'stdlib.h' file not found`
**Solution**: Install build essentials: `sudo apt-get install build-essential`

### Performance Issues

**Issue**: Tests are running slowly
**Solution**: Increase parallelism: `go test -parallel 8`

**Issue**: Build takes too long
**Solution**: Use build cache: `export GOCACHE=/tmp/go-cache`

For more troubleshooting help, see our [Troubleshooting Guide](troubleshooting.md).

---

**Congratulations!** Your TiDB development environment is now ready. Start exploring the codebase with our [Architecture Analysis](architecture_analysis.md) guide!