# TiDB Development Setup Guide

This guide will help you set up a complete TiDB development environment, from building the project to running tests and debugging.

## 📋 Prerequisites

### System Requirements

**Supported Operating Systems:**
- Linux (Ubuntu 18.04+, CentOS 7+, etc.)
- macOS (10.15+)
- Windows (via WSL2)

**Hardware Requirements:**
- **CPU**: 4+ cores recommended
- **Memory**: 8GB+ RAM (16GB+ for development)
- **Storage**: 20GB+ free space
- **Network**: Internet access for dependencies

### Required Software

#### 1. Go Programming Language

TiDB requires Go 1.23.12 or later:

```bash
# Install Go (Linux/macOS)
wget https://go.dev/dl/go1.23.12.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.23.12.linux-amd64.tar.gz

# Add to PATH
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.bashrc
source ~/.bashrc

# Verify installation
go version
```

#### 2. Git Version Control

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install git

# CentOS/RHEL
sudo yum install git

# macOS (with Homebrew)
brew install git

# Verify installation
git --version
```

#### 3. Make Build Tool

```bash
# Ubuntu/Debian
sudo apt-get install build-essential

# CentOS/RHEL
sudo yum groupinstall "Development Tools"

# macOS (included in Xcode Command Line Tools)
xcode-select --install
```

#### 4. Development Tools (Optional but Recommended)

```bash
# Install additional development tools
# Ubuntu/Debian
sudo apt-get install curl wget vim htop

# Development utilities for Go
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
go install golang.org/x/tools/cmd/goimports@latest
go install github.com/go-delve/delve/cmd/dlv@latest
```

## 🔄 Clone and Build TiDB

### 1. Clone the Repository

```bash
# Clone TiDB repository
git clone https://github.com/pingcap/tidb.git
cd tidb

# Check out a specific version (optional)
# git checkout v7.5.0

# Check repository structure
ls -la
```

### 2. Build TiDB Server

```bash
# Build TiDB server binary
make

# This creates the tidb-server binary in ./bin/
ls -la bin/

# Alternative: Build with optimizations disabled for debugging
make server_debug

# Build specific components
make parser     # Build only the parser
make tools      # Build development tools
```

### 3. Verify Build

```bash
# Check TiDB version
./bin/tidb-server -V

# Should output something like:
# Release Version: v7.5.0-alpha
# Edition: Community
# Git Commit Hash: [commit-hash]
# Git Branch: master
# UTC Build Time: [build-time]
# GoVersion: go1.23.12
```

## 🎯 Development Environment Setup

### 1. IDE Configuration

#### Visual Studio Code Setup

```bash
# Install VS Code extensions
code --install-extension golang.Go
code --install-extension ms-vscode.vscode-go
code --install-extension ms-vscode.vscode-json

# Create workspace settings
mkdir .vscode
cat > .vscode/settings.json << 'EOF'
{
    "go.useLanguageServer": true,
    "go.languageServerExperimentalFeatures": {
        "diagnostics": true
    },
    "go.lintTool": "golangci-lint",
    "go.lintOnSave": "package",
    "go.formatTool": "goimports",
    "go.testFlags": ["-v", "-race"],
    "go.buildFlags": ["-race"],
    "files.associations": {
        "*.y": "yacc"
    }
}
EOF
```

#### GoLand/IntelliJ Setup

```bash
# Install GoLand from JetBrains
# Configure Go SDK path: /usr/local/go
# Set GOPATH: $HOME/go
# Enable Go modules support
```

### 2. Git Configuration

```bash
# Configure Git for TiDB development
git config user.name "Your Name"
git config user.email "your.email@example.com"

# Set up pre-commit hooks (optional)
cp hooks/pre-commit .git/hooks/
chmod +x .git/hooks/pre-commit

# Configure Git aliases for TiDB workflow
git config alias.co checkout
git config alias.br branch
git config alias.ci commit
git config alias.st status
git config alias.unstage 'reset HEAD --'
git config alias.last 'log -1 HEAD'
```

### 3. Development Dependencies

```bash
# Install additional Go tools
go install github.com/kisielk/errcheck@latest
go install honnef.co/go/tools/cmd/staticcheck@latest
go install github.com/mdempsky/unconvert@latest

# Install MySQL client for testing
# Ubuntu/Debian
sudo apt-get install mysql-client

# CentOS/RHEL
sudo yum install mysql

# macOS
brew install mysql-client
```

## 🗃️ Database Setup for Development

### 1. TiKV and PD Setup (for full cluster)

For complete TiDB development, you'll need TiKV and PD:

```bash
# Option 1: Use TiUP for easy cluster setup
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
source ~/.bashrc

# Start a local test cluster
tiup playground

