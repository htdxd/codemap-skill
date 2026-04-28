---
name: codemap
description: Generate and navigate CODEMAP.md index files for large codebases. Creates hierarchical per-directory CODEMAP.md files containing simplified directory structure, file/subdirectory function summaries, and directory-level key exports with source location annotations. Use this skill whenever the user wants to index a codebase for efficient agent navigation, generate CODEMAP files, or when the user mentions "codemap", "code map", "codebase index", "project index", "codebase navigation", "generate navigation", or wants to understand a large project structure. Also triggers when the user asks to update or refresh existing CODEMAP files. Even if the user just says "index this project" or "map this codebase", use this skill.
---

# CODEMAP — Codebase Navigation Index Generator

Generate hierarchical `CODEMAP.md` files that serve as a structured navigation index for large codebases. Each directory gets its own CODEMAP.md containing a simplified directory tree, file/subdirectory summaries, and directory-level key exports with source location annotations. For extra-large source files (>1000 lines), generate a companion deep-analysis file. Agent navigation follows a layered lazy-loading strategy: read index first, drill down, then batch-read only the source files actually needed.

## Language Rule

All generated CODEMAP.md files and analysis.md files MUST be written in the same language as the user's request. If the user asks in Chinese, all summaries, purpose descriptions, function descriptions, and section headings (except code identifiers and file names) are written in Chinese. If the user asks in English, write in English. Code identifiers (`ClassName`, `function_name`, file paths) always retain their original form regardless of language.

## Two Operating Modes

Ask the user **before** doing anything else:

```
Q1: Project mode?
    A) Learning — read-only study, no code changes expected
    B) Maintenance — active development, bugs/features/refactors

Q2: Enable parallel sub-agents for faster generation?
    A) Yes → how many max? (default: 3)
    B) No (single-agent serial generation)

Q3: Additional ignore patterns beyond defaults? (optional, Enter to skip)
```

Mode affects:

| Aspect | Learning | Maintenance |
|---|---|---|
| CODEMAP frontmatter | `mode: learning` | `mode: maintenance`, includes `commit: <hash>` |
| Update strategy | One-time generation, no updates | Incremental via `git diff`, regenerate changed dirs only |
| Content tone | May include brief design-intent notes | Concise, purely navigational |
| CLAUDE.md clause | Declares CODEMAP existence + read rules | Additionally declares: update CODEMAP after code changes |

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
2. Combine with the filtered directory topology from Phase 1 to produce a **project global context summary**. Keep it concise — summarize what the project does, its high-level architecture, and major modules. Do not pad with installation instructions, badges, or changelogs.

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
- Instruction: for each assigned directory and all its subdirectories, read source files and generate CODEMAP.md files following the format spec below

**Each sub-agent produces:**
- One CODEMAP.md per directory it is responsible for
- One `<filename>.analysis.md` per large file (>1000 lines) in its scope

### Phase 2 (alternative): Serial Generation (if sub-agents disabled)

Main agent processes first-level subdirectories one by one in descending line-count order. For each directory: read all source files within it, generate CODEMAP.md for it and all its subdirectories. Generate deep-analysis files for any source file exceeding 1000 lines.

### Phase 3: Root Assembly

1. Main agent reads the CODEMAP.md of each first-level subdirectory (summary line only, not full content).
2. Generate root-level CODEMAP.md: global context summary + simplified directory tree + subdirectory table + root file table + directory-level key exports with source annotations.
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

> <Project global context summary: what it does, architecture overview. Concise, based on README + directory structure. Any inferred content marked as "inferred, verify against code.">

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

| Directory | Purpose |
|---|---|
| `src/` | Core source code: models, data processing, utilities |
| `configs/` | Configuration files for training, deployment, and environments |
| `tests/` | Test suites: unit, integration, e2e |

## Files

| File | Function |
|---|---|
| `main.py` | Application entry point, CLI argument parsing |
| `setup.py` | Package build and installation configuration |
| `Makefile` | Common development task shortcuts (lint, test, build) |
```

### Subdirectory-Level CODEMAP.md

```markdown
---
mode: learning | maintenance
commit: abc1234f          # maintenance mode only
---

# CODEMAP — src/models/

> <One-paragraph summary of this directory's role in the project, 2-4 sentences>

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

| Directory | Purpose |
|---|---|
| `backbones/` | Backbone network implementations (ResNet, ViT, etc.) |
| `heads/` | Task-specific heads (classification, detection, segmentation) |

## Files

