# Agent vs Command vs Skill —— 何时使用什么

Claude Code 中三种扩展机制的对比：子 agent、命令和技能。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

![Slash 菜单显示 time-skill、time-command 和 time-agent](assets/agent-command-skill-1.jpg)

---

## 概览

| | Agent | Command | Skill |
|---|---|---|---|
| **位置** | `.claude/agents/<name>.md` | `.claude/commands/<name>.md` | `.claude/skills/<name>/SKILL.md` |
| **上下文** | 独立的子 agent 进程 | 内联（主对话） | 内联（主对话） |
| **用户可调用** | 无 `/` 菜单 —— 由 Claude 调用或用工具调用它 | /command-name`）命令发起 | 是 —— `/skill-name`（但 `user-invocable: false` 时除外 |
| **Claude 自动调用** | 是 —— 通过 `description` 字段 | 否 | 是 —— 通过 `description` 字段（除非 `disable-model-invocation: true`） |
| **接受参数** | 通过 `prompt` 参数 | `$ARGUMENTS`、`$0`、`$1` | `$ARGUMENTS`、`$0`、`$1` |
| **动态上下文注入** | 否 | 是 —— `` !`command` `` | 是 —— `` !`command` `` |
| **拥有独立上下文窗口** | 是 —— 隔离的 | 否 —— 共享主窗口 | 否 —— 共享主窗口（除非 `context: fork`） |
| **模型覆盖** | `model:` frontmatter | `model:` frontmatter | `model:` frontmatter |
| **工具限制** | `tools:` / `disallowedTools:` | `allowed-tools:` | `allowed-tools:` |
| **Hooks** | `hooks:` frontmatter | — | `hooks:` frontmatter |
| **记忆** | `memory:` frontmatter（user/project/local） | — | — |
| **预加载技能** | 是 —— `skills:` frontmatter | — | — |
| **MCP 服务器** | `mcpServers:` frontmatter | — | — |

---

## 何时使用每种

### 使用 Agent 的场景：

- 任务是**自主且多步骤的** —— agent 需要探索、决策并在不需持续指导的情况下行动
- 你需要**上下文隔离** —— 工作不应污染主对话窗口
- agent 需要跨会话的**持久记忆**（例如学习模式的代码审查员）
- 你想通过技能**预加载领域知识**，而又不弄乱主上下文
- 任务受益于**后台运行**或**git worktree**
- 你需要**工具限制**或**不同的权限模式**（例如 `acceptEdits`、`plan`）

**示例**：`weather-agent` —— 使用预加载的 `weather-fetcher` 技能自主获取天气数据，在受限工具的独立上下文中运行。

### 使用 Command 的场景：

- 你需要**用户发起的入口点** —— 用户显式触发的一个工作流
- 工作流涉及**编排**其他 agent 或技能
- 你想**保持上下文精简** —— 命令内容在用户触发之前不会注入会话上下文

**示例**：`weather-orchestrator` —— 用户触发它，它会询问 C/F 偏好，然后调用 agent，接着调用 SVG 技能。

### 使用 Skill 的场景：

- 你想让**Claude 根据用户意图自动调用** —— 技能描述注入会话上下文，用于语义匹配
- 任务是**可复用的过程**，可以从多个地方调用（命令、agent 或 Claude 本身）
- 你需要**agent 预加载** —— 将领域知识在启动时注入到特定 agent 中

**示例**：`weather-svg-creator` —— 当用户要求天气卡片时，Claude 自动调用它；也可从命令中调用。

---

## Command → Agent → Skill 架构

本仓库展示了一个分层编排模式：

```
用户触发 /command
    ↓
命令编排工作流
    ↓
命令调用 Agent（独立上下文，自主执行）
    ↓
Agent 使用预加载的 Skill（领域知识）
    ↓
命令调用 Skill（内联，用于输出生成）
```

**具体示例** —— 天气系统：

```
/weather-orchestrator（命令 —— 入口点，询问 C/F）
    ↓
weather-agent（agent —— 自主获取温度）
    ├── weather-fetcher（agent 技能 —— 预加载的 API 指令）
    ↓
weather-svg-creator（技能 —— 内联创建 SVG）
```

---

## Frontmatter 对比

### Agent Frontmatter

```yaml
---
name: my-agent
description: Use this agent PROACTIVELY when...
tools: Read, Write, Edit, Bash
model: sonnet
maxTurns: 10
permissionMode: acceptEdits
memory: user
skills:
  - my-skill
