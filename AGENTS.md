# AGENTS.md

This file provides guidance to LLM tools when working with code in this repository.

## Project Overview

**NeuG** is a C++20 graph database for HTAP workloads with Cypher query support. Two modes: **Embedded** (analytics) and **Service** (transactional).

## Build & Test

**Parallelism limit**: Always cap `-j` to the number of physical cores. Use `cmake --build build -j$(sysctl -n hw.ncpu)` (macOS) or `cmake --build build -j$(nproc)` (Linux). **Never use bare `-j`** (unlimited parallelism) — it exhausts memory and freezes the machine on this codebase.

All bindings (Python, future Node/Rust) share a single root build tree at `<repo>/build/`. The core library `libneug.dylib`/`libneug.so` is built once; each binding produces its own `.so` that dynamically links to it.

### Quick Start

```bash
# One-shot from repo root: configures root build, compiles core +
# neug_py_bind, installs Python deps. Idempotent — also serves as the
# incremental rebuild entry point.
make python-dev
```

Or directly from `tools/python_bind/`:
```bash
make requirements    # one-time: install Python deps
make dev             # incremental rebuild; auto-bootstraps if root build
                     # is missing or wasn't configured with BUILD_PYTHON=ON
```

For **C++ only** (no Python):
```bash
make cpp-build                     # core + executables, Release
BUILD_TEST=ON make cpp-test        # enable tests
BUILD_TYPE=Debug make cpp-build    # override build type
EXTRA_CMAKE_FLAGS="-DBUILD_HTTP_SERVER=ON" make cpp-build   # extra flags
```

### Common Build Variables

- `BUILD_TYPE=DEBUG|RELEASE` — default Release
- `BUILD_TEST=ON` — build test suites
- `BUILD_PYTHON=ON` — build `neug_py_bind` target (off by default in pure C++ builds)
- `NEUG_BUILD_DIR=<path>` — override the root build dir consumed by `setup.py` and the Python loader (default `<repo>/build`)
- `NEUG_PACKAGE_BUILD=ON` — CMake option for portable distributable artifacts; package builds must keep `NEUG_NATIVE_ARCH=OFF`
- `NEUG_NATIVE_ARCH=ON` — enable host-specific CPU tuning in local source builds; supported by root/Python/Node Makefiles
- `DEBUG=ON` + `GLOG_v=10` — enable verbose C++ logging

### Running Tests

```bash
# Python tests — loader auto-finds neug_py_bind*.so in the root build dir
cd tools/python_bind
python3 -m pytest -s tests/test_db_query.py

# C++ tests
ctest --test-dir build

# Debugging with verbose logging
GLOG_v=10 lldb -- python3 -m pytest -sv tests/test_db_query.py
```

### Building a Wheel

```bash
cd tools/python_bind
make wheel    # reuses root build; produces dist/neug-*.whl with libneug bundled
```

The wheel ships `neug_py_bind*.so` and `libneug.{dylib,so}` together; they find each other at runtime via `@loader_path` / `$ORIGIN` RPATH set in `tools/python_bind/CMakeLists.txt`.

### Pre-commit

```bash
# From repository root
make format-check    # clang-format + isort + black + flake8
```

## Architecture

```
include/neug/    # Public C++ headers (mirrors src/)
src/
├── compiler/    # ANTLR4 Cypher parser → logical plan → physical plan (via gopt/)
├── execution/   # Physical operators: scan, filter, project, join, aggregation
├── storages/    # CSR-based graph storage, schema, property columns
├── main/        # Core DB implementation: neug_db, connection, query processor
└── server/      # HTTP server for Service Mode
tools/python_bind/
├── src/         # pybind11 bindings
├── neug/        # Python API: Database, Connection, Session
└── tests/       # Python test suite
```

### Query Pipeline

```
Cypher → ANTLR Parser → Binder → Logical Plan → gopt Converter → Physical Plan → Execution
```

## Known Limitations

- **List literals require explicit CAST**: In `CREATE`, `SET`, and `MERGE` clauses, list values must be wrapped with `CAST(..., 'TYPE[]')` — bare list literals like `[1, 2, 3]` are rejected. Use `CAST([1, 2, 3], 'INT64[]')` instead.
- **List types cannot be primary keys**: Declaring a `PRIMARY KEY` on a `T[]` column is rejected.
- **Insert transactions may fail for non-empty list properties**: After loading a graph from a checkpoint directory, the `elements` column of a `ListPropertyColumn` has zero spare capacity (`elements_tail_ == elements_->size()`). The insert-transaction path always passes `insert_safe=false`, so inserting a **non-empty** list property will throw a `StorageException`. Workaround: insert with an empty list, then use an update-mode query (executed by `SnapshotCowWriteTransaction`, which allows resize), or pre-populate list data during bulk load. See `storages/README.md` §5.4 for details.

## Code Style

- **C++**: C++20, clang-format (style=file)
- **Python**: isort, black, flake8

