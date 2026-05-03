---
name: codemap
description: Generate and navigate CODEMAP.md index files for large codebases. Creates hierarchical per-directory CODEMAP.md files containing simplified directory structure, file/subdirectory function summaries, domain annotations, task-to-file routing, file-level dependency tables, and directory-level key exports with source location annotations. Use this skill whenever the user wants to index a codebase for efficient agent navigation, generate CODEMAP files, or when the user mentions "codemap", "code map", "codebase index", "project index", "codebase navigation", "generate navigation", or wants to understand a large project structure. Also triggers when the user asks to update or refresh existing CODEMAP files. Even if the user just says "index this project" or "map this codebase", use this skill.
---

# CODEMAP — Codebase Navigation Index Generator

Generate hierarchical `CODEMAP.md` files that serve as a structured navigation index for large codebases. Each directory gets its own CODEMAP.md containing a simplified directory tree, domain-annotated file/subdirectory summaries, task-to-file routing guides, file-level dependency tables, and directory-level key exports with source location annotations. For extra-large source files (>1000 lines), generate a companion deep-analysis file with feature-to-code index. Agent navigation follows a layered lazy-loading strategy: Task Guide first → Domain filter → drill down → batch-read only the source files actually needed.

## Language Rule

All generated CODEMAP.md files and analysis.md files MUST be written in the same language as the user's request. If the user asks in Chinese, all summaries, purpose descriptions, function descriptions, and section headings (except code identifiers and file names) are written in Chinese. If the user asks in English, write in English. Code identifiers (`ClassName`, `function_name`, file paths) always retain their original form regardless of language.

## Two Operating Modes

Use the `AskUserQuestion` tool to ask the user **before** doing anything else. Present all three questions in a single call:

**Question 1** (header: "Mode", single-select):
- **Learning (Recommended)** — read-only study, no code changes expected
- **Maintenance** — active development, bugs/features/refactors

**Question 2** (header: "Sub-agents", single-select):
- **Yes, max 3 (Recommended)** — parallel sub-agent generation with default limit of 3
- **Yes, custom limit** — parallel sub-agent generation, user specifies max count
- **No** — single-agent serial generation

**Question 3** (header: "Ignore", single-select):
- **Defaults only (Recommended)** — use built-in ignore list + .gitignore
- **Add custom patterns** — user provides additional ignore patterns

Mode affects:

| Aspect | Learning | Maintenance |
|---|---|---|
| CODEMAP frontmatter | `mode: learning` | `mode: maintenance`, includes `commit: <hash>` |
| Update strategy | One-time generation, no updates | Incremental via `git diff`, regenerate changed dirs only |
| Content tone | May include brief design-intent notes | Concise, purely navigational |
| AGENTS.md clause | Declares CODEMAP existence + read/constraint rules | Additionally declares: full update rules with decision tree |

## Ignore Rules (Three Layers)

Apply in order, merge results:

**Layer 1 — Built-in defaults:**
```
node_modules/, .git/, dist/, build/, out/, target/,
__pycache__/, .venv/, venv/, env/, .env, .egg-info/,
*.pyc, *.pyo, *.min.js, *.min.css, *.map,
*.lock, package-lock.json, yarn.lock, pnpm-lock.yaml,
.DS_Store, Thumbs.db, *.log,
.idea/, .vscode/, .vs/, *.swp, *.swo,
coverage/, .nyc_output/, .pytest_cache/, .mypy_cache/,
*.so, *.dylib, *.dll, *.o, *.obj, *.exe,
*.png, *.jpg, *.jpeg, *.gif, *.ico, *.svg, *.bmp,
*.woff, *.woff2, *.ttf, *.eot
```

**Layer 2 — Project `.gitignore`:** If exists, read and merge patterns.

**Layer 3 — User custom:** From Q3 above.

## Generation Workflow

### Phase 0: Global Context

1. Read `README.md` (or `README.rst`, `README.txt`) at project root.
   - If no README, fall back to `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod` / `pom.xml` description fields + root file list.
   - If none of the above exist, synthesize from root file list + first-level subdirectory names. Reasonable functional guesses are acceptable when information is scarce, but always note "based on structure inference, subject to code verification" for any guessed content.
2. Combine with the filtered directory topology from Phase 1 to produce a **project global context summary**. Keep it concise — summarize what the project does, its high-level architecture, and major functional domains. Do not pad with installation instructions, badges, or changelogs.

