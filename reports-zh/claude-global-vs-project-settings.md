# Claude Code: 全局功能 vs 项目级功能

全面对比 Claude Code 中哪些功能是全局独有的（`~/.claude/`），哪些同时具有全局和项目级（`.claude/`）。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## 目录

1. [概览](#概览)
2. [全局独有功能](#全局独有功能)
3. [双重作用域功能](#双重作用域功能)
4. [设置优先级](#设置优先级)
5. [目录结构对比](#目录结构对比)
6. [任务系统](#任务系统)
7. [Agent 团队](#agent-团队)
8. [设计原则](#设计原则)
9. [来源](#来源)

---

## 概览

Claude Code 使用**作用域层级**，某些功能同时存在于全局（`~/.claude/`）和项目级（`.claude/`），而有些功能则是全局独有的。设计原则：属于**个人状态**或**跨项目协调**的内容放在全局；属于**团队可共享项目配置**的内容放在项目级。

- `~/.claude/` 是你的**用户级主目录**（全局，所有项目）
- 仓库内的 `.claude/` 是你的**项目级主目录**（仅对该项目作用域）

---

## 全局独有功能

这些**仅**存在于 `~/.claude/` 下，不能按项目作用域设置：

| 功能 | 位置 | 用途 |
|---------|----------|---------|
| **任务** | `~/.claude/tasks/` | 跨会话和 agent 的持久任务列表 |
| **Agent 团队** | `~/.claude/teams/` | 多 agent 协调配置（实验性，2026 年 2 月） |
| **自动记忆** | `~/.claude/projects/<hash>/memory/` | Claude 为每个项目自撰的学习内容（个人的，永不共享） |
| **凭据和 OAuth** | 系统密钥链 + `~/.claude.json` | API 密钥、OAuth 令牌（绝不会在项目文件中） |
| **快捷键** | `~/.claude/keybindings.json` | 自定义键盘快捷键 |
| **MCP 用户服务器** | `~/.claude.json`（`mcpServers` 键） | 所有项目的个人 MCP 服务器 |
| **偏好设置/缓存** | `~/.claude.json` | 主题、模型、输出风格、会话状态 |

---

## 双重作用域功能

这些存在于两个级别，**项目级优先于全局**：

| 功能 | 全局（`~/.claude/`） | 项目级（`.claude/`） | 优先级 |
|---------|----------------------|---------------------|------------|
| **CLAUDE.md** | `~/.claude/CLAUDE.md` | `./CLAUDE.md` 或 `.claude/CLAUDE.md` | 项目覆盖全局 |
| **设置** | `~/.claude/settings.json` | `.claude/settings.json` + `.claude/settings.local.json` | 项目 > 全局 |
| **规则** | `~/.claude/rules/*.md` | `.claude/rules/*.md` | 项目覆盖 |
| **Agent/子 agent** | `~/.claude/agents/*.md` | `.claude/agents/*.md` | 项目覆盖 |
| **命令** | `~/.claude/commands/*.md` | `.claude/commands/*.md` | 两者均可用 |
| **技能** | `~/.claude/skills/` | `.claude/skills/` | 两者均可用 |
| **Hooks** | `~/.claude/hooks/` | `.claude/hooks/` | 两者都执行 |
| **MCP 服务器** | `~/.claude.json`（用户作用域） | `.mcp.json`（项目作用域） | 三个作用域：local > project > user |

---

## 设置优先级

用户可写的设置按以下优先级覆盖（高到低）：

| 优先级 | 位置 | 作用域 | 版本控制 | 用途 |
|----------|----------|-------|-----------------|---------|
| 1 | 命令行标志 | 会话 | 不适用 | 单次会话覆盖 |
| 2 | `.claude/settings.local.json` | 项目 | 否（git-ignored） | 个人项目特定 |
| 3 | `.claude/settings.json` | 项目 | 是（已提交） | 团队共享设置 |
| 4 | `~/.claude/settings.local.json` | 用户 | 不适用 | 个人全局覆盖 |
| 5 | `~/.claude/settings.json` | 用户 | 不适用 | 全局个人设置 |

策略层：`managed-settings.json` 由组织实施，不能被本地文件覆盖。

**重要**：`deny` 规则具有最高安全优先级，不能被低优先级的 allow/ask 规则覆盖。

---

## 目录结构对比

### 全局作用域（`~/.claude/`）

```
~/.claude/
├── settings.json              # 用户级设置（所有项目）
├── settings.local.json        # 个人覆盖
├── CLAUDE.md                  # 用户记忆（所有项目）
├── agents/                    # 用户子 agent（所有项目可用）
│   └── *.md
├── rules/                     # 用户级模块化规则
│   └── *.md
├── commands/                  # 用户级命令
│   └── *.md
├── skills/                    # 用户级技能
│   └── */SKILL.md
├── tasks/                     # 全局独有：任务列表
│   └── {task-list-id}/
├── teams/                     # 全局独有：Agent 团队配置
│   └── {team-name}/
│       └── config.json
├── projects/                  # 全局独有：每项目自动记忆
│   └── {project-hash}/
│       └── memory/
│           ├── MEMORY.md
│           └── *.md
├── keybindings.json           # 全局独有：快捷键
└── hooks/                     # 用户级 hooks
    ├── scripts/
    └── config/

~/.claude.json                 # 全局独有：MCP 服务器、OAuth、偏好设置、缓存
```

### 项目作用域（`.claude/`）

```
.claude/
├── settings.json              # 团队共享设置
├── settings.local.json        # 个人项目覆盖（git-ignored）
├── CLAUDE.md                  # 项目记忆（替代 ./CLAUDE.md）
├── agents/                    # 项目子 agent
│   └── *.md
├── rules/                     # 项目级模块化规则
│   └── *.md
├── commands/                  # 自定义斜杠命令
│   └── *.md
├── skills/                    # 自定义技能
│   └── {skill-name}/
│       ├── SKILL.md
│       └── supporting-files/
├── hooks/                     # 项目级 hooks
│   ├── scripts/
│   └── config/
└── plugins/                   # 安装的插件

.mcp.json                      # 项目作用域 MCP 服务器（仓库根目录）
```

---

## 任务系统

**Claude Code v2.1.16**（2026 年 1 月 22 日）引入，替换了已弃用的 TodoWrite 系统。

### 存储

任务存储在本地文件系统的 `~/.claude/tasks/`（不是云数据库）。这使得任务状态可审计、可进行版本控制、可崩溃恢复。

### 工具

| 工具 | 用途 |
|------|---------|
| **TaskCreate** | 创建新任务，含 `subject`、`description` 和 `activeForm` |
| **TaskGet** | 通过 ID 检索特定任务的详细信息 |
| **TaskUpdate** | 更改状态、设置所有者、添加依赖或删除 |
| **TaskList** | 列出所有任务及其当前状态 |

### 任务生命周期

```
pending  →  in_progress  →  completed
```

### 依赖管理

任务可以通过 `addBlockedBy`/`addBlocks` 阻塞其他任务，创建依赖图来防止过早执行。

### 多会话协作

```bash
CLAUDE_CODE_TASK_LIST_ID=my-project-tasks claude
```

共享同一 ID 的所有会话都能实时看到任务更新，支持并行工作流和会话恢复。

### 与旧 Todos 的关键区别

| 功能 | 旧 Todos | 新 Tasks |
|---------|-----------|-----------|
| 作用域 | 单会话 | 跨会话、跨 agent |
| 依赖 | 无 | 完整依赖图 |
| 存储 | 仅内存 | 文件系统（`~/.claude/tasks/`） |
| 持久性 | 会话结束时丢失 | 重启和崩溃后存活 |
| 多会话 | 不可能 | 通过 `CLAUDE_CODE_TASK_LIST_ID` |

---

## Agent 团队

**2026 年 2 月 5 日**宣布为实验性功能。Agent 团队允许多个 Claude Code 会话协调共享任务。

### 启用

```json
// 在 ~/.claude/settings.json 中
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### 配置

团队配置位于 `~/.claude/teams/{team-name}/`，支持以下模式：

| 模式 | 描述 | 要求 |
|------|-------------|--------------|
| **进程内**（默认） | 所有队友都在你的终端内部运行 | 无 |
| **分割窗格** | 每个队友都有自己的窗格 | tmux 或 iTerm2（不支持 VS Code 终端） |

---

## 设计原则

全局独有 vs 双重作用域拆分遵循清晰模式：

| 类别 | 作用域 | 理由 |
|----------|-------|-----------|
| **协调状态**（任务、团队） | 全局独有 | 需要在任何单个项目之外持续存在 |
| **安全状态**（凭据、OAuth） | 全局独有 | 防止意外提交到版本控制 |
| **个人学习**（自动记忆） | 全局独有 | 用户特定，不可团队共享 |
| **输入偏好**（快捷键） | 全局独有 | 用户肌肉记忆，非项目特定 |
| **配置**（设置、规则、agent） | 两个级别 | 团队需要共享项目特定行为 |
| **工作流定义**（命令、技能） | 两个级别 | 可以是个人或团队共享 |

自动记忆（`~/.claude/projects/<hash>/memory/`）是一个显著混合体：它**关于**特定项目，但存储在全局，因为它代表的是个人学习，而非团队可共享的配置。

---

## 来源

- [Claude Code Settings 文档](https://code.claude.com/docs/en/settings)
- [Orchestrate Teams of Claude Code Sessions](https://code.claude.com/docs/en/agent-teams)
- [What are Tasks in Claude Code - ClaudeLog](https://claudelog.com/faqs/what-are-tasks-in-claude-code/)
- [Claude Code Task Management - ClaudeFast](https://claudefa.st/blog/guide/development/task-management)
- [Claude Code Tasks Update - VentureBeat](https://venturebeat.com/orchestration/claude-codes-tasks-update-lets-agents-work-longer-and-coordinate-across)
- [Where Are Claude Code Global Settings - ClaudeLog](https://claudelog.com/faqs/where-are-claude-code-global-settings/)
- [Claude Opus 4.6 Agent Teams - VentureBeat](https://venturebeat.com/technology/anthropics-claude-opus-4-6-brings-1m-token-context-and-agent-teams-to-take)
- [How to Set Up Claude Code Agent Teams (Full Walkthrough) - r/ClaudeCode](https://www.reddit.com/r/ClaudeCode/comments/1qz8tyy/how_to_set_up_claude_code_agent_teams_full/)
- [Anthropic replaced Claude Code's old 'Todos' with Tasks - r/ClaudeAI](https://www.reddit.com/r/ClaudeAI/comments/1qkjznp/anthropic_replaced_claude_codes_old_todos_with/)
