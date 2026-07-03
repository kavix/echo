# Contributing to Eko ✦

Thank you for your interest in contributing to Eko! Eko is an AI-powered snapshot versioning CLI designed to help capture and restore project states.

This guide provides a comprehensive overview of the architecture, internal modules, database design, concurrency patterns, and conventions to help you get started as a contributor.

---

## 1. Directory Structure & Architecture

Eko is written entirely in Go as a structured command-line application using the Cobra library. Below is an overview of the core file structure:

```
eko/
├── cmd/              # CLI Command Layer (Cobra CLI subcommands)
│   ├── commands_test.go   # Integration tests for all CLI commands
│   ├── history.go    # "eko history" command
│   ├── init.go       # "eko init" command
│   ├── project.go     # Initialization and check guards
│   ├── project_test.go# Initializer guards unit tests
│   ├── restore.go     # "eko restore" command
│   ├── root.go        # Cobra Root command
│   └── save.go        # "eko save" command
├── internal/
│   ├── api/          # Directory Comparison & Diffing
│   │   ├── diff.go        # Walks snapshots and builds before/after file pairs
│   │   └── diff_test.go   # Tests for diff calculations
│   ├── db/           # SQLite Persistence Layer
│   │   ├── db.go          # Database setup and schema initialization
│   │   └── db_test.go     # SQLite operation unit tests
│   ├── snapshot/     # Core Snapshot Engine
│   │   ├── snapshot.go    # High-level snapshot creation and restoration
│   │   └── snapshot_test.go # Tests for snapshot engines
│   └── util/         # Concurrency and OS Utilities
│       ├── fs.go          # Concurrent file copy utilities
│       └── fs_test.go     # Concurrent file copy unit tests
├── .github/          # GitHub Workflows and Templates
├── Dockerfile        # Containerized runtime configuration
├── go.mod            # Go module definitions
└── main.go           # CLI Application Entry Point
```

---

## 2. Core Engine Internals & Concurrency

Eko is designed to be extremely fast by leveraging Go's concurrency primitives. Contributing to the engine requires an understanding of these models:

### A. Concurrent Copying (Worker Pool)
Located in `internal/util/fs.go`, directory copying uses a worker pool model:
1. **Directory Tree Walk**: The source tree is walked serially. Directories are created immediately in the destination path to guarantee parent paths exist before workers write files, preventing race conditions.
2. **Worker Pool**: A pool of `runtime.NumCPU()` goroutines is spawned. File copy tasks are fed into a buffered `tasks` channel.
3. **Thread-Safe Errors**: The first error encountered by any worker is communicated back through an `errs` channel, and the walk is safely aborted.

### B. Parallel Deletion (Lock-Free CAS)
Located in `internal/snapshot/snapshot.go`, when restoring a snapshot, existing workspace files are deleted concurrently:
1. **Parallel Delete**: Eko spawns a goroutine for each top-level entry in the directory (excluding ignored files like `.eko` and `.git`).
2. **Compare-And-Swap (CAS)**: To record the first error thread-safely without mutex locks, it utilizes atomic CAS operations on an `unsafe.Pointer`:
   ```go
   atomic.CompareAndSwapPointer(&firstErr, nil, unsafe.Pointer(&errVal))
   ```

---

## 3. Database Schema

All snapshot metadata is stored in a local SQLite database at `.eko/db.sqlite`. It consists of a single `snapshots` table:

```sql
CREATE TABLE IF NOT EXISTS snapshots (
    id TEXT PRIMARY KEY,
    message TEXT,
    path TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

- **`id`**: A unique 8-character random hexadecimal identifier generated using cryptographically secure random bytes.
- **`message`**: A short log message associated with the snapshot.
- **`path`**: The file path to the snapshot's storage directory (e.g. `.eko/snapshots/<id>`).
- **`created_at`**: Creation timestamp.

---

## 4. Development Workflow

### Setup
1. Fork the repository and clone it locally:
   ```bash
   git clone https://github.com/<your-username>/eko.git
   cd eko
   ```
2. Build the project:
   ```bash
   go build -o eko .
   ```

### Running Tests
Make sure to run the test suite and verify everything passes before submitting changes:
```bash
go test -v ./...
```

### Coding Standards
* **Error Handling**: Always return errors up the stack instead of calling `panic` or `log.Fatal` inside packages.
* **Formatting**: Run `gofmt -s -w` on all Go files.
* **Imports**: Use standard grouping for imports: standard library first, third-party libraries second, local packages third.

---

## 5. Submitting a Pull Request

1. Create a feature branch:
   ```bash
   git checkout -b feat/your-feature-name
   ```
2. Commit your changes with clear, semantic commit messages (e.g., `feat: add custom snapshot descriptions`).
3. Push to your fork and open a Pull Request. Ensure that the PR template is filled out completely.