# Option 2: Use docker-compose
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  pd:
    image: pingcap/pd:latest
    ports:
      - "2379:2379"
    command: ["--name=pd", "--data-dir=/pd-data", "--client-urls=http://0.0.0.0:2379", "--advertise-client-urls=http://pd:2379", "--peer-urls=http://0.0.0.0:2380", "--advertise-peer-urls=http://pd:2380", "--initial-cluster=pd=http://pd:2380"]
    
  tikv:
    image: pingcap/tikv:latest
    ports:
      - "20160:20160"
    command: ["--pd-endpoints=pd:2379", "--addr=0.0.0.0:20160", "--data-dir=/tikv-data"]
    depends_on:
      - pd
EOF

docker-compose up -d
```

### 2. MockTiKV for Unit Testing

For development and testing, you can use MockTiKV:

```bash
# MockTiKV is built into TiDB for testing
# No additional setup required
# Used automatically in unit tests
```

## 🧪 Running Tests

### 1. Unit Tests

```bash
# Run all unit tests
make test

# Run tests with race detection
make test-race

# Run specific package tests
go test ./pkg/parser/...
go test ./pkg/planner/core/...
go test ./pkg/executor/...

# Run with verbose output
go test -v ./pkg/session/...

# Run specific test function
go test -run TestSpecificFunction ./pkg/parser/

# Run tests with coverage
go test -cover ./pkg/...
go test -coverprofile=coverage.out ./pkg/...
go tool cover -html=coverage.out
```

### 2. Integration Tests

```bash
# Run integration tests (requires running cluster)
make integration_test

# Run MySQL compatibility tests
cd tests/integrationtest
go test ./...

# Run specific integration test suites
make test_part_1
make test_part_2
```

### 3. Benchmark Tests

```bash
# Run benchmark tests
go test -bench=. ./pkg/executor/
go test -bench=BenchmarkSpecific ./pkg/parser/

# Run with memory profiling
go test -bench=. -memprofile=mem.prof ./pkg/planner/core/
go tool pprof mem.prof
```

## 🔧 Development Workflow

### 1. Code Style and Linting

```bash
# Run linter
make check

# Fix formatting issues
make fmt

# Run specific linters
golangci-lint run

# Check imports
goimports -w .

# Check for ineffective assignments
ineffassign .
```

### 2. Building Components

```bash
# Build parser (regenerates from yacc)
make parser

# Build server with debug symbols
make server_debug

# Build tools
make tools

# Clean build artifacts
make clean

# Build with specific tags
go build -tags=debug ./cmd/tidb-server
```

### 3. Code Generation

```bash
# Generate code (if you modify .y files)
make parser

# Generate mock files
go generate ./...

# Update error codes
make errcheck
```

## 🐛 Debugging Setup

### 1. Debugging with Delve

```bash
# Install Delve debugger
go install github.com/go-delve/delve/cmd/dlv@latest

# Debug TiDB server
dlv exec ./bin/tidb-server

# Debug with arguments
dlv exec ./bin/tidb-server -- --store=mocktikv --log-level=debug

# Debug tests
dlv test ./pkg/session/
```

### 2. VS Code Debugging Configuration

Create `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug TiDB Server",
            "type": "go",
            "request": "launch",
            "mode": "exec",
            "program": "${workspaceFolder}/bin/tidb-server",
            "args": [
                "--store=mocktikv",
                "--log-level=debug"
            ],
            "cwd": "${workspaceFolder}",
            "env": {},
            "showLog": true
        },
        {
            "name": "Debug Current Test",
            "type": "go", 
            "request": "launch",
            "mode": "test",
            "program": "${workspaceFolder}",
            "args": [
                "-test.run",
                "TestFunctionName"
            ]
        }
    ]
}
```

### 3. Profiling Setup

```bash
# CPU profiling
go test -cpuprofile=cpu.prof ./pkg/executor/
go tool pprof cpu.prof

# Memory profiling  
go test -memprofile=mem.prof ./pkg/executor/
go tool pprof mem.prof

# HTTP profiling (add to main)
import _ "net/http/pprof"
go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()

# Access profiling data
go tool pprof http://localhost:6060/debug/pprof/profile
```

## 🚀 Running TiDB in Development

### 1. Start TiDB Server

```bash
# Start with MockTiKV (for development)
./bin/tidb-server --store=mocktikv --log-level=debug

# Start with real TiKV cluster
./bin/tidb-server --store=tikv --path="127.0.0.1:2379" --log-level=info

# Start with custom configuration
./bin/tidb-server --config=config.toml
```

### 2. Connect to TiDB

```bash
# Connect with MySQL client
mysql -h 127.0.0.1 -P 4000 -u root