### Phase 1: Scan, Filter, and Measure

1. Run `Glob` + `ls` on the project root, applying all three ignore layers.
2. Produce a filtered directory topology: for each first-level subdirectory, collect its source files recursively.
3. Identify root-level loose files (files not inside any subdirectory).
4. **Collect metrics** for sub-agent dispatch (run these commands after filtering):
   - **Per first-level subdirectory**: total file count, total code line count (`wc -l` or equivalent on all source files), total size on disk (`du -sh` or equivalent).
   - **Project total**: sum of the above.
   - These metrics determine sub-agent count and load balancing (see Phase 2).
5. **Identify large files**: Flag any source file exceeding **1000 lines**.
   - If the count of large files is **≤ 5**: generate deep-analysis companion files for all of them automatically.
   - If the count is **> 5**: present the list of large files (with line counts) to the user and ask:
     ```
     Found N files exceeding 1000 lines. Generate deep-analysis files for:
       A) All of them
       B) Only the top 5 largest
       C) Let me pick which ones
       D) None — skip deep analysis
     ```
   - See "Large File Deep Analysis" section for the companion file format.

### Phase 2: Sub-agent Dispatch (if enabled)

**Determine actual sub-agent count K:**

Let L = total code line count after filtering, S = total size on disk, N = user-specified max sub-agents.

| Condition | K |
|---|---|
| L <= 3,000 lines OR S <= 500 KB | 1 (not worth splitting) |
| 3,000 < L <= 15,000 OR 500 KB < S <= 3 MB | min(N, 2) |
| L > 15,000 OR S > 3 MB | N |

When line count and size suggest different K values, use the larger K.

**Load balancing — Greedy Bin Packing by code line count:**

1. Sort first-level subdirectories by code line count descending.
2. Maintain K bins (one per sub-agent), each tracking total line count.
3. For each directory: assign to the bin with the smallest current total.
4. Root-level loose files: assign to the lightest bin, or handle by the main agent if total lines <= 200.

The goal is to equalize **code line count** across sub-agents (not file count), since line count correlates more closely with actual reading and analysis effort.

