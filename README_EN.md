# codemap-skill

**[中文](README.md)** | **English**

---

A **hierarchical codebase navigation index** skill for Claude Code and other AI coding agents.

Generates `CODEMAP.md` index files in the project root and every source subdirectory, plus `<filename>.analysis.md` deep-analysis companion files for extra-large source files (>1000 lines). Agents navigate by reading index files layer by layer instead of scanning the entire codebase blindly.

## Features

- **Hierarchical indexing**: Root + per-subdirectory `CODEMAP.md` with simplified directory structure and file/subdirectory summaries
- **Key Exports table**: Directory-level aggregated export symbols with source file path and line number (`L:<number>`)
- **Large file deep analysis**: Files >1000 lines get a `.analysis.md` companion with top-level symbol table, class hierarchy, and logical section line ranges
- **Parallel sub-agent generation**: Multiple sub-agents with greedy bin-packing load balancing by code line count
- **Dual mode**: Learning (one-time generation) / Maintenance (incremental updates via `git diff`)
- **Three-layer ignore rules**: Built-in defaults + `.gitignore` + user custom
- **Multi-language output**: CODEMAP content language follows the user's request language
- **Navigation protocol**: Auto-declares navigation workflow rules in `CLAUDE.md` / `AGENTS.md`

## Generated Files

After running this skill, the following files are added to the project:

```
project-root/
├── CODEMAP.md                          # Root index
├── CLAUDE.md (appended navigation protocol)
├── src/
│   ├── CODEMAP.md                      # src/ index
│   ├── models/
│   │   ├── CODEMAP.md                  # src/models/ index
│   │   └── large_model.py.analysis.md  # Large file analysis (if >1000 lines)
│   └── utils/
│       └── CODEMAP.md                  # src/utils/ index
└── tests/
    └── CODEMAP.md                      # tests/ index
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

How agents use CODEMAPs:

```
Read root CODEMAP.md → identify 1-3 relevant subdirectories
    → read subdirectory CODEMAPs in parallel → locate target files
        → if .analysis.md exists, read it first for line ranges
            → batch-read all target source files in parallel
```

## License

MIT