# Run SQL commands
mysql> CREATE DATABASE testdb;
mysql> USE testdb;
mysql> CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(100));
mysql> INSERT INTO users VALUES (1, 'Alice'), (2, 'Bob');
mysql> SELECT * FROM users;
```

### 3. Development Configuration

Create `config.toml`:

```toml
[log]
level = "debug"
format = "text"
file.filename = "tidb.log"

[performance]
max-procs = 0
max-memory = 0
tcp-keep-alive = true
cross-join = true
run-auto-analyze = true

[tikv-client]
grpc-connection-count = 4
grpc-keepalive-time = 10
grpc-keepalive-timeout = 3
commit-timeout = "41s"

[txn-local-latches]
enabled = false
capacity = 2048000

[experimental]
allow-expression-index = false
```

## 📊 Monitoring and Observability

### 1. Enable Metrics

```bash
# Start TiDB with metrics endpoint
./bin/tidb-server --store=mocktikv --status-port=10080

# Access metrics
curl http://localhost:10080/metrics

# Access status information
curl http://localhost:10080/status
curl http://localhost:10080/info
```

### 2. Logging Configuration

```bash
# Configure logging in config.toml
[log]
level = "debug"           # debug, info, warn, error
format = "json"           # json, text
file.filename = "tidb.log"
file.max-size = 300       # MB
file.max-days = 0
file.max-backups = 0

# View logs
tail -f tidb.log | jq .   # For JSON format
tail -f tidb.log          # For text format
```

## ⚡ Performance Development Tips

### 1. Faster Builds

```bash
# Use build cache
export GOCACHE=$HOME/.cache/go-build

# Parallel builds
make -j$(nproc)

# Disable CGO for faster builds (when possible)
CGO_ENABLED=0 go build ./cmd/tidb-server

# Use go mod proxy
export GOPROXY=https://proxy.golang.org,direct
```

### 2. Faster Tests

```bash
# Run tests in parallel
go test -parallel 4 ./pkg/...

# Use test cache
go clean -testcache  # Clear when needed

# Skip long-running tests
go test -short ./pkg/...

# Run specific test files
go test ./pkg/executor/executor_test.go
```

## 🛠️ Common Development Tasks

### 1. Adding New SQL Functions

```bash
# 1. Add function definition in pkg/expression/builtin_*.go
# 2. Add tests in pkg/expression/builtin_*_test.go  
# 3. Update parser if needed (pkg/parser/parser.y)
# 4. Run tests: go test ./pkg/expression/

# Example workflow:
vim pkg/expression/builtin_string.go
vim pkg/expression/builtin_string_test.go
make test
```

### 2. Modifying Parser

```bash
# 1. Edit parser grammar
vim pkg/parser/parser.y

# 2. Regenerate parser
make parser

# 3. Update AST nodes if needed
vim pkg/parser/ast/*.go

# 4. Test changes
go test ./pkg/parser/
```

### 3. Performance Optimization

```bash
# 1. Identify bottleneck with profiling
go test -bench=. -cpuprofile=cpu.prof ./pkg/executor/

# 2. Analyze profile
go tool pprof cpu.prof

# 3. Make optimizations
vim pkg/executor/your_executor.go

# 4. Verify improvement
go test -bench=. ./pkg/executor/
```

## 🔍 Troubleshooting

### Common Build Issues

**Issue: Go version mismatch**
```bash
# Solution: Update Go version
go version  # Check current version
# Install Go 1.23.12+ as shown in prerequisites
```

**Issue: Missing dependencies**
```bash
# Solution: Download dependencies
go mod download
go mod tidy
```

**Issue: Parser build failure**
```bash
# Solution: Regenerate parser
make parser
# May need to install yacc/bison on some systems
```

### Common Runtime Issues

**Issue: Connection refused**
```bash
# Check if TiDB is running
ps aux | grep tidb-server

# Check ports
netstat -tlnp | grep 4000
```

**Issue: Permission denied**
```bash
# Fix binary permissions
chmod +x bin/tidb-server

# Check file ownership
ls -la bin/tidb-server
```

## 📚 Next Steps

After setting up your development environment:

1. **Read the [Code Walkthrough](walkthrough.md)** to understand the codebase structure
2. **Explore [Architecture Overview](../architecture/overview.md)** for system understanding
3. **Check [Contribution Guidelines](../guides/contributing.md)** before making changes
4. **Join the Community** via Discord, Slack, or GitHub Discussions

## 📖 Additional Resources

- [TiDB Development Guide](https://pingcap.github.io/tidb-dev-guide/)
- [Go Programming Language](https://golang.org/doc/)
- [TiDB Official Documentation](https://docs.pingcap.com/tidb/stable)
- [TiKV Documentation](https://tikv.org/docs/)
- [PD Documentation](https://docs.pingcap.com/tidb/stable/pd-configuration-file)