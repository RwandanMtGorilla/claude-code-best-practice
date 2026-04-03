# Claude Code 长程运行策略

本文梳理了仓库中讨论的所有关于让 Claude Code 长时间、持续运行的方法与最佳实践。

---

## 一、核心挑战：Context Window 污染与"Dumb Zone"

长程运行的根本障碍是 **context window 有限**。随着对话积累，上下文被填满，Claude 的响应质量会下降。

> "In a long coding session, earlier mistakes accumulate in context. The model sees its own errors and may perpetuate them. This is the most common cause of 'Claude got dumber' within a single session — it's not the model degrading, it's context contamination."
> — `reports/llm-day-to-day-degradation.md`

**Dumb Zone 概念**（出自 Dex / MLOps Community 演讲）：大约在 context 使用率 ~40% 时，结果开始退化。使用越少的 context，结果越好。

---

## 二、Context 管理策略

### 2.1 手动 /compact — 不要等自动压缩

```
在 ~50% context 使用率时手动执行 /compact
```

仓库多处强调这一点：
- `CLAUDE.md`: "Perform manual `/compact` at ~50% context usage"
- `README.md`: "avoid agent dumb zone, do manual /compact at max 50%"
- `presentation/index.html`: "Don't wait until it auto-compacts — you lose control over what gets preserved"

**来源**: `CLAUDE.md:97`, `README.md:203`

### 2.2 环境变量微调压缩行为

| 变量 | 作用 |
|------|------|
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 自动压缩阈值百分比（1-100），默认 ~95%，设低可提前触发 |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | 设定用于压缩计算的 token 容量上限 |
| `DISABLE_AUTO_COMPACT` | 禁用自动压缩（手动 `/compact` 仍可用） |

本仓库自身的配置即将 auto-compact 设为 80%：
```json
"CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "80"
```

**来源**: `best-practice/claude-settings.md:711,835,836`

### 2.3 /clear 重置上下文

当切换到完全不同的任务时，使用 `/clear` 重置上下文，而非在同一被污染的 context 中继续。

**来源**: `README.md:203`

### 2.4 任务拆分：每个子任务 < 50% context

```
Break subtasks small enough that each can be completed in under 50% context.
Commit after each subtask.
```

**来源**: `CLAUDE.md:100`, `presentation/index.html:711`

---

## 三、Subagent 策略：隔离 Context 保持主会话清洁

### 3.1 Subagent 天然隔离

Subagent 运行在 **独立的 context 中**，完成后返回摘要，其 context 被丢弃。这意味着：

- 主 agent 的 context 不会被子任务细节污染
- 同一个模型在不同 context 中可以发现彼此的 bug

> "Offload individual tasks to subagents to keep your main agent's context window clean and focused."
> — Boris, 10 Tips (Feb 2026)

**来源**: `tips/claude-boris-10-tips-01-feb-26.md:117`

### 3.2 Test Time Compute — 多 Context Window 策略

> "Using separate context windows makes the result even better — this is what makes subagents work, and why one agent can cause bugs and another (using the same exact model) can find them."
> — Boris, 2 Tips (Mar 2026)

**来源**: `tips/claude-boris-2-tips-10-mar-26.md:30`

---

## 四、长时间自主运行的具体方案

### 4.1 Ralph Wiggum Loop — 自主开发循环

仓库中收录的官方插件，用于长程自主任务迭代直到完成：

> "Autonomous development loop for long-running tasks — iterates until completion"

**来源**: `README.md:58`（CONCEPTS 表格）

### 4.2 /loop — 本地定时循环（最长 3 天）

`/loop` 技能可以在固定间隔重复执行命令：

```bash
/loop 5m /babysit        # 每 5 分钟自动处理代码审查和 rebase
/loop 30m /slack-feedback # 每 30 分钟自动提交 PR
/loop 1h /pr-pruner       # 每小时清理过期 PR
```

特点：
- 基于 cron 工具（`CronCreate`/`CronList`/`CronDelete`）
- 会话级作用域 — 退出 Claude 后停止
- 自动过期上限 3 天

**来源**: `tips/claude-boris-15-tips-30-mar-26.md:46-56`, `implementation/claude-scheduled-tasks-implementation.md`

### 4.3 /schedule — 云端定时任务（离线也可运行）

> "use /schedule for cloud-based recurring tasks that run even when your machine is off"

`/schedule` 创建云端远程 agent（trigger），即使本地机器关闭也能在 cron 计划上运行。

