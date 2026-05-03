# codemap-skill

**[中文](README.md)** | **English**

---

A **hierarchical codebase navigation index** skill for Claude Code and other AI coding agents.

Generates `CODEMAP.md` index files in the project root and every source subdirectory, plus `<filename>.analysis.md` deep-analysis companion files for extra-large source files (>1000 lines). Uses a **constraint-based navigation strategy** — Task Guide → Domain filter → dependency safety net — to prevent agents from reading irrelevant files.

## Features

### Index & Navigation
- **Hierarchical indexing**: Root + per-subdirectory `CODEMAP.md` with simplified directory structure and file/subdirectory summaries
- **Domain annotations**: Each file/subdirectory tagged with its functional domain (Auth/User Data/API, etc.), enabling agents to filter out irrelevant files by domain
- **Task Guide routing**: Predefined task-type → target-file mapping (e.g., "Add loss function → `losses.py`"), eliminating the agent's speculative semantic expansion
- **Also Check cross-directory associations**: Empirically validated cross-directory file references in Task Guide (not exhaustive dependency graphs), ensuring related files are not missed

### Dependency Management (File-Level + Directory-Level)
- **File Dependencies (within directory)**: IMPORT → EXPOSED_TO bidirectional table. Triggers additional reads ONLY when interface contracts need understanding or public signatures change — no unconditional chain-reading
- **Cross-Dir Dependencies**: Each file lists its cross-directory Imports and Exposed To. Exposed To ≤5 lists exact paths; >5 degrades to a grep command ("foundational"), preventing stale static lists for heavily-depended-on files
- **Directory-level Dependencies**: Marked internal/external to distinguish in-scope vs out-of-scope dependencies

### Large File Precision
- **Feature Index (intent → line range)**: `.analysis.md` now includes intent-to-line-range mapping, allowing agents to match task keywords and jump directly to relevant code sections
- **Logical Sections**: Fallback when no Feature Index row matches
- **Top-Level Symbols table + Class Hierarchy**: Quick file structure overview

### Agent Behavior Constraints
- **Two-Stage Read Protocol**: Stage 1 reads only Task Guide + Domain matched files; Stage 2 supplements only when analysis proves additional files are needed
- **Dependency read gating**: Imports read only for interface contract understanding; Exposed To read only for public signature/semantics changes; >5 foundational files use grep + Domain filter first
- **Auto-declared navigation protocol**: Writes 10 enhanced constraint rules + decision tree update rules into `CLAUDE.md` / `AGENTS.md`

### Engineering
- **Parallel sub-agent generation**: Greedy bin-packing load balancing by code line count
- **Dual mode**: Learning (one-time) / Maintenance (incremental via `git diff` + autonomous decision tree)
- **Three-layer ignore rules**: Built-in defaults + `.gitignore` + user custom
- **Multi-language output**: CODEMAP content language follows user's request language

## Generated Files

After running this skill, the following files are added to the project:

```
project-root/
├── CODEMAP.md                          # Root index (with Task Guide + Domain + Dependencies)
├── CLAUDE.md (appended navigation protocol)
├── src/
│   ├── CODEMAP.md                      # src/ index (with Task Guide + Domain + Dependencies)
│   ├── models/
│   │   ├── CODEMAP.md                  # Files(Domain+Cross-Dir Deps) + File Deps + Task Guide
│   │   └── large_model.py.analysis.md  # Large file analysis (Feature Index + Logical Sections)
│   └── utils/
│       └── CODEMAP.md
└── tests/
    └── CODEMAP.md
```

## Installation

Copy `SKILL.md` to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/codemap
cp SKILL.md ~/.claude/skills/codemap/SKILL.md
```

## Usage

Trigger in a Claude Code session:

- `/codemap`
- Or say: "index this project" / "map this codebase" / "generate codemap"

The skill will ask:
1. Project mode (Learning / Maintenance)
2. Enable parallel sub-agents? (default max: 3)
3. Additional ignore patterns?

Then it automatically scans and generates all CODEMAP.md and analysis files.

## Navigation Workflow

How agents use CODEMAPs with the **constraint-based read protocol**:

```
Step 1: Read root CODEMAP → Task Guide match against current task
        ↓ Hit → Target + Also Check directly define the initial file set
        ↓ Miss → Domain filter to identify target directories

Step 2: Read target subdirectory CODEMAPs IN PARALLEL
        ↓ Task Guide for precise file targeting
        ↓ Domain column excludes non-matching files
        ↓ File Dependencies consulted ONLY conditionally (interface change / contract check)

Step 3: Resolve cross-directory dependencies
        ↓ Also Check → read directly (empirically validated)
        ↓ Exposed To ≤5 → read only on signature/semantics change
        ↓ Exposed To >5 (foundational) → grep → Domain filter → read

Step 4: Compile final target file set → deduplicate

Step 5: Large files: read .analysis.md Feature Index first → locate line ranges

Step 6: Batch-read ALL target source files IN PARALLEL (offset/limit for large files)

Step 7 (Stage 2): Supplement only when analysis proves additional files are needed
```

**Efficiency principle**: Reading 3-4 CODEMAPs (~200 lines) precisely locates 5-8 source files. Task Guide + Domain filter narrows the candidate set before any source code is read. Feature Index enables line-level precision for large files. Dependency information is a **safety net, not a reading mandate** — it triggers additional reads only under specific conditions.

## Update Rules (Maintenance Mode)

After modifying code, the agent autonomously decides via this **decision tree**:

1. **Structural change** (file/directory add/delete/move/rename) → Regenerate affected directory's full CODEMAP
2. **Interface change** (public symbol signature/name changed) → Update Key Exports, Cross-Dir Dependencies (with threshold crossing handling), propagate upward
3. **Implementation change** (bug fix / internal refactor / param tweak) → No CODEMAP update (only adjust Logical Sections line ranges if offset >20 lines in large files)

## License

MIT