**Each sub-agent receives:**
- Project global context summary (compressed, from Phase 0)
- Ignore rules
- Output language (matching the user's request language)
- List of directories assigned to it (with paths)
- List of large files (>1000 lines) within its assigned directories that the user confirmed for deep analysis (from Phase 1 Step 5 selection)
- Instruction: for each assigned directory and all its subdirectories, read source files and generate CODEMAP.md files following the format spec below. Additionally:
  - Analyze each file's imports to populate the **Domain** column, **Cross-Dir Dependencies** column, and **File Dependencies** table.
  - Identify functional domains within the directory and create the **Task Guide** table mapping task types to target files.
  - For `.analysis.md` files, generate the **Feature Index** mapping development intents to line ranges.

**Each sub-agent produces:**
- One CODEMAP.md per directory it is responsible for
- One `<filename>.analysis.md` per large file (>1000 lines) in its scope

### Phase 2 (alternative): Serial Generation (if sub-agents disabled)

Main agent processes first-level subdirectories one by one in descending line-count order. For each directory: read all source files within it, generate CODEMAP.md for it and all its subdirectories. Generate deep-analysis files for any source file exceeding 1000 lines. All new fields (Domain, Cross-Dir Dependencies, File Dependencies, Task Guide, Feature Index) must be populated.

### Phase 3: Root Assembly

1. Main agent reads the CODEMAP.md of each first-level subdirectory (summary line + Task Guide + Dependencies sections, not full content).
2. Generate root-level CODEMAP.md: global context summary + simplified directory tree + subdirectory table (with Domain) + root file table (with Domain) + directory-level Dependencies (with internal/external marking) + root Task Guide (synthesized from subdirectory Task Guides) + directory-level key exports with source annotations.
3. Update `CLAUDE.md` or `AGENTS.md` (create if absent) with the CODEMAP declaration clause.

## CODEMAP.md Format Specification

### Root-Level CODEMAP.md

```markdown
---
mode: learning | maintenance
commit: abc1234f          # maintenance mode only
ignore: node_modules/, dist/, __pycache__/, ...
generated_at: 2026-04-28
stats:
  total_files: 114
  total_lines: 18200
  total_size: 4.2 MB
---

# CODEMAP — <project-name>/

> <Project global context summary: what it does, architecture overview, major functional domains. Concise, based on README + directory structure. Any inferred content marked as "inferred, verify against code.">

## Directory Structure

\```
src/
  models/
  data/
  utils/
configs/
scripts/
tests/
\```

## Key Exports

| Symbol | Source | Line |
|---|---|---|
| `main()` | `main.py` | L:15 |
| `App` | `src/app.py` | L:8 |
| `create_server()` | `src/server.py` | L:42 |
| `Config` | `configs/base.py` | L:12 |

## Subdirectories

| Directory | Domain | Purpose |
|---|---|---|
| `src/` | Core | Core source code: models, data processing, utilities |
| `configs/` | Configuration | Configuration files for training, deployment, and environments |
| `tests/` | Testing | Test suites: unit, integration, e2e |

## Dependencies

Internal = directories under this CODEMAP's scope. External = directories outside this scope.

| Directory | Depends On | Depended By |
|---|---|---|
| `src/models/` | `src/data/` (internal), `configs/` (external) | `src/training/` (internal), `src/inference/` (internal) |
| `src/data/` | `configs/` (external) | `src/models/` (internal), `tests/` (external) |
| `src/utils/` | — | all other modules |
| `configs/` | — | `src/models/` (external), `src/data/` (external) |
| `tests/` | `src/` (external) | — |

## Task Guide

Maps common development task types to the directories and files most likely involved. Also Check lists cross-directory files that are empirically relevant to this task type.

| Task Type | Domain | Target Subdirs / Files | Also Check |
|---|---|---|---|
| 新增/修改认证方式 | Auth | `src/auth/` | `configs/auth.yaml` |
| Token 黑名单/过期逻辑 | Auth | `src/auth/token_store.py`, `src/auth/jwt.py` | — |
| 用户数据模型字段变更 | User Data | `src/models/user.py`, `src/schemas/user.py` | `migrations/` |
| 新增 API 端点 | API | `src/routes/`, `src/services/` | `src/middleware/auth.py` |
| 配置项新增/变更 | Configuration | `configs/` | `src/utils/config_loader.py` |

## Files

| File | Domain | Function |
|---|---|---|
| `main.py` | Entry Point | Application entry point, CLI argument parsing |
| `setup.py` | Build | Package build and installation configuration |
| `Makefile` | Build | Common development task shortcuts (lint, test, build) |
```

### Subdirectory-Level CODEMAP.md

```markdown
---
mode: learning | maintenance
commit: abc1234f          # maintenance mode only
---

# CODEMAP — src/models/

> <One-paragraph summary of this directory's role in the project, 2-4 sentences. Reference the project global context to explain the role.>

## Directory Structure

\```
backbones/
heads/
\```

## Key Exports

| Symbol | Source | Line |
|---|---|---|
| `BaseModel` | `base_model.py` | L:23 |
| `build_model()` | `registry.py` | L:56 |
| `MODEL_REGISTRY` | `registry.py` | L:10 |
| `FocalLoss` | `losses.py` | L:18 |
| `DiceLoss` | `losses.py` | L:87 |

## Subdirectories

| Directory | Domain | Purpose |
|---|---|---|
| `backbones/` | Model Architecture | Backbone network implementations (ResNet, ViT, etc.) |
| `heads/` | Model Architecture | Task-specific heads (classification, detection, segmentation) |

## Dependencies

Internal = directories under this CODEMAP's scope. External = directories outside this scope.

| Directory | Depends On | Depended By |
|---|---|---|
| `backbones/` | `heads/` (internal) | — |
| `heads/` | — | `backbones/` (internal) |

## Task Guide

| Task Type | Domain | Target Files | Also Check |
|---|---|---|---|
| Backbone 架构变更 | Model Architecture | `backbones/`, `base_model.py` | `configs/model/` |
| 新增 Loss 函数 | Training | `losses.py` | `configs/training.yaml` |
| 模型注册/工厂变更 | Model Registry | `registry.py` | `src/training/trainer.py` |

## File Dependencies (within this directory)

| File | Imports (in-dir) | Exposed To (in-dir) |
|---|---|---|
| `base_model.py` | — | `user_model.py`, `order_model.py` |
| `user_model.py` | `base_model.py` | `user_repo.py`, `user_service.py` |
| `registry.py` | `base_model.py`, `backbones/`, `heads/` | `src/training/` (external) |
| `losses.py` | — | — |

## Files

Cross-Dir Dependencies: **Imports** = files this file depends on outside this directory. **Exposed To** = files outside this directory that depend on this file. If Exposed To exceeds 5 files, replace the list with a grep command and mark as "foundational."

| File | Domain | Cross-Dir Dependencies | Function |
|---|---|---|---|
| `base_model.py` | Model Architecture | **Imports:** `configs/db.yaml`<br>**Exposed To:** 17 files — foundational. `grep -r "from.*base_model import\|import.*base_model" src/` to list dependents when needed. | Abstract base class for all models, defines forward pass and weight loading interfaces |
| `registry.py` | Model Registry | **Imports:** `configs/model_registry.yaml`<br>**Exposed To:** `src/training/trainer.py`, `src/inference/predictor.py` | Model registry, instantiates models by name string |
| `losses.py` | Training | **Imports:** —<br>**Exposed To:** `src/training/trainer.py` | Loss function collection (Focal, Dice, etc.) |
| `__init__.py` | — | — | Module public interface |
```

### Format Rules

1. **Directory Structure section**: Show only directory names (no files), depth limited to 2 levels below current directory. Purpose: give the agent a quick spatial overview.
2. **Key Exports section**: A table listing directory-level aggregate symbols — classes, functions, constants that callers from other directories would use. Each entry includes the **source file path** (relative to current directory) and **line number** (`L:<number>`). Sort by importance descending: most architecturally significant symbols first. This is a directory-level view — do NOT create per-file export lists.
3. **Subdirectories table**: One row per immediate subdirectory. Domain column: functional domain this subdirectory belongs to (e.g., Auth, Model Architecture, Data Access). Purpose column: one sentence, 10-25 words.
4. **Dependencies section** (root-level and mid-level CODEMAPs only): A table listing inter-directory dependency relationships. Each row shows what a directory depends on and what depends on it. Mark each entry as `(internal)` if the referenced directory is within this CODEMAP's scope, or `(external)` if outside. External dependencies stop at the directory level — do not expand to file-level chains. This helps agents trace the impact direction of a modification. Omit for leaf directories.
5. **Task Guide section** (all CODEMAPs): A table mapping common development task types to their most likely target files or subdirectories. Each row represents a concrete task scenario (verb + object, e.g., "新增 Loss 函数" not "Loss 相关"). Columns:
   - **Task Type**: Concrete development task description.
   - **Domain**: The functional domain this task belongs to (must match Domain values used in Files/Subdirectories tables).
   - **Target Files / Target Subdirs**: The file(s) or subdirectory(ies) directly involved in this task type. Use project-relative paths for cross-directory targets.
   - **Also Check**: Cross-directory files empirically relevant to this task type. These are NOT exhaustive dependency listings — only files that experience shows are commonly needed. Use project-relative paths.
   - The Task Guide is the **primary navigation source** for agents. When a task matches a Task Type row, the Target + Also Check columns define the initial read set.
6. **File Dependencies section** (subdirectory CODEMAPs only): A table listing import relationships between files **within this directory only**. Cross-directory relationships go in the Files table's Cross-Dir Dependencies column. Columns:
   - **File**: The source file name (relative to this directory).
   - **Imports (in-dir)**: Files within this same directory that this file imports.
   - **Exposed To (in-dir)**: Files within this same directory that import this file's public symbols.
   - Purpose: enable precise impact analysis for same-directory changes. The Exposed To column triggers additional reads ONLY when the modification changes a public symbol's signature, return type, or documented semantics — not for internal implementation changes.
7. **Files table**: One row per file in the current directory (not recursive). Columns:
   - **File**: File name.
   - **Domain**: Functional domain this file belongs to. Must be one of the Domain values used in Task Guide and Subdirectories tables. Files that serve multiple domains may list the primary one. For `__init__.py` and other pure re-export files, use `—`.
   - **Cross-Dir Dependencies** (subdirectory level only; omit for root-level CODEMAPs): **Imports** lists files outside this directory that this file depends on. **Exposed To** lists files outside this directory that depend on this file. When Exposed To has ≤5 entries, list exact file paths. When >5 entries, write the count followed by "— foundational." and a grep command to dynamically list dependents (e.g., `grep -r "from.*base_model import\|import.*base_model" src/`). Use project-relative paths for all entries. If neither Imports nor Exposed To has content, use `—`.
   - **Function**: One sentence, 10-30 words describing the file's purpose. For large files (>1000 lines), append `→ see <filename>.analysis.md`.
   - Skip `__init__.py` if it only re-exports (mention in Key Exports instead).
8. **Summary paragraph** (below the heading): describe this directory's responsibility within the project. Reference the project global context to explain the role. 2-4 sentences. When information is insufficient for certainty, state a reasonable inference and mark it: "inferred, verify against code."
9. **Leaf directories** (no subdirectories): omit the "Directory Structure" and "Subdirectories" sections. The "Dependencies" section is also omitted (no subdirectories to relate). Keep File Dependencies, Task Guide, Files, and Key Exports.
10. **Domain consistency**: Domain values used in Subdirectories, Files, and Task Guide tables within the same CODEMAP must match exactly (case-sensitive). Use short, descriptive names: e.g., `Auth`, `User Data`, `Model Architecture`, `Training`, `API`, `Configuration`, `Data Access`, `Observability`.

## Large File Deep Analysis

For any source file exceeding **1000 lines**, generate a companion file named `<filename>.analysis.md` in the same directory as the source file. This file provides a structural map so agents can navigate the large file without reading it in full.

### Format

```markdown
---
source: <filename>
lines: 2847
generated_at: 2026-04-28
---

# Analysis — <filename>

> <One-sentence summary of this file's overall purpose>

## Top-Level Symbols

| Symbol | Type | Line | Purpose |
|---|---|---|---|
| `TransformerEncoder` | class | L:45 | Main encoder class, manages multi-head attention layers and feed-forward blocks |
| `MultiHeadAttention` | class | L:198 | Scaled dot-product attention with configurable heads |
| `PositionalEncoding` | class | L:412 | Sinusoidal position embeddings for sequence inputs |
| `build_encoder()` | function | L:680 | Factory function, builds encoder from config dict |
| `DEFAULT_CONFIG` | constant | L:12 | Default hyperparameters for encoder construction |
| `_compute_mask()` | function | L:720 | Internal: generates causal attention masks |

## Class Hierarchy (if applicable)

\```
nn.Module
  └── TransformerEncoder
        ├── MultiHeadAttention
        └── PositionalEncoding
\```

## Feature Index

Maps development intents to the line ranges that need to be read. Each row targets a specific task scenario. Notes column flags cross-section dependencies within the same file to avoid missing related code.

| Intent | Target Sections (line range) | Notes |
|---|---|---|
| 新增注意力机制 | L:198-411 (`MultiHeadAttention`) | 继承 `AttentionBase`（L:45-80），需同步修改注册表 L:680-720 |
| 修改损失计算 | L:520-680 | 独立区块，不依赖文件内其他符号 |
| 调整编码器架构 | L:45-197, L:680-850 | 涉及类定义 + 工厂函数两处 |
| 修改默认超参数 | L:1-44 | 仅常量区，自包含 |
| 理解数据流入流出 | L:120-160 (`forward()`), L:680-720 (工厂函数) | 快速建立心智模型 |

## Logical Sections

| Line Range | Content |
|---|---|
| 1-44 | Imports, constants, configuration defaults |
| 45-197 | `TransformerEncoder` class definition |
| 198-411 | `MultiHeadAttention` class definition |
| 412-679 | `PositionalEncoding` + utility functions |
| 680-850 | Factory functions and public API |
| 851-end | Internal helpers and deprecated code |
```

### Rules

1. **Top-Level Symbols table**: List all classes, standalone functions, and module-level constants/variables. Include both public and important private symbols (prefix `_`). Sort by line number ascending.
2. **Type column**: `class`, `function`, `constant`, `variable`, `decorator`, `type alias`, etc.
3. **Class Hierarchy**: Only include if the file defines inheritance relationships. Use a tree diagram. Omit if all classes are independent.
4. **Feature Index**: Maps development intents (verb + object tasks like "新增注意力机制", "修改损失计算") to the specific line ranges that must be read. This is the primary navigation tool for agents working with this file — match the task to an Intent row, read the listed line ranges. Rules:
   - Each Intent row describes a concrete task scenario, not a general feature category.
   - Target Sections lists one or more line ranges, with the relevant symbol name in parentheses for clarity.
   - Notes column flags cross-section dependencies (e.g., "需同步修改注册表 L:680-720") or notes that a section is self-contained.
   - Multiple non-contiguous ranges are listed with commas (e.g., `L:45-197, L:680-850`).
   - If no clear task-to-range mapping can be inferred for a file, the Feature Index may have fewer rows than Logical Sections. That is acceptable — Logical Sections serves as the fallback.
5. **Logical Sections**: Divide the file into coherent blocks by line range. Purpose: fallback navigation when no Feature Index row matches the task, or when the agent needs to understand overall file structure.
6. Keep the analysis file concise — it is a structural map, not documentation. No code snippets, no API signatures, no implementation details.

## CLAUDE.md / AGENTS.md Declaration

Append the following block to the project's `CLAUDE.md` or `AGENTS.md` (create the file if neither exists; prefer `AGENTS.md`). These are **default navigation conventions** — the user may override or relax any rule if they have a specific workflow preference.

```markdown
## CODEMAP Navigation Protocol

This project contains `CODEMAP.md` index files in the root and each source subdirectory. These files provide a structured navigation index for efficient codebase exploration. Companion `<filename>.analysis.md` files provide structural maps for files exceeding 1000 lines.

### Reading Rules

1. **Task Guide First**: Before any file reads, check the root CODEMAP's Task Guide table. If a Task Type row matches the current task, the Target + Also Check columns define the initial read set. This is the primary navigation path — skip domain guessing.
2. **Domain-First Filtering**: If no Task Guide row matches, filter by Domain. Match the task to a Domain value, then read only the CODEMAPs and files whose Domain column matches. Files with non-matching Domains are excluded unless explicitly listed in a matched Task Guide's Also Check column.
3. **Layer-by-layer drill-down**: Read root CODEMAP → identify relevant subdirectories (via Task Guide or Domain match) → read their CODEMAPs in parallel → identify target files → batch-read source files.
4. **Task Guide precision at every level**: At each CODEMAP level, consult the local Task Guide before reading files. The Target Files column at that level defines the precise file set for that directory — do not expand beyond it unless first-pass analysis proves additional files are needed.
5. **Cross-Dir Dependency discipline**:
   - **Imports** entries list files the current file depends on. Read them only when the current file's logic cannot be understood without seeing the imported interface's contract (signature, return type, exception semantics).
   - **Exposed To** entries (≤5 files) list files that depend on the current file. Read them only when the modification changes a public symbol's signature, return type, or documented behavior — not for internal implementation changes.
   - **Exposed To** entries (>5 files, "foundational" with grep command): Run the provided grep command to identify dependents, then filter the results to those matching the current task's Domain. Read only those filtered files.
6. **File Dependencies discipline** (within-directory Imports/Exposed To): Same rules as Cross-Dir Dependencies. Imports read only when interface contract understanding is needed. Exposed To read only when public signature/semantics change.
7. **Two-Stage Read Protocol**:
   - Stage 1: Read exactly the files identified by Task Guide (Target + Also Check) and Domain match. For large files, read `.analysis.md` Feature Index first, then targeted line ranges.
   - Stage 2: Only if Stage 1 analysis reveals that additional files are needed, read them with explicit justification for each additional file. Never pre-emptively expand the read set "just in case."
8. **Batch parallel reads**: After identifying all target files through CODEMAP navigation, read them all in one parallel batch — not one by one.
9. **Key Exports shortcut**: When searching for a specific symbol (class, function, constant), scan the "Key Exports" table in each CODEMAP to locate which directory and file owns it, along with the exact line number.
10. **Feature Index shortcut**: For large files, match the task intent against the `.analysis.md` Feature Index rows. Read only the line ranges listed in matching rows. Use Logical Sections as fallback when no Feature Index row matches.
```

For **maintenance mode**, additionally append:

```markdown
### Update Rules

After completing a code modification task, the agent autonomously assesses whether CODEMAP updates are needed. Use the following decision tree:

1. **Did the change add, delete, move, or rename any file or directory?** → **Structural change**. Regenerate the affected directory's CODEMAP.md (all sections: Files, Key Exports, File Dependencies, Cross-Dir Dependencies, Task Guide). Propagate upward: update parent CODEMAP Subdirectories and Dependencies tables. Create/delete/rename `.analysis.md` if large files were added/removed. Update `commit` field in all modified CODEMAPs.

2. **Did the change modify a public symbol's declaration?** (class/function/constant added, removed, renamed; function signature parameters added/removed/renamed; return type changed) → **Interface change**. Update:
   - Key Exports table in the affected directory's CODEMAP (add/remove/rename entries).
   - Propagate to parent CODEMAP Key Exports if the symbol was listed there.
   - Files table Cross-Dir Dependencies: update Imports if this file added/removed cross-directory imports. Update Exposed To in the imported file's directory CODEMAP if the dependency count crosses the 5-file threshold.
   - Task Guide: update only if the signature change creates a new typical task pattern.
   - `commit` field in all modified CODEMAPs.

3. **Did the change only modify internal implementation?** (bug fix, internal refactor, algorithm tweak, parameter adjustment, comment changes) → **Implementation change**. No CODEMAP updates needed. Exception: update `.analysis.md` Logical Sections line ranges if code shifted by more than 20 lines within a >1000-line file. Feature Index line ranges: update only if the logical mapping between intents and code sections changed (not for mere line shifts — those are corrected at next full regeneration).

4. **No updates for**: comment-only changes, test file changes (outside CODEMAP scope), config value changes that don't alter which files import the config.

The agent decides autonomously after each task — no user intervention needed.
```

## Navigation Workflow (for agents reading the codebase)

This is how an agent should use CODEMAP.md files when performing any code reading or search task:

```
Step 1: Read root CODEMAP.md
        → Check Task Guide first: does any Task Type row match the current task?
          Yes → Target Subdirs / Files + Also Check define the initial read set.
          No  → Match task to a Domain. Use Subdirectories + Files tables filtered
                by that Domain to identify target directories and root-level files.
        → If Key Exports directly shows the target symbol, note file + line.

Step 2: Read target subdirectories' CODEMAP.md files IN PARALLEL
        → Again, Task Guide first at each level — match Task Type to get precise
          Target Files + Also Check.
        → If no Task Guide match, filter Files table by Domain column.
        → Consult File Dependencies table ONLY if:
          - The task involves understanding an imported interface's contract →
            read Imports files.
          - The task changes a public symbol's signature/semantics →
            read Exposed To files.
          - Otherwise skip File Dependencies — do not chain-read.

Step 3: Resolve cross-directory dependencies from the Files table
        → Cross-Dir Dependencies column:
          - Imports: read only when interface contract understanding is needed.
          - Exposed To ≤5: read only when public signature/semantics change.
          - Exposed To >5 (foundational): run the grep command → filter results
            by current task's Domain → read only matching files.
        → Task Guide Also Check: read all listed files — they are empirically
          validated for this task type.

Step 4: Compile the final list of target source files
        → Union of: Task Guide Target + Also Check + Domain-matched files
          + Exposed To (if public change) + Imports (if contract check needed).
        → Remove duplicates. This is the complete read set.

Step 5: For large files in the target list, read their .analysis.md IN PARALLEL
        → Check Feature Index: match task intent → read listed line ranges.
        → No Feature Index match → use Logical Sections to determine ranges.
        → Regular files in the target list: read in full.

Step 6: Batch-read ALL target source files IN ONE PARALLEL BATCH
        → Large files: use offset/limit from Step 5.
        → Regular files: read in full.
        → This is the only step where actual source code is read.

Step 7 (Stage 2, conditional): Only if Step 6 analysis proves additional files
        are needed, read them with explicit justification for each file.
        Never pre-emptively expand the read set beyond Step 4's result.
```

**Efficiency principle**: Reading 3-4 CODEMAPs (~200 lines) to precisely locate 5-8 source files is far cheaper than scanning the entire codebase. The Task Guide + Domain filter narrows the candidate set before any source code is read. For large files, the Feature Index further reduces token consumption by enabling targeted line-range reads. File Dependencies and Cross-Dir Dependencies are safety nets — not reading mandates — and should only trigger reads when the specific conditions (signature/semantics change, contract understanding needed) are met.

## Incremental Update (Maintenance Mode Only)

After completing a code modification task, the agent autonomously assesses whether CODEMAP updates are needed:

### Decision Tree

```
修改完成后：
1. 是否增删移了文件/目录？        → 是 → 结构变更流程
2. 是否变更了公开符号声明？        → 是 → 接口变更流程
3. 是否仅内部实现/注释/配置值？    → 是 → 实现变更流程（几乎无更新）
```

### Change Type Classification

#### 1. Structural Change (file/directory added, deleted, moved, renamed)

| Update Item | Action |
|---|---|
| Affected directory's CODEMAP.md | **Regenerate**: Files table, Key Exports, File Dependencies, Cross-Dir Dependencies, Task Guide |
| Parent CODEMAP.md | **Update**: Subdirectories table, Dependencies table |
| Affected dependents' Exposed To | **Update**: file paths in Cross-Dir Dependencies Exposed To entries of directories whose files import the moved/renamed file |
| Affected Task Guide entries | **Update**: file paths in Target Files and Also Check columns that reference moved/renamed files |
| `.analysis.md` | **Create/delete/rename** if a large file (>1000 lines) was added, deleted, or renamed |
| `commit` field | **Update** in all modified CODEMAPs |

#### 2. Interface Change (public class/function/constant added, removed, renamed; signature params added/removed/renamed; return type changed)

| Update Item | Action |
|---|---|
| Key Exports table (this dir) | **Update**: add/remove/rename symbol entries; update Line numbers |
| Key Exports table (parent) | **Update**: if the symbol was listed in the parent's Key Exports |
| Files table Cross-Dir Dependencies — Imports | **Update**: if this file added or removed cross-directory imports |
| Affected file's Exposed To (in the imported file's directory CODEMAP) | **Update**: add the new dependent if ≤5 total; if count exceeds 5, replace list with grep command and mark "foundational" |
| Task Guide tables | **Update**: only if the signature change enables a new typical task pattern or invalidates an existing one |
| File Dependencies table | **No update** — unless this file added/removed imports of same-directory files |
| `commit` field | **Update** in all modified CODEMAPs |