**来源**: `README.md:216`

### 4.4 Stop Hook — 验证长程任务结果

> "For very long-running tasks, either (a) prompt Claude to verify its work with a background agent when it's done, (b) use an agent Stop hook to do that more deterministically, or (c) use the ralph-wiggum plugin."
> — Boris, 13 Tips (Jan 2026)

Stop Hook 可以在 Claude 完成每个 turn 后自动触发验证逻辑。

**来源**: `tips/claude-boris-13-tips-03-jan-26.md:130-132`

---

## 五、并行运行多个 Claude 实例

### 5.1 Git Worktrees — 并行开发

Boris 同时运行 **数十个 Claude**，靠的就是 worktree：

```bash
claude -w  # 在新 worktree 中启动会话
```

每个 worktree 是独立的代码副本，互不干扰。

**来源**: `tips/claude-boris-15-tips-30-mar-26.md:138-146`, `tips/claude-boris-10-tips-01-feb-26.md:24`

### 5.2 /batch — 扇出大规模变更

```
/batch interviews you, then has Claude fan out the work to as many
worktree agents as it takes (dozens, hundreds, even thousands).
```

适用于大规模代码迁移等可并行化工作。

**来源**: `tips/claude-boris-15-tips-30-mar-26.md:150-155`

### 5.3 Agent Teams — 多 Agent 协作

通过 tmux 启动多个独立 Claude 会话，通过共享任务列表协调：

```bash
tmux new -s dev
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 claude
```

每个 teammate 拥有完整的 context window、CLAUDE.md、MCP servers 和 skills。

**来源**: `implementation/claude-agent-teams-implementation.md:20`

---

## 六、会话持久化与恢复

### 6.1 /rename + /resume — 命名并恢复会话

```bash
/rename "TODO - refactor task"  # 命名当前会话
claude --resume                  # 恢复最近会话
claude -r <session-id>           # 恢复指定会话
```

**来源**: `README.md:208`

### 6.2 CLAUDE_CODE_RESUME_INTERRUPTED_TURN

环境变量，设为 `1` 可在上次中断处自动恢复。

**来源**: `best-practice/claude-settings.md:754`

### 6.3 静态资产策略 — 免除压缩依赖

Dex 团队的做法：将所有重要信息写入静态文件（markdown 等），而非依赖 context：

> "We don't use the built-in compaction because everything that matters is going into static assets and so you can always resume from where you left off without having to worry about the quality of an autocompact or a manual compact."
> — Dex, MLOps Community

**来源**: `videos/claude-dex-mlops-community-24-mar-26.md:189`

---

## 七、远程/无人值守运行

### 7.1 Headless Mode (`-p`)

```bash
claude -p "run migration"  # 非交互模式执行
```

适合 CI/CD 和脚本调用。

**来源**: `best-practice/claude-cli-startup-flags.md:80`

### 7.2 Remote Control

从手机、平板或浏览器继续本地会话：

```
/remote-control  # 或 /rc
```

**来源**: `README.md:56`

---

## 八、策略总结

| 策略 | 适用场景 | 持续时间上限 |
|------|---------|-------------|
| 手动 `/compact` | 所有长会话 | 无（持续管理） |
| 任务拆分 < 50% context | 复杂多步骤任务 | 每子任务约一半 context |
| Subagent 隔离 | 保持主 context 清洁 | 每个 subagent 独立 |
| Ralph Wiggum Loop | 自主迭代开发 | 直到完成 |
| `/loop` | 本地定时监控/自动化 | 最长 3 天 |
| `/schedule` | 云端定时任务 | 无（云端持久） |
| Git Worktrees | 并行多任务 | 每个 worktree 独立 |
| `/batch` | 大规模并行变更 | 取决于任务数量 |
| Agent Teams | 多 agent 协作 | 共享任务列表驱动 |
| `/resume` | 会话恢复 | 跨会话 |
| Stop Hook 验证 | 确保自主任务质量 | 每 turn 触发 |
| Headless + Remote | 无人值守/远程操控 | 依运行环境 |

---

## 核心原则

1. **越少 context 用量 = 越好的结果** — 积极管理 context
2. **隔离比共享好** — 用 subagent/worktree 而非堆积在一个 context
3. **写入静态资产** — 重要信息持久化到文件，不依赖 context 记忆
4. **验证机制不可少** — 给 Claude 反馈循环（测试、hook、background agent）
5. **拆小任务** — 每个子任务在 50% context 内可完成
