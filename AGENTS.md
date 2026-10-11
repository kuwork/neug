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

<!-- WIKI GUIDELINES START -->

## Wiki Knowledge Base

This project maintains an LLM-managed wiki inside the backlog directory for cross-referencing and compounding knowledge from tasks, docs, and decisions.

### Location
- `backlog/wiki/` — LLM-maintained knowledge base (do not edit manually)
- `backlog/wiki_output/` — Query products and generated artifacts

### Raw Sources (Human Input Layer)
The LLM reads from the following backlog folders as raw input. These are immutable for wiki purposes:
- `tasks/` — Requirements, acceptance criteria, implementation notes
- `docs/` — Documentation, guides, API references
- `decisions/` — ADRs, design choices, rationale
- `drafts/` — Draft ideas, brainstorming notes
- `milestones/` — Milestone definitions, roadmap items
- `archive/` — Archived tasks and records
- `completed/` — Completed task records with final summaries
- `assets/` — Images, diagrams, attachments
- `src/` (or other project source directories) — Project source code, implementation files (optional, when backlog is inside a project repo)

### Wiki Structure (LLM-Maintained Layer)
- `wiki/index.md` — Content catalog; read this FIRST on any operation
- `wiki/log.md` — Append-only chronological log (`## [YYYY-MM-DD HH:mm:ss] {op} | {title}`). The detailed timestamp enables git-aware incremental ingestion.
- `wiki/overview.md` — High-level synthesis of the entire knowledge base
- `wiki/sources/` — One summary per backlog source
- `wiki/concepts/` — Extracted concept articles
- `wiki/entities/` — People, tools, projects, organizations
- `wiki/comparisons/` — Cross-cutting analyses
- `wiki/patterns/` — Reusable task execution patterns extracted from completed work. AI detects similarities across tasks and proposes patterns; human confirms before creation.
- `wiki/decisions/` — Informal micro-decisions made during task execution ("chose X over Y because..."). Distinct from formal ADRs in `backlog/decisions/`.
- `wiki/reasoning/` — AI planning traces documenting how tasks were decomposed from docs. Humans review before execution to validate or correct planning logic.
- `wiki/execution/` — Key knowledge extracted from task Implementation Notes, elevated into cross-task reusable form. AI-maintained; human reviews occasionally.
- `wiki/retrospectives/` — Lightweight period reviews based on local task data (completion rates, cycle times, blockers). AI generates data; human adds qualitative observations.
- `wiki/usermanual/` — Optional structured user manual

### 页面风格约定 (Page Style Conventions)

自 2026-10-09 起生效；存量页面按 Lint 逐步收敛，新页面必须遵守。

1. **语言**：正文、标题、小节名一律简体中文。英文仅限代码标识符、命令、字段名、文件路径与固有专名（fail-closed、wikilink、hotfix、memo 等不译词）。同一页面不得整段切换语言。
2. **frontmatter 字段顺序固定**：`title`、`labels`、`created_date`、`updated_date`；source 页在 `updated_date` 后加 `source_path`；decision 页允许可选 `description`（一行摘要）；execution/patterns 页允许 `extracted_from`（裸任务 ID 列表，如 `- BACK-757`）。日期一律不加引号（`updated_date: 2026-10-09 22:00`）；labels 一律行内数组（`labels: [concept, graph]`）；不加约定外字段。
3. **title 格式**：source 页 = `BACK-757 - 中文短标题`（实体 ID + 连字符 + 中文短标题；doc/milestone/draft 页同理带各自 ID；顶层文件页如 README 用 `README.md 中文说明` 式，无 ID）。concept / decision / execution / reasoning / patterns / entities 页用中文名词短语、不含 ID。title 永不加引号。
4. **H1**：每页有且仅有一个 H1，逐字等于 frontmatter title（含 ID）。禁止无 H1 或 H1 丢失 ID。
5. **source 页骨架**：H1 → 导语一段（背景+结论）→ `## 问题`（bug）或 `## 目标`（feature）→ `## 解决方案` 或 `## 实现要点` → `## 验证`（有实测数据时）→ Related 小节。已有内容按此归位；禁止导语后没有任何正文小节直接进入 Related；禁止为凑结构凭空造节。
6. **decision 页骨架**：`## 背景` → `## 决定` → `## 理由` → `## 被否方案` → Related。小节名不得使用 Context/Decision 等英文。
7. **execution / patterns 页骨架**：`## 适用场景` → `## 标准步骤` → `## 常见陷阱` → `## Related Sources`（不叫"参考任务"；列表内只放 source 链接）。
8. **Related 小节**：命名只用 `Related Concepts` / `Related Sources` / `Related Entities` / `Related Decisions` / `Related Reasoning`，按此固定顺序；列表项格式 `- [[path/to/page]] — 一句中文说明`；禁止合并为 `## Related`；禁止空 Related 小节。每页至少一个 Related 小节。
9. **wikilink**：正文 prose 中允许别名 `[[target|BACK-543]]`；Related 列表内不用别名；所有跨页引用一律 `[[path]]`（不带 `.md`）。
10. **翻译忠实性**：英文存量页译成中文时保留全部事实、file:line 引用、命令、标识符与 wikilink 原样，不增删内容。
11. **术语对照**（翻译与写作统一）：modal=模态框、watcher=监视器、corpus=语料、terminal status=终态、cross-branch=跨分支、drill-down=钻取、pinboard=钉板、graph=图谱、fallback=回退、acceptance criteria=验收标准、board=看板、plain output=plain 输出。
12. **行尾 LF**；不改动 `wiki_output/`、`index.md`、`overview.md`、`log.md`（由 ingest/lint 流程统一维护）。

### Rules
- **NEVER** write to `tasks/`, `docs/`, `decisions/`, or other backlog source folders during wiki operations
- **NEVER** recursively ingest `wiki/` or `wiki_output/`
- Use `[[wikilinks]]` for all cross-references within the wiki
  - **CRITICAL:** In `index.md` tables, use `[[path/to/file]]` (without `.md`) as the cell value, not standard Markdown links like `[text](path.md)`
  - Example: `| [[sources/task-1-feature]] | Task | Description |` — NOT `| [task-1](sources/task-1.md) | Task | Description |`
  - **CRITICAL:** In page bodies (sources, concepts, entities), Related Concepts / Related Sources / Related Entities sections must also use `[[path/to/file]]`, not `[text](path.md)`
- Filenames: lowercase-with-hyphens
- YAML frontmatter on every wiki page at minimum: `title`, `created_date`, `updated_date`; `labels` as optional array of tags
- This project is **not** a git repository, so ingestion uses full-scan incrementality rather than git-aware detection.

### Operations
- **Ingest** — Read backlog sources, extract concepts/entities, create source summaries, update index/overview/log
- **Query** — Read index, synthesize from compiled wiki, produce chat responses / reports / slides / charts
- **Lint** — Scan for contradictions, orphans, stale claims, missing cross-references
- **Flowback** — Save valuable query results back into the wiki for compounding

<!-- WIKI GUIDELINES END -->