#### 3. Implementation Change (bug fix, internal refactor, algorithm tweak, parameter adjustment)

| Update Item | Action |
|---|---|
| All CODEMAP tables | **No update** — file purpose, exports, Domain, dependencies, and task mappings unchanged |
| `.analysis.md` Logical Sections | **Update** line ranges only if code shifted by >20 lines within a >1000-line file |
| `.analysis.md` Feature Index | **Update** only if the logical mapping between intents and code sections changed (not for mere line shifts) |
| `commit` field | **No update** needed |

#### 4. No Update Scenarios

| Scenario | Reason |
|---|---|
| Comment-only changes | No impact on any index information |
| Test file modifications (`tests/` directory) | Outside CODEMAP index scope |
| Config value changes (not changing which files import the config) | Import relationships unchanged, Cross-Dir Dependencies unaffected |

## Edge Cases

- **Monorepo with multiple packages**: Treat each package root as a semi-independent project. Generate a CODEMAP.md per package root, plus a top-level CODEMAP.md that indexes packages.
- **Very deep nesting (>5 levels)**: The drill-down navigation still works — each level adds one CODEMAP read. If a single directory has >200 files, split the file table into logical groups with subheadings.
- **Generated code directories** (e.g., `proto/gen/`, `graphql/generated/`): Include in CODEMAP with Domain = "Generated" or the appropriate domain, and a note "auto-generated, do not edit manually" in the function column. Agent knows to skip these for modification tasks.
- **No README and no project metadata files**: Synthesize the global context summary from the root directory's file list and first-level subdirectory names. Mark inferred content explicitly.
- **Files near the 1000-line threshold**: Use 1000 lines as a hard threshold. Files at 900-999 lines do not get analysis files — the regular CODEMAP entry is sufficient for navigation.
- **Cross-Dir Exposed To crossing the 5-file threshold**: When a previously ≤5 Exposed To grows to 6+, replace the static list with a grep command and mark "foundational." When a foundational file's dependents drop to ≤5 after refactoring, restore the static list.
- **Task Guide coverage gaps**: Not every possible task needs a Task Guide row. If a directory has no clear task-to-file patterns (e.g., a `utils/` directory with miscellaneous helpers), the Task Guide may have few or zero rows. Domain filtering via the Files table remains the fallback.
