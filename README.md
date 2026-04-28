# codemap-skill

**[English](#english)** | **中文（默认）**

---

为 Claude Code 及其他 AI 编程 Agent 设计的**层级化代码库导航索引** Skill。

在项目根目录和每个源码子目录下生成 `CODEMAP.md` 索引文件，以及针对超大文件（>1000 行）的 `<filename>.analysis.md` 深度分析伴生文件。Agent 通过逐层读取索引文件定位目标代码，而非盲目扫描整个代码库。

## 核心特性

- **层级化索引**：根目录 + 每个子目录各自生成 `CODEMAP.md`，包含精简目录结构、文件/子目录功能概要
- **关键导出表（Key Exports）**：目录级聚合导出符号，标注来源文件路径和行号（`L:<number>`），支持符号级快速定位
- **大文件深度分析**：超过 1000 行的源文件生成 `.analysis.md` 伴生文件，包含顶层符号表、类继承关系、逻辑分段行范围
- **并行 Sub-agent 生成**：支持多 sub-agent 并行生成索引，按代码行数贪心装箱均衡负载
- **双模式**：学习模式（一次生成，无需更新）/ 维护模式（基于 `git diff` 增量更新）
- **三层忽略规则**：内置默认 + `.gitignore` + 用户自定义
- **多语言输出**：CODEMAP 内容语言跟随用户提问语言
- **导航协议声明**：自动在 `CLAUDE.md` / `AGENTS.md` 中写入导航工作流规则

## 生成的文件

运行此 Skill 后，项目中会新增以下文件：

```
project-root/
├── CODEMAP.md                          # 根目录索引
├── CLAUDE.md (追加导航协议)             # 或 AGENTS.md
├── src/
│   ├── CODEMAP.md                      # src/ 目录索引
│   ├── models/
│   │   ├── CODEMAP.md                  # src/models/ 目录索引
│   │   └── large_model.py.analysis.md  # 大文件分析（若 >1000 行）
│   └── utils/
│       └── CODEMAP.md                  # src/utils/ 目录索引
└── tests/
    └── CODEMAP.md                      # tests/ 目录索引
```

## 安装

将 `SKILL.md` 复制到 Claude Code 的 skills 目录：

```bash
# 全局安装
mkdir -p ~/.claude/skills/codemap
cp SKILL.md ~/.claude/skills/codemap/SKILL.md
```

## 使用

在 Claude Code 会话中触发（以下任一方式）：

- `/codemap`
- 对 Claude 说 "帮我索引这个项目" / "map this codebase" / "generate codemap"

Skill 会依次询问：
1. 项目模式（学习 / 维护）
2. 是否启用并行 sub-agent（默认上限 3）
3. 额外忽略规则

然后自动扫描、生成所有 CODEMAP.md 和分析文件。

## 导航工作流

Agent 使用 CODEMAP 的方式：

```
读取根 CODEMAP.md → 锁定 1-3 个相关子目录
    → 并行读取子目录 CODEMAP.md → 锁定目标文件
        → 若有 .analysis.md 则先读分析文件定位行范围
            → 并行批量读取目标源文件
```

## 许可证

MIT

---

<a id="english"></a>

## English

A **hierarchical codebase navigation index** skill for Claude Code and other AI coding agents.

Generates `CODEMAP.md` index files in the project root and every source subdirectory, plus `<filename>.analysis.md` deep-analysis companion files for extra-large source files (>1000 lines). Agents navigate by reading index files layer by layer instead of scanning the entire codebase blindly.

### Features

- **Hierarchical indexing**: Root + per-subdirectory `CODEMAP.md` with simplified directory structure and file/subdirectory summaries
- **Key Exports table**: Directory-level aggregated export symbols with source file path and line number (`L:<number>`)
- **Large file deep analysis**: Files >1000 lines get a `.analysis.md` companion with top-level symbol table, class hierarchy, and logical section line ranges
- **Parallel sub-agent generation**: Multiple sub-agents with greedy bin-packing load balancing by code line count
- **Dual mode**: Learning (one-time generation) / Maintenance (incremental updates via `git diff`)
- **Three-layer ignore rules**: Built-in defaults + `.gitignore` + user custom
- **Multi-language output**: CODEMAP content language follows the user's request language
- **Navigation protocol**: Auto-declares navigation workflow rules in `CLAUDE.md` / `AGENTS.md`

### Generated Files

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

### Installation

Copy `SKILL.md` to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/codemap
cp SKILL.md ~/.claude/skills/codemap/SKILL.md
```

### Usage

Trigger in a Claude Code session:

- `/codemap`
- Or say: "index this project" / "map this codebase" / "generate codemap"

The skill will ask:
1. Project mode (Learning / Maintenance)
2. Enable parallel sub-agents? (default max: 3)
3. Additional ignore patterns?

Then it automatically scans and generates all CODEMAP.md and analysis files.

### Navigation Workflow

How agents use CODEMAPs:

```
Read root CODEMAP.md → identify 1-3 relevant subdirectories
    → read subdirectory CODEMAPs in parallel → locate target files
        → if .analysis.md exists, read it first for line ranges
            → batch-read all target source files in parallel
```

### License

MIT
