---
sidebar_position: 4
---

# CLI Reference Manual

Exhaustive reference guide for all commands, flags, options, and aliases in the **Eko** command line interface.

---

## 📋 Command Matrix

| Command | Aliases | Description | Key Flags |
| ------- | ------- | ----------- | --------- |
| [`eko init`](#1-eko-init) | None | Initialize Eko project & SQLite database | None |
| [`eko save`](#2-eko-save) | None | Capture project snapshot in CAS object store | `-m`, `-a/--ai`, `--provider` |
| [`eko summary`](#3-eko-summary-alias-eko-summarize) | `summarize` | Generate AI-powered change summary | `-j/--json`, `-p/--provider`, `-s/--save` |
| [`eko history`](#4-eko-history) | None | List snapshot history | `-j/--json`, `-v/--verbose` |
| [`eko restore`](#5-eko-restore-snapshot-id) | None | Revert project to a past snapshot state | `<snapshot-id>`, `<tag-name>` |
| [`eko tag`](#6-eko-tag) | None | Assign human-readable tag/alias to a snapshot | `<snapshot-id>`, `<tag-name>` |
| [`eko clean`](#7-eko-clean) | None | Remove old snapshots & garbage-collect blobs | `--keep`, `--dry-run` |
| [`eko migrate`](#8-eko-migrate) | None | Convert legacy snapshots to CAS format | `--dry-run` |
| [`eko ai status`](#9-eko-ai-status) | None | Intent-based workspace status & file role analysis | None |
| [`eko ai review`](#10-eko-ai-review) | None | Automated code review & commit risk scoring | None |
| [`eko ai semdiff`](#11-eko-ai-semdiff) | None | Behavioral semantic diff analysis | None |
| [`eko ai risk`](#12-eko-ai-risk) | None | Multi-dimensional commit risk evaluation | None |
| [`eko ai impact`](#13-eko-ai-impact) | None | Subsystem change impact & test suite match | None |
| [`eko ai bisect`](#14-eko-ai-bisect) | None | Automated AI regression bug isolation | `[failing-test]` |
| [`eko ai ask`](#15-eko-ai-ask) | None | Query repository architecture memory | `<query>` |
| [`eko ai owners`](#16-eko-ai-owners) | None | Identify code maintainers & PR reviewers | `<file-path>` |
| [`eko ai next`](#17-eko-ai-next) | None | AI task & issue recommendation engine | None |
| [`eko ai security`](#18-eko-ai-security) | None | AI hardcoded secret & vulnerability scanner | None |
| [`eko ai gate`](#19-eko-ai-gate) | None | AI pre-commit quality gate evaluation | None |

---

## 1. `eko init`

Initializes a new Eko project in the current working directory. Creates the hidden `.eko/` folder containing the `snapshots/` directory and local SQLite database (`db.sqlite`).

```bash
eko init
```

**Behavior & Safety Guards**:
- Checks if the project is already initialized.
- Detects if a `.git` repository exists and displays a tip (Eko operates independently of Git and automatically ignores `.git`).

---

## 2. `eko save`

Captures the current filesystem state (excluding `.eko`, `.git`, `node_modules`, build artifacts) and stores it as a new 8-hex-character snapshot ID.

```bash
# Save snapshot with default message ("snapshot")
eko save

# Save with custom log description
eko save -m "fixed SQLite concurrency bug"

# Auto-generate AI change summary when saving
eko save --ai

# Auto-generate AI summary using a specific AI provider
eko save --ai --provider gemini
```

### Flags

| Flag | Short | Type | Default | Description |
| ---- | ----- | ---- | ------- | ----------- |
| `--message` | `-m` | String | `"snapshot"` | Log message describing the snapshot |
| `--ai` | `-a` | Bool | `false` | Auto-generate AI change summary using LLM/heuristic provider |
| `--provider` | | String | `"auto"` | AI provider for auto-summary (`auto`, `heuristic`, `openai`, `gemini`) |

---

## 3. `eko summary` (Alias: `eko summarize`)

Calculates file diffs (insertions, deletions, modifications) between snapshots and generates an AI-powered summary.

```bash
# Summarize changes in the latest snapshot vs predecessor
eko summary

# Summarize changes introduced in snapshot <id>
eko summary 3b7f2a1e

# Summarize changes between two specific snapshots
eko summary 3b7f2a1e 8c9d1a2f

# Output summary in JSON format
eko summary --json

# Force Gemini AI provider and save generated summary to SQLite DB
eko summary 3b7f2a1e --provider gemini --save
```

### Flags

| Flag | Short | Type | Default | Description |
| ---- | ----- | ---- | ------- | ----------- |
| `--json` | `-j` | Bool | `false` | Output change stats and summary in structured JSON format |
| `--provider` | `-p` | String | `"auto"` | AI provider engine (`auto`, `heuristic`, `openai`, `gemini`) |
| `--save` | `-s` | Bool | `false` | Save/update the generated summary in the SQLite database record |

---

## 4. `eko history`

Lists all recorded snapshots in reverse chronological order with creation timestamps, log messages, and AI summaries.

```bash
# Standard history view
eko history

# Verbose view with detailed AI summaries
eko history --verbose

# Programmatic JSON output
eko history --json

# Markdown table, for pasting into a changelog or a PR description
eko history --format md

# CSV, for a spreadsheet or a reporting pipeline
eko history --format csv > history.csv
```

### Flags

| Flag | Short | Type | Default | Description |
| ---- | ----- | ---- | ------- | ----------- |
| `--format` | | String | `text` | Output format: `text`, `json`, `md`, or `csv` |
| `--json` | | Bool | `false` | Output history list as JSON array (shortcut for `--format json`) |
| `--verbose` | `-v` | Bool | `false` | Show verbose history with detailed AI summaries |

**Format Notes**:
- `md` renders a table with fixed `ID`, `Created At`, `Message`, `Summary` columns. Embedded newlines are collapsed and pipes escaped so one snapshot stays one row, and the header is written even when there are no snapshots.
- `csv` uses RFC 4180 quoting, so commas, quotes, and newlines inside a message survive intact. The header is `id,created_at,message,summary`.
- `--json` and `--format` may be combined only when they agree; `--json --format md` is rejected rather than silently printing JSON.

---

## 5. `eko restore <snapshot-id>`

Reverts the working directory to the exact state captured in snapshot `<snapshot-id>`.

```bash
eko restore 3b7f2a1e
```

**Restoration Engine Details (Differential Smart Restore)**:
1. **Workspace Diff Scan**: Walks workspace and target manifest tree (`.eko/manifests/<id>.json`).
2. **Selective Removal**: Deletes only files that do NOT exist in the target snapshot.
3. **Identical File Skip**: Compares size & SHA-256 hashes — skips re-decompressing files that are already identical on disk (90%+ I/O reduction).
4. **Parallel Worker Pool**: Decompresses and extracts only missing/modified blobs from `.eko/objects/` in parallel (~27.6 ms for 1,000 files).
5. **Environment Restoration**: Generates `.eko_env_restore.sh` to restore captured environment variables.

---

## 6. `eko clean`

Removes old snapshots from `.eko/snapshots` and from the database, freeing the disk space they use. Snapshots are ordered newest first; the newest `--keep` are retained and every older one is removed.

```bash
# Keep the 10 newest snapshots (default) and remove the rest
eko clean

# Keep only the 5 newest
eko clean --keep 5

# Show exactly what would be removed, without removing anything
eko clean --keep 5 --dry-run
```

### Flags

| Flag | Short | Type | Default | Description |
| ---- | ----- | ---- | ------- | ----------- |
| `--keep` | | Int | `10` | Number of most recent snapshots to keep |
| `--dry-run` | | Bool | `false` | Show what would be removed without removing anything |

**Safety Details**:
1. **Validate-Then-Delete**: Every snapshot selected for removal is validated before any of them is deleted. A single unexpected path aborts the run before anything is touched.
2. **Path Confinement**: A recorded path is only accepted when it resolves, through symlinks, to a direct child of `.eko/snapshots` whose directory name matches the snapshot ID.
3. **Inert Dry Run**: `--dry-run` opens the database read-only and rejects writes at the connection level, so it cannot change a single byte.
4. **Progress on Failure**: Removal is not atomic. If a deletion fails partway, the error reports exactly how many snapshots were removed, and the next run continues from there.
5. **CAS Garbage Collection**: Automatically purges orphaned blobs from `.eko/objects/` that are no longer referenced by any snapshot manifest.

---

## 6. `eko tag <snapshot-id> <tag-name>`

Assigns a human-readable tag/alias to an 8-character snapshot ID so you can restore or summarize using human names (e.g., `v1.0`, `pre-refactor`).

```bash
eko tag 8c9d1a2f pre-refactor
eko restore pre-refactor
```

---

## 7. `eko migrate`

Converts legacy full-directory snapshots (`.eko/snapshots/<id>/`) to the high-efficiency Content-Addressable Storage (CAS) object store and JSON manifest format (`.eko/manifests/<id>.json`).

```bash
# Preview what snapshots will be converted
eko migrate --dry-run

# Run migration
eko migrate
```

### Flags

| Flag | Short | Type | Default | Description |
| ---- | ----- | ---- | ------- | ----------- |
| `--dry-run` | | Bool | `false` | Inspect legacy snapshots eligible for migration without modifying files |

---

## 9. `eko ai` — GitMind AI Intelligence Suite

The `eko ai` command suite turns Eko into an architecture-aware AI developer agent.

```bash
# Intent-based status analysis & file role classification
eko ai status

# Automated AI code review & commit risk score (0-100)
eko ai review

# Behavioral semantic diff analysis (diffs behavior, not lines)
eko ai semdiff

# Output:
# Behavioral change:
# Before: Deletion logic executed only when Project = Ready.
# After: Deletion logic executes when Project = Active.
# Potential impact: Projects transitioning between Active and Ready may now follow a different lifecycle path.

# Multi-dimensional commit risk analysis
eko ai risk

# Output:
# Commit Risk Analysis
# Overall: HIGH
# ┌──────────────────────┬────────┐
# │ Area                 │ Risk   │
# ├──────────────────────┼────────┤
# │ Database             │ HIGH   │
# │ Authentication       │ LOW    │
# │ API                  │ MEDIUM │
# │ Tests                │ HIGH   │
# │ Configuration        │ LOW    │
# └──────────────────────┴────────┘
# Reasons:
# ⚠ Database schema changed
# ⚠ Migration has no rollback

# Subsystem change impact graph
eko ai impact

# Automated regression bug isolation
eko ai bisect "go test ./..."

# Query repository architecture memory
eko ai ask "Why do we use CAS storage?"

# Code ownership & reviewer match
eko ai owners internal/snapshot/snapshot.go

# Task & issue recommendation engine
eko ai next

# AI hardcoded secret & vulnerability scanner
eko ai security

# AI pre-commit quality gate evaluation
eko ai gate
```

---

## Global Environment Variables

| Variable | Default Value | Usage |
| -------- | ------------- | ----- |
| `GEMINI_API_KEY` | *(None)* | API Key for Google Gemini LLM provider |
| `OPENAI_API_KEY` | *(None)* | API Key for OpenAI LLM provider |
| `EKO_AI_API_KEY` | *(None)* | General API Key for AI provider |
| `EKO_AI_ENDPOINT` | `https://api.openai.com/v1` | Custom endpoint for OpenAI-compatible LLMs (vLLM, Ollama) |
| `EKO_AI_MODEL` | `gpt-4o-mini` / `gemini-1.5-flash` | Model override for AI provider |