<!-- BACKLOG.MD GUIDELINES START -->
<CRITICAL_INSTRUCTION>

## Backlog.md Workflow

This project uses Backlog.md for task and project management.

**At the beginning of each conversation in this project, run `backlog instructions overview` before answering or taking action. Re-read it only if you have not read it yet in the current conversation.**

Use the overview to decide whether to search, read, create, or update Backlog tasks.

Use the detailed guides when needed:
- `backlog instructions task-creation` for creating or splitting tasks
- `backlog instructions task-execution` for planning and implementation workflow
- `backlog instructions task-finalization` for completion and handoff
- `backlog instructions milestones` for creating, editing, removing, and archiving milestones
- `backlog instructions documents` for creating, updating, listing, and viewing project documents
- `backlog instructions drafts` for creating, promoting, demoting, or archiving drafts

Use `backlog <command> --help` before running unfamiliar commands. Help shows options, fields, and examples.

### Wiki Skill Installation

Backlog.md includes an LLM-managed wiki under `backlog/wiki/`. To enable wiki operations for the current agent, install the bundled `llm-wiki-for-backlog` skill:

- `backlog wiki install claude` — Claude Code / Claude Desktop
- `backlog wiki install codex` — OpenAI Codex CLI
- `backlog wiki install agents` — Generic agents directory

Use `--dry-run` to preview the operation and `--force` to overwrite an existing installation. After installing, refer to the skill guide for ingestion, query, lint, and flowback workflows.

Do not edit Backlog task, draft, document, decision, or milestone markdown files directly. Use the `backlog` CLI so metadata, relationships, and history stay consistent.

- If you encounter duplicate or ambiguous task IDs (e.g., `task-1` and `task-01`), run `backlog doctor` first. Do not rename files or edit frontmatter IDs manually. Apply `backlog doctor --fix` only after reviewing the preview, then use `backlog doctor --commit` to finalize or `backlog doctor --rollback` to undo before committing.

</CRITICAL_INSTRUCTION>
<!-- BACKLOG.MD GUIDELINES END -->

<!-- BACKLOG.MD STATE MACHINE START -->
## This project's state machine

**Moving a task:**
- Find the task's current status below and follow the matching `next` edge; obey the edge's `ai` and gather its `evidence` first.
- Never take an edge listed under "Stop and wait for the user" without the user's answer.

### Statuses

| Status | Category | Exit | Board |
| --- | --- | --- | --- |
| To Do | active | - | shown |
| Planning | wip | - | shown |
| Plan Review | blocked | - | shown |
| In Progress | wip | - | shown |
| In Review | blocked | - | shown |
| Done | done | complete | shown |
| Dropped | dropped | archive | hidden |

### AI permission tiers

| `ai` | What it means |
| --- | --- |
| `allowed` | the AI may make this move on its own |
| `allowed_if` | the AI may move only while the edge's `if` holds |
| `propose` | the AI proposes the move and waits for the user |
| `forbidden` | only the user may make this move |

### Transitions

Every declared `next` edge, grouped by the status it starts from.

#### To Do

- `To Do` -> `Planning` · when: The user has confirmed the description and acceptance criteria are complete; work can start · ai: allowed
- `To Do` -> `Dropped` · when: The task is obsolete or abandoned; tell the user that dropping it archives the task · ai: propose

#### Planning

- `Planning` -> `Plan Review` · when: The implementation plan has been written to implementationPlan · ai: allowed_if · if: implementationPlan is not empty

#### Plan Review

- `Plan Review` -> `In Progress` · when: The user has approved the implementation plan · ai: allowed_if · if: the user has explicitly approved the implementation plan in conversation or comments
- `Plan Review` -> `Planning` · when: The plan was rejected or needs changes · ai: propose · evidence: comments (stating the rejection reason)

#### In Progress

- `In Progress` -> `In Review` · when: Implementation is complete; diff / tests / acceptance notes are ready · ai: allowed_if · if: finalSummary is not empty
- `In Progress` -> `Planning` · when: The plan needs adjustment discovered during implementation · ai: allowed

#### In Review

- `In Review` -> `Done` · when: The user has accepted the work · ai: allowed_if · if: the user has explicitly accepted the work in conversation or comments
- `In Review` -> `In Progress` · when: Acceptance failed; rework needed · ai: propose · evidence: comments (stating the rework reason)

`Done` and `Dropped` have no outgoing transitions.

### Terminal statuses

Reaching a status whose `category` is `done` or `dropped` (see the Statuses table) stamps `actualEnd`: `exit: complete` marks finished work, `exit: archive` marks work set aside, archived rather than completed.

### Stop and wait for the user

These edges are not the AI's to take:

- `To Do` -> `Dropped` · ai: propose
- `Plan Review` -> `Planning` · ai: propose
- `In Review` -> `In Progress` · ai: propose
<!-- BACKLOG.MD STATE MACHINE END -->
