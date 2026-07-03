# Eko ✦

[![Go Report Card](https://goreportcard.com/badge/github.com/kavix/eko)](https://goreportcard.com/report/github.com/kavix/eko)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**Eko** is an AI-powered, high-performance snapshot versioning command-line interface (CLI) written in Go. It allows developers to capture, compare, and restore directory states concurrently. Think of it as a lightning-fast "Time Machine" designed for local development.

---

## Features

- **Concurrent Snapshots:** Instantly capture filesystem states using a worker-pool concurrency model.
- **Atomic Restores:** Revert workspaces cleanly with safety guards and thread-safe error recovery.
- **Metadata Management:** Structured storage of snapshots and logs in a local, self-contained SQLite database.
- **Diff Comparison:** Efficiently compare changes between snapshot points.

---

## Getting Started

### Prerequisites

- **Go**: Version 1.21+ is required to build from source.

### Installation

#### 1. Homebrew (macOS)
```bash
brew tap kavix/tap
brew install eko
```

#### 2. Build From Source
Clone the repository and compile the binary:
```bash
git clone https://github.com/kavix/eko.git
cd eko
go build -o eko .
```

---

## CLI Usage Guide

### 1. Initialize Eko
Initialize a new Eko project in the current working directory. This creates the local database and backup storage inside the `.eko` folder:
```bash
eko init
```

### 2. Save a Snapshot
Capture the current filesystem state (automatically ignores files matching patterns like `.git`, `node_modules`, etc.):
```bash
eko save
```

### 3. View Snapshot Log
List all recorded snapshots along with their unique identifiers and creation timestamps:
```bash
eko history
```

### 4. Restore a State
Revert the current directory to the exact state of any past snapshot:
```bash
eko restore <snapshot-id>
```

---

## Contributing

We welcome contributions to Eko! Please review our [Contributing Guide](CONTRIBUTING.md) to understand the codebase architecture, concurrency engine, database layout, and contribution requirements.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
