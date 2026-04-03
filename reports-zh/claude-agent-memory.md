# Claude Code: Agent 记忆 Frontmatter

子 agent 的持久记忆 —— 让 agent 跨会话学习、记忆和构建知识。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## 概述

**Claude Code v2.1.33**（2026 年 2 月）引入了 `memory` frontmatter 字段，每个子 agent 拥有自己持久的基于标记的知识存储。在此之前，每次 agent 调用都从零开始。

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Write, Edit, Bash
model: sonnet
memory: user
---

You are a code reviewer. As you review code, update your agent memory with
patterns, conventions, and recurring issues you discover.
```

---

## 记忆作用域

| 作用域 | 存储位置 | 版本控制 | 共享 | 最适合 |
|-------|-----------------|-------------------|--------|----------|
| `user` | `~/.claude/agent-memory/<agent-name>/` | 否 | 否 | 跨项目知识（推荐默认值） |
| `project` | `.claude/agent-memory/<agent-name>/` | 是 | 是 | 团队应共享的项目特定知识 |
| `local` | `.claude/agent-memory-local/<agent-name>/` | 否（git-ignored） | 否 | 个人化的项目特定知识 |

这些作用域与设置层级（`~/.claude/settings.json` → `.claude/settings.json` → `.claude/settings.local.json`）对应。

---

## 工作原理

1. **启动时**：`MEMORY.md` 的前 200 行注入到 agent 的系统提示中
2. **工具访问**：`Read`、`Write`、`Edit` 自动启用，agent 可以管理记忆
3. **执行期间**：agent 自由读写其记忆目录
4. **管理**：如果 `MEMORY.md` 超过 200 行，agent 将细节移到主题特定文件中

```
~/.claude/agent-memory/code-reviewer/     # user 作用域示例
├── MEMORY.md                              # 主要文件（加载前 200 行）
├── react-patterns.md                      # 主题特定文件
└── security-checklist.md                  # 主题特定文件
```

---

## Agent 记忆与其他记忆系统对比

| 系统 | 谁写入 | 谁读取 | 作用域 |
|--------|-----------|-----------|-------|
| **CLAUDE.md** | 你（手动） | 主 Claude + 所有 agent | 项目 |
| **自动记忆** | 主 Claude（自动） | 仅主 Claude | 每项目每用户 |
| **`/memory` 命令** | 你（通过编辑器） | 仅主 Claude | 每项目每用户 |
| **Agent 记忆** | agent 自身 | 仅该特定 agent | 可配置（user/project/local） |

这些系统是**互补的** —— agent 读取 CLAUDE.md（项目上下文）和自己的记忆（agent 特定知识）。

---

## 实际示例

```yaml
---
name: api-developer
description: Implement API endpoints following team conventions
tools: Read, Write, Edit, Bash
model: sonnet
memory: project
skills:
  - api-conventions
  - error-handling-patterns
---

Implement API endpoints. Follow the conventions from your preloaded skills.
As you work, save architectural decisions and patterns to your memory.
```

这将**技能**（启动时的静态知识）与**记忆**（随时间构建的动态知识）结合起来。

---

## 提示

- **提示使用记忆** —— 包含显式指令：`"开始前，回顾你的记忆。完成后，更新记忆。"`
- **调用 agent 时请求记忆检查**：`"Review this PR, and check your memory for patterns you've seen before."`
- **选择合适的作用域** —— `user` 用于跨项目，`project` 用于团队共享，`local` 用于个人

---

## 来源

- [Create custom subagents — Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Manage Claude's memory — Claude Code Docs](https://code.claude.com/docs/en/memory)
- [Claude Code v2.1.33 Release Notes](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
