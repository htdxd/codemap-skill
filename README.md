# codemap-skill

**中文** | **[English](README_EN.md)**

---

为 Claude Code 及其他 AI 编程 Agent 设计的**层级化代码库导航索引** Skill。

在项目根目录和每个源码子目录下生成 `CODEMAP.md` 索引文件，以及针对超大文件（>1000 行）的 `<filename>.analysis.md` 深度分析伴生文件。采用 Task Guide → Domain 过滤 → 依赖安全网的**约束式导航策略**，从源头防止 Agent 扩散性读取无关文件。

## 核心特性

### 索引与导航
- **层级化索引**：根目录 + 每个子目录各自生成 `CODEMAP.md`，包含精简目录结构、文件/子目录功能概要
- **Domain 功能域标注**：每个文件/子目录标注所属功能域（Auth/User Data/API 等），Agent 按域过滤排除无关文件
- **Task Guide 任务路由**：预定义任务类型→目标文件的精确映射（如"新增 Loss 函数 → `losses.py`"），消除 Agent 自行语义扩展的不确定性
- **Also Check 跨目录关联**：Task Guide 中附带经验性跨目录关联文件列表（非全量依赖图），确保关联文件不被遗漏

### 依赖关系管理（文件级 + 目录级）
- **File Dependencies（同目录内）**：IMPORT → EXPOSED_TO 双向链表，仅在接口契约需要理解或公开签名变更时触发读取，禁止无条件链式遍历
- **Cross-Dir Dependencies（跨目录）**：每个文件标注其跨目录 Imports 和 Exposed To。Exposed To ≤5 精确列出文件路径；>5 则降级为 grep 动态查询指令（"foundational"），防止基础文件的静态列表过时
- **目录级 Dependencies**：标注 internal/external 以区分视野范围内外的依赖

### 大文件精确定位
- **Feature Index（功能→行范围索引）**：`.analysis.md` 中新增意图→行范围映射表，Agent 直接匹配任务关键词定位需要读取的代码段
- **Logical Sections（逻辑分段）**：作为 Feature Index 未匹配时的回退方案
- **顶层符号表 + 类继承关系**：快速了解文件结构

### Agent 行为约束
- **Two-Stage Read Protocol**：Stage 1 仅读取 Task Guide + Domain 匹配的文件；Stage 2 仅在分析证明确实需要时补充读取
- **依赖读取条件门控**：Imports 仅在接口契约理解需要时读；Exposed To 仅在公开签名/语义变更时读；>5 foundational 文件先 grep 再按 Domain 过滤
- **自动写入导航协议**：在 `CLAUDE.md` / `AGENTS.md` 中写入 10 条增强约束规则 + 决策树更新规则

### 工程特性
- **并行 Sub-agent 生成**：按代码行数贪心装箱均衡负载
- **双模式**：学习模式（一次生成）/ 维护模式（基于 `git diff` 增量更新 + 决策树自主判断更新范围）
- **三层忽略规则**：内置默认 + `.gitignore` + 用户自定义
- **多语言输出**：CODEMAP 内容语言跟随用户提问语言

## 生成的文件

运行此 Skill 后，项目中会新增以下文件：

```
project-root/
├── CODEMAP.md                          # 根目录索引（含 Task Guide + Domain + Dependencies）
├── CLAUDE.md (追加导航协议)             # 或 AGENTS.md
├── src/
│   ├── CODEMAP.md                      # src/ 索引（含 Task Guide + Domain + Dependencies）
│   ├── models/
│   │   ├── CODEMAP.md                  # 含 Files(Domain+Cross-Dir Deps) + File Deps + Task Guide
│   │   └── large_model.py.analysis.md  # 大文件分析（含 Feature Index + Logical Sections）
│   └── utils/
│       └── CODEMAP.md
└── tests/
    └── CODEMAP.md
```

## 安装

将 `SKILL.md` 复制到 Claude Code 的 skills 目录：

```bash
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

Agent 使用 CODEMAP 的**约束式读取流程**：

```
Step 1: 读取根 CODEMAP → Task Guide 匹配当前任务类型
        ↓ 命中 → Target + Also Check 直接给出初始文件集
        ↓ 未命中 → Domain 过滤定位目标目录

Step 2: 并行读取目标子目录 CODEMAP
        ↓ Task Guide 精确定位文件
        ↓ Domain 列排除非匹配文件
        ↓ File Dependencies 仅条件性触发（接口变更/契约理解）

Step 3: 处理跨目录依赖
        ↓ Also Check → 直接读取（经验验证）
        ↓ Exposed To ≤5 → 仅在签名/语义变更时读
        ↓ Exposed To >5 (foundational) → grep → Domain 过滤 → 读

Step 4: 编译最终文件集 → 去重

Step 5: 大文件先读 .analysis.md Feature Index → 定位行范围

Step 6: 并行批量读取所有目标源文件（使用 offset/limit 精准读取大文件行范围）

Step 7 (Stage 2): 仅在分析证明确实需要时补充读取更多文件
```

**核心效率原则**：阅读 3-4 个 CODEMAP（~200 行）即可精确定位 5-8 个源文件。Task Guide + Domain 过滤在读取任何源码之前缩小候选集。Feature Index 对大文件实现行级精准读取。依赖关系信息是**安全网而非读取指令**——仅在特定条件触发下才引发额外读取。

## 更新规则（维护模式）

Agent 在修改代码后，通过以下**决策树**自主判断是否需要更新 CODEMAP：

1. **结构变更**（文件/目录增删移）→ 重新生成受影响目录的全部 CODEMAP 内容
2. **接口变更**（公开符号签名/名称变更）→ 更新 Key Exports、Cross-Dir Dependencies（含阈值穿越处理），向上传播
3. **实现变更**（bug fix / 内部重构 / 参数调整）→ 无需更新 CODEMAP（仅在大文件行偏移 >20 行时更新 Logical Sections 行号）

## 许可证

MIT