| File | Function |
|---|---|
| `base_model.py` | Abstract base class for all models, defines forward pass and weight loading interfaces |
| `registry.py` | Model registry, instantiates models by name string |
| `losses.py` | Loss function collection (Focal, Dice, etc.) |
| `__init__.py` | Module public interface |
```

### Format Rules

1. **Directory Structure section**: Show only directory names (no files), depth limited to 2 levels below current directory. Purpose: give the agent a quick spatial overview.
2. **Key Exports section**: A table listing directory-level aggregate symbols — classes, functions, constants that callers from other directories would use. Each entry includes the **source file path** (relative to current directory) and **line number** (`L:<number>`). Sort by importance descending: most architecturally significant symbols first. This is a directory-level view — do NOT create per-file export lists.
3. **Subdirectories table**: One row per immediate subdirectory. Purpose column: one sentence, 10-25 words.
4. **Files table**: One row per file in the current directory (not recursive). Function column: one sentence, 10-30 words. Skip `__init__.py` if it only re-exports (mention in Key Exports instead). For large files (>1000 lines), append `→ see <filename>.analysis.md` in the function column.
5. **Summary paragraph** (below the heading): describe this directory's responsibility within the project. Reference the project global context to explain the role. 2-4 sentences. When information is insufficient for certainty, state a reasonable inference and mark it: "inferred, verify against code."
6. **Leaf directories** (no subdirectories): omit the "Directory Structure" and "Subdirectories" sections entirely.

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
4. **Logical Sections**: Divide the file into coherent blocks by line range. Purpose: let the agent read only the relevant section (e.g., `Read file_path offset=198 limit=213`) instead of the entire file.
5. Keep the analysis file concise — it is a structural map, not documentation. No code snippets, no API signatures, no implementation details.

## CLAUDE.md / AGENTS.md Declaration

Append the following block to the project's `CLAUDE.md` or `AGENTS.md` (create the file if neither exists; prefer `CLAUDE.md`). These are **default navigation conventions** — the user may override or relax any rule if they have a specific workflow preference.

```markdown
## CODEMAP Navigation Protocol

This project contains `CODEMAP.md` index files in the root and each source subdirectory. These files provide a structured navigation index for efficient codebase exploration.

### Reading Rules (Default)

1. **Start from root `CODEMAP.md`** when exploring the codebase or searching for code related to a task.
2. **Layer-by-layer drill-down**: Read root CODEMAP → identify relevant subdirectories → read their CODEMAPs → identify target files → batch-read source files.
3. **Prefer index over direct reads**: Consult CODEMAP.md first to locate the right files before reading source code. Exception: when the user provides an exact file path, read it directly.
4. **Batch parallel reads**: After identifying all target files through CODEMAP navigation, read them all in one parallel batch — not one by one.
5. **Key Exports shortcut**: When searching for a specific symbol (class, function, constant), scan the "Key Exports" table in each CODEMAP to locate which directory and file owns it, along with the exact line number.
6. **Large file navigation**: When a file has a companion `<filename>.analysis.md`, read the analysis file first to identify the relevant line range, then read only that range from the source file using offset/limit parameters.
```

For **maintenance mode**, additionally append:

```markdown
### Update Rules

- After modifying, adding, or deleting source files, regenerate the CODEMAP.md for the affected directory and all its ancestor directories up to root.
- Use `git diff <commit-in-codemap-frontmatter>..HEAD` to detect which directories need updates.
- When regenerating, preserve the existing format and only update changed entries.
- If a large file (>1000 lines) was modified, also regenerate its `<filename>.analysis.md`.
```

## Navigation Workflow (for agents reading the codebase)

This is how an agent should use CODEMAP.md files when performing any code reading or search task:

```
Step 1: Read root CODEMAP.md
        → From the summary, directory table, and key exports,
          identify which 1-3 subdirectories are relevant to the task.
        → If the Key Exports table directly shows the target symbol
          with file path and line number, skip to Step 3.

Step 2: Read those subdirectories' CODEMAP.md files IN PARALLEL
        → Further narrow down to specific files or deeper subdirectories.
        → If deeper subdirectories exist, repeat this step one level down.

Step 3: Compile the final list of target source files.
        → For any target file that has a companion .analysis.md,
          read the analysis file first to identify the relevant line range.

Step 4: Read ALL target source files IN ONE PARALLEL BATCH.
        → For files with analysis data, use offset/limit to read
          only the relevant sections instead of the full file.
        → This is the only step where actual source code is read.
```

**Efficiency principle**: Minimize total tool calls and tokens consumed. Each CODEMAP.md is typically 30-80 lines. Reading 3-4 CODEMAPs (~200 lines) to locate 5-8 source files is far cheaper than scanning the entire codebase. For large files, the analysis companion further reduces token consumption by enabling targeted line-range reads.

## Incremental Update (Maintenance Mode Only)

When the user asks to update CODEMAPs after code changes:

1. Read `commit` from root CODEMAP.md frontmatter.
2. Run `git diff --name-only <old-commit>..HEAD` to get changed file list.
3. Identify affected directories (any directory containing a changed file).
4. For each affected directory, re-read its source files and regenerate its CODEMAP.md.
5. For any changed file exceeding 1000 lines, regenerate its `<filename>.analysis.md`.
6. Propagate upward: regenerate parent CODEMAPs if a child directory's summary or key exports changed.
7. Update the `commit` field in all regenerated CODEMAPs to current HEAD.

## Edge Cases

- **Monorepo with multiple packages**: Treat each package root as a semi-independent project. Generate a CODEMAP.md per package root, plus a top-level CODEMAP.md that indexes packages.
- **Very deep nesting (>5 levels)**: The drill-down navigation still works — each level adds one CODEMAP read. If a single directory has >200 files, split the file table into logical groups with subheadings.
- **Generated code directories** (e.g., `proto/gen/`, `graphql/generated/`): Include in CODEMAP with a note "auto-generated, do not edit manually" in the function column. Agent knows to skip these for modification tasks.
- **No README and no project metadata files**: Synthesize the global context summary from the root directory's file list and first-level subdirectory names. Mark inferred content explicitly.
- **Files near the 1000-line threshold**: Use 1000 lines as a hard threshold. Files at 900-999 lines do not get analysis files — the regular CODEMAP entry is sufficient for navigation.