---
```

### Command Frontmatter

```yaml
---
description: Do something useful
argument-hint: [issue-number]
allowed-tools: Read, Edit, Bash(gh *)
model: sonnet
---
```

### Skill Frontmatter

```yaml
---
name: my-skill
description: Do something when the user asks for...
argument-hint: [file-path]
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Grep, Glob
model: sonnet
context: fork
agent: general-purpose
---
```

---

## 关键区别

### 自动调用

| 机制 | Claude 能自动调用吗？ | 如何阻止 |
|-----------|------------------------|----------------|
| Agent | 是 —— 通过 `description`（用 "PROACTIVELY" 鼓励） | 移除或弱化描述 |
| Command | 否 —— 始终由用户通过 `/` 发起 | 不适用 |
| Skill | 是 —— 通过 `description` | 设置 `disable-model-invocation: true` |

### `/` 菜单中的可见性

| 机制 | 出现在 `/` 菜单中？ | 如何隐藏 |
|-----------|---------------------|-------------|
| Agent | 否 | 不适用 |
| Command | 是 —— 始终 | 无法隐藏 |
| Skill | 是 —— 默认 | 设置 `user-invocable: false` |

### 上下文隔离

| 机制 | 在自己的上下文中运行？ | 如何配置 |
|-----------|---------------------|-----------------|
| Agent | 始终是 | 内置行为 |
| Command | 从不 | 不适用 |
| Skill | 可选 | 设置 `context: fork` |

---

## 完整示例："当前时间是什么？"

本仓库为同一任务定义了三种机制 —— 显示 PKT 当前时间。当用户输入 **"What is the current time?"** 且未显式调用任何 `/` 命令时：

| 机制 | 会触发吗？ | 原因 |
|-----------|--------------|---------------|
| `time-command` | 否 | 命令**永远不会自动调用**。用户需要显式输入 `/time-command` 才能运行。命令没有自动发现途径 —— 严格由用户发起。 |
| `time-agent` | **可能** | agent 的 `description` 写着 *"Use this agent to display the current time in Pakistan Standard Time"*。Claude 将其与用户意图匹配并可能通过 Agent 工具生成它。然而，agent 在**独立上下文窗口**中运行，对于这个简单任务来说过于重了。 |
| `time-skill` | **最可能** | 技能的 `description` 写着 *"Display the current time in Pakistan Standard Time (PKT, UTC+5). Use when the user asks for the current time, Pakistan time, or PKT."* Claude 将其与用户意图匹配并通过 Skill 工具调用它。由于它**内联运行**，没有上下文开销，它是最高效的匹配。 |

### 解决顺序

当多个机制匹配同一意图时，Claude 偏好**最轻量级选项**：

```
1. Skill（内联，无上下文开销）     ← 首选
2. Agent（独立上下文，自主执行）    ← 无技能可用或任务复杂时使用
3. Command（从不 —— 需要显式 /）   ← 仅当用户输入 /time-command 时
```

### 如果技能设置了 `disable-model-invocation: true` 会怎样？

那么 Claude**无法**自动调用该技能。agent 成为唯一可自动调用的选项，因此 Claude 会生成 `time-agent` —— 但代价是为了一行 bash 命令开了一个独立的上下文窗口。

### 如果技能和 agent 的自动调用都被禁用了呢？

**没有东西会自动触发**。Claude 会回退到自己的通用知识，很可能直接运行 `TZ='Asia/Karachi' date` —— 不涉及任何扩展机制。用户需要显式输入 `/time-command` 或 `/time-skill` 才能使用。

![Claude 在用户问 "What is the current time?" 时自动调用 time-skill](assets/agent-command-skill-2.png)

---

## 来源

- [Claude Code Skills — Docs](https://code.claude.com/docs/en/skills)
- [Claude Code Sub-agents — Docs](https://code.claude.com/docs/en/sub-agents)
- [Claude Code Slash Commands — Docs](https://code.claude.com/docs/en/slash-commands)
- [Skills Best Practice](../best-practice/claude-skills.md)
- [Commands Best Practice](../best-practice/claude-commands.md)
- [Sub-agents Best Practice](../best-practice/claude-subagents.md)
