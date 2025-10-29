# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenTofu is an open-source Infrastructure as Code (IaC) tool, forked from Terraform. It's built in Go and designed for building, changing, and versioning infrastructure safely and efficiently through a declarative configuration language.

**Current Version:** 1.12.0-dev
**License:** Mozilla Public License v2.0 (MPL-2.0)
**Go Version:** 1.25.3 (see `.go-version`)

## Development Commands

### Building
```bash
# Build the tofu binary (output to ./tofu)
go build -ldflags "-X main.version=$(git describe --tags --always --dirty)" -o tofu ./cmd/tofu

# Or use Make
make build
```

### Testing
```bash
# Run all tests
go test ./...
make test

# Run tests for a specific package
go test ./internal/command/...
go test ./internal/addrs

# Run a specific test
go test ./internal/lang/evalchecks/eval_for_each_test.go -test.run TestName

# Run tests with coverage
make test-with-coverage

# Integration tests (require cloud credentials)
make test-s3      # AWS S3 backend tests
make test-gcp     # GCP backend tests
make test-azure   # Azure backend tests
make test-pg      # PostgreSQL backend tests
```

### Code Quality
```bash
# Run linter (takes ~60 minutes)
make golangci-lint

# Check license compliance
make license-check

# Generate code (including go generate)
make generate

# Generate protobuf stubs
make protobuf
```

### Running OpenTofu
```bash
# After building
./tofu --version
./tofu init
./tofu plan
./tofu apply
```

## Architecture Overview

OpenTofu follows a layered architecture with clear separation of concerns:

### Core Request Flow
```
User Command → CLI (command package) → Backend → Local Executor → Context → Graph Builder → Graph Walker → Provider Plugins
```

### Key Packages

**`/cmd/tofu/`** - Main application entry point. Contains `main.go` and command routing in `commands.go`.

**`/internal/command/`** - CLI command implementations (apply, plan, init, etc.). Each command:
1. Parses command-line arguments and environment variables
2. Constructs a `backend.Operation` object
3. Passes operation to the backend for execution

**`/internal/backend/`** - Backend implementations that determine where state is stored:
- `local/` - Local state storage and operation execution
- `remote-state/` - Remote state backends (S3, Azure, GCP, Consul, PostgreSQL, etc.)
- `cloud/` - Cloud backend implementation
- Most backends delegate operation execution to `local.Local`

**`/internal/tofu/`** - Core engine. Contains `Context` which orchestrates operations:
- Graph construction via graph builders
- Graph walking for execution
- State management during operations
- Plan generation and application

**`/internal/configs/`** - Configuration parsing and validation. Handles HCL configuration files.

**`/internal/states/`** - State management and storage. Contains state representation and state managers.

**`/internal/plans/`** - Execution plan generation and management. Plans show what will change.

**`/internal/providers/`** - Provider plugin interface and management. Defines how OpenTofu communicates with provider plugins.

**`/internal/lang/`** - HCL language features, expressions, and function evaluation.

**`/internal/addrs/`** - Address parsing and handling for resources, modules, providers, etc.

**`/internal/dag/`** - Directed Acyclic Graph implementation for dependency management.

**`/internal/encryption/`** - State encryption functionality.

### Graph System

OpenTofu uses a graph-based execution model:
1. **Graph Builder** - Constructs a DAG of operations based on configuration and state
2. **Graph Walker** - Traverses the graph, executing operations in correct dependency order
3. **Resource Graph** - Enables parallel execution of independent resources

### Provider Plugin Architecture

OpenTofu uses HashiCorp's go-plugin for provider plugins:
- **`/internal/tfplugin5/`** - Protocol v5 definitions
- **`/internal/plugin6/`** - Protocol v6 implementations
- **`/internal/provider-simple-v6/`** - Simple provider example
- Providers run as separate processes communicating via gRPC

## Contributing Guidelines

### Important Constraints

1. **Issue-Driven Development**: Only work on issues with `accepted` and `help wanted` labels. Comment on the issue to get assigned before starting work.

2. **DCO Signing Required**: All commits must be signed with `git commit -s` to comply with the Developer Certificate of Origin. Write code yourself, avoid copy/paste from other projects.

3. **RFC Process**: Complex features require an RFC document in `/rfc/` before implementation.

4. **AI Coding Assistants**: The project asks contributors to disable AI coding assistants to ensure code is written by the contributor.

5. **Changelog Updates**: Update `CHANGELOG.md` with your changes.

### Workflow

1. Find an accepted, help-wanted issue
2. Comment to get assigned
3. Develop and test locally
4. Sign commits with `-s` flag
5. Update CHANGELOG.md
6. Submit PR using the provided template
7. Wait for maintainer review

### Code Organization Patterns

- **Internal Package Structure**: Code is organized under `/internal/` to prevent external imports
- **Co-located Tests**: Test files (`*_test.go`) live alongside source files
- **Test Fixtures**: Test data and fixtures stored near test files
- **Provider Interface**: Providers implement standard interfaces defined in `internal/providers/`

## Debugging

### Environment Variables
- `TF_LOG=trace` - Enable trace-level logging
- `TF_LOG_PATH=path` - Write logs to file
- Full list: https://opentofu.org/docs/cli/config/environment-variables/

### VSCode Launch Configuration
Create `.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "tofu plan",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${workspaceFolder}/cmd/tofu",
      "env": {"TF_LOG": "trace"},
      "args": ["plan"]
    },
    {
      "name": "test specific",
      "type": "go",
      "request": "launch",
      "mode": "test",
      "program": "${workspaceFolder}/internal/path/to/test.go",
      "args": ["-test.run", "TestName"]
    }
  ]
}
```

### Remote Debugging
Use `scripts/debug-opentofu` to run OpenTofu with dlv debugger on port 2345.

## Development Environment

### Prerequisites
- Go 1.25.3 (version from `.go-version`)
- Git
- (Optional) Docker/Podman for devcontainer support

### Devcontainer
VSCode and GoLand/IntelliJ support `.devcontainer.json` for containerized development.

## Common Patterns

### State Management
State is accessed through state managers (`internal/states/statemgr`) which provide:
- Lock/unlock for concurrent access protection
- Persistence to various backends
- State versioning

### Configuration Loading
1. Configuration parsed from HCL files in `internal/configs`
2. Validated against schema
3. References resolved
4. Variables interpolated

### Provider Communication
1. OpenTofu spawns provider plugin as subprocess
2. Communication via gRPC using protobuf-defined interfaces
3. Provider returns schemas, resources, data sources
4. CRUD operations sent to provider during apply

### Graph Construction
1. Start with configuration resources
2. Add state resources
3. Build dependency edges
4. Add provisioners and outputs
5. Validate graph (no cycles)
6. Walk graph in topological order

## CI/CD

GitHub Actions workflows in `.github/workflows/`:
- `build.yml` - Build verification
- `checks.yml` - Code quality (linting, formatting)
- `govulncheck.yml` - Security scanning
- `release.yml` - Release automation
- `nightly.yml` - Nightly builds
- `website.yml` - Documentation deployment

## Key Design Principles

1. **Declarative Configuration**: Users declare desired state, OpenTofu determines actions
2. **Graph-Based Execution**: Dependencies modeled as DAG for correctness and parallelism
3. **Plugin Architecture**: Extensibility via provider plugins
4. **State Management**: Current infrastructure state tracked separately from configuration
5. **Plan Before Apply**: Changes previewed before execution
6. **Backend Flexibility**: State storage decoupled from execution
