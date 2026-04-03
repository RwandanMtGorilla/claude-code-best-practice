# Deep Research Skill 优化调研报告：Skill 中穿插 Agent 的实践

> 基于 `claude-code-best-practice` 仓库的全方位调研
>
> 2026-04-03 — 回答用户的两个核心问题：
> 1. 本仓库是否有 deep-research 相关的实践案例？
> 2. 在 Skill 中穿插使用 Sub-Agent 的做法是否推荐？

---

## 一、结论摘要

**你的直觉是对的。** 本仓库不推荐"把所有信息塞进一个 Skill"的做法，而是推崇 **Command → Agent → Skill 的分层架构**，核心原则是：

| 原则 | 仓库实践 |
|------|---------|
| **技能做知识，Agent 做执行** | Skill 是静态知识注入，Agent 是自主多步执行 |
| **上下文隔离优于单一大上下文** | Agent 有独立上下文窗口，Skill 共享主上下文 |
| **不要使用 Skill 做控制流** | Dex 在 2026-03-24 演讲中明确：用代码做控制流，不用 Prompt |
| **指令预算有限（~150-200条）** | 大 Skill 必然丢失指令遵循 |

---

## 二、本仓库中的 Deep Research 实践案例

### 2.1 RPI（Research-Plan-Implement）工作流

仓库中的 `development-workflows/rpi/` 是最接近 deep-research 的实践。它采用 **三阶段 Command 架构**，每阶段调用不同的 Agent：

```
/rpi:research → 解析需求 → 产品分析 → 技术探索 → 可行性评估 → 生成报告
/rpi:plan     → 读研究报告 → 架构设计 → 任务拆解 → 生成4份文档
/rpi:implement → 按计划逐步实现
```

**关键设计：**
- 每阶段是独立的 Command，不依赖单一 Skill 堆叠
- 每个阶段调用 4-6 个专业 Agent（product-manager、senior-software-engineer、Explore 等）
- 每个 Agent 有独立上下文窗口，避免上下文膨胀
- 阶段结束后主动建议 `/compact` 清理上下文

![RPI 调研](../reports/claude-agent-command-skill.md)

### 2.2 调研 Command 中的 Agent 委托模式

`rpi/research.md` 中明确展示了 **多 Agent 分阶段委托** 模式：

| 阶段 | Agent | 类型 | 目的 |
|------|-------|------|------|
| Phase 1 | requirement-parser | 自定义 | 结构化需求提取 |
| Phase 2 | product-manager | 自定义 | 产品可行性分析 |
| Phase 2.5 | Explore | 内置 | 深入代码库探索（CRITICAL PHASE） |
| Phase 3 | senior-software-engineer | 自定义 | 技术可行性评估 |
| Phase 4 | technical-cto-advisor | 自定义 | 战略综合评估 |
| Phase 5 | documentation-analyst-writer | 内置 | 生成研究报告 |

**关键洞察：** 这里不是把所有研究逻辑塞一个 Skill，而是通过 Command 编排多个 Agent，每个 Agent 负责一个分析维度。

### 2.3 Dex Horthy 演讲中的 "Crispy" 方法论

仓库收录了 Dex Horthy 在 2026-03-24 的演讲全文（`videos/claude-dex-mlops-community-24-mar-26.md`），其中明确指出：

> "不要使用 Prompt 做控制流，用代码做控制流。"
>
> "LLMs 只能一致地遵循约 150-200 条指令，超过这个数量就会注意力分散。"
>
> "与其用一个 85 条指令的 Mega Prompt，不如拆分成多个 < 40 条指令的独立 Prompt。"

演讲将 RPI 拆分为 7 个更小的阶段：Questions → Research → Design → Structure → Outline → Plan → Implement，**核心思想是拆分而非堆叠**。

---

## 三、Skill 中穿插 Sub-Agent 的做法分析

### 3.1 仓库支持的 Skill + Agent 配合模式

仓库演示了以下三种 Skill 和 Agent 的配合模式：

#### 模式 A：Agent 预加载 Skill（Agent Skill）

```yaml
# .claude/agents/weather-agent.md
skills:
  - weather-fetcher    # Skill 作为静态知识注入 Agent
```

- Skill 内容在 Agent 启动时**完整注入**到 Agent 的上下文
- Agent 遵循 Skill 的指令执行操作
- Skill 不自己调用 Agent — 是 Agent 加载 Skill

#### 模式 B：Skill 使用 `context: fork` 在隔离子 Agent 中运行

```yaml
# .claude/skills/<name>/SKILL.md
context: fork
agent: general-purpose
```

- Skill 本身可以在隔离的 Agent 上下文中执行
- 这是 Skill 级别的 `context: fork`，不是 Skill 内部调用 Agent
- 等同于"这个 Skill 需要独立的上下文窗口"

#### 模式 C：Command 调用 Agent → Command 调用 Skill

```
/weather-orchestrator (Command)
    ├── Agent tool → weather-agent (独立上下文)
    └── Skill tool → weather-svg-creator (共享上下文)
```

这才是本仓库倡导的**标准编排模式**。

### 3.2 Skill 能否在内部调用 Agent？

**技术上可行**，但不是本仓库推荐的做法。原因：

| 维度 | Skill 调用 Agent 的问题 | 推荐替代方案 |
|------|----------------------|-------------|
| **架构清晰度** | Skill 变成隐式编排器，职责混乱 | Command 做编排，Skill 做知识 |
| **上下文管理** | Skill 与主会话共享上下文，没有隔离 | Agent 天然隔离上下文 |
| **可维护性** | 大 Skill 内容 + Agent 调用逻辑混杂 | 分离关注点 |
| **指令预算** | Skill 既要当知识库又要当调度员 | 各自专注单一职责 |

### 3.3 `context: fork` — 技能级别的 Agent 隔离

这是仓库中 **最接近"Skill 中穿插 Agent"的官方机制**：

```yaml
---
name: my-heavy-skill
context: fork
agent: general-purpose
---
```

- 设置 `context: fork` 后，Skill 在一个**隔离的 subagent 上下文**中执行
- 可以指定 `agent` 字段选择特定的子 Agent 类型
- 适用于技能内容较重、需要隔离上下文的场景
- **注意**：这不是"Skill 内部调用 Agent"，而是"Skill 本身在独立 Agent 中运行"

---

## 四、针对 Deep Research Skill 的优化建议

### 4.1 你当前做法的问题

```
❌ 把所有研究信息、步骤、数据收集逻辑都塞进一个 Skill
→ 指令超预算、上下文膨胀、指令遵循丢失
```

### 4.2 推荐的架构

```
/deep-research (Command — 入口)
    ↓
Step 1: research-planner (Agent — 生成研究问题和计划)
    ↓
Step 2: Explore agent (Agent — 深入代码/网络探索)
    ↓
Step 3: research-analyst (Agent — 分析发现、综合结论)
    ↓
Step 4: report-writer (Skill 或 Agent — 生成报告)
```

**或者，如果 Skill 必须有：**

```yaml
# Skill 只包含领域知识
# .claude/skills/deep-research-domain/SKILL.md
---
name: deep-research-domain
user-invocable: false   # 不暴露给用户，仅供 Agent 预加载
---
# 仅包含研究方法论、数据源列表、报告模板等静态知识
```

```yaml
# Agent 执行研究
# .claude/agents/deep-research-agent.md
---
name: deep-research-agent
description: 自主执行深度研究任务
skills:
  - deep-research-domain   # 预加载领域知识
maxTurns: 20
memory: project
---
# 遵循预加载 Skill 的方法论执行研究
```

### 4.3 关键设计原则

1. **Skill = 静态知识，Agent = 动态执行**
   - Skill 存放"怎么做"的方法论和约定
   - Agent 负责"做"的多步骤执行

2. **每个 Agent 一个职责，不要堆叠**
   - 参考 RPI 模式：research Agent、plan Agent、implement Agent 各司其职

3. **利用 `memory` 字段做跨会话知识积累**
   ```yaml
   memory: project   # 研究结果沉淀，下次研究可以复用
   ```

4. **利用 `maxTurns` 控制 Agent 开销**
   ```yaml
   maxTurns: 15   # 防止 Agent 无限循环
   ```

5. **Command 中用 `/compact` 提示用户清理上下文**
   - 参考 RPI 的 `Post-Completion Action` 模式

---

## 五、关于"像真人员工一样持续运行并汇报"

仓库中相关的实践：

### 5.1 Agent Teams 模式

`implementation/claude-agent-teams-implementation.md` 展示了多 Agent 并行协作模式：
- 每个 teammate 有**独立的完整上下文窗口**
- 通过**共享 Task List** 协调
- 需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 启用

### 5.2 定时任务 + 循环执行

- `/loop` 技能支持在固定间隔重复执行命令（最长 3 天）
- `CronCreate` 工具支持定时任务调度

### 5.3 "持续运行汇报"的可行方案

仓库没有直接的"持续守护进程"模式，但可组合以下机制模拟：

| 机制 | 用途 |
|------|------|
| `CronCreate` | 定时触发研究任务 |
| `Agent` (background) | 后台运行研究 Agent |
| 共享记忆文件 | Agent 之间通过文件传递进展 |
| `/loop` | 定期执行检查点 |
| Agent `memory` | 跨会话记住研究进度 |

---

## 六、本仓库关键文件索引

| 文件 | 相关内容 |
|------|---------|
| `best-practice/claude-skills.md` | Skill 最佳实践，含 `context: fork` 说明 |
| `best-practice/claude-subagents.md` | Sub-Agent 最佳实践，含 `skills:` 预加载 |
| `best-practice/claude-commands.md` | Command 最佳实践，含 64 个内置命令 |
| `reports/claude-agent-command-skill.md` | Agent vs Command vs Skill 全面对比 |
| `reports/claude-skills-for-larger-mono-repos.md` | 大仓库中的 Skill 发现机制 |
| `reports/claude-agent-memory.md` | Agent 持久化记忆，支持跨会话积累 |
| `implementation/claude-skills-implementation.md` | Skill 实现示例 |
| `implementation/claude-subagents-implementation.md` | Sub-Agent 实现示例 |
| `implementation/claude-commands-implementation.md` | Command 实现示例 |
| `implementation/claude-agent-teams-implementation.md` | Agent Teams 实现 |
| `orchestration-workflow/orchestration-workflow.md` | Command → Agent → Skill 流程图 |
| `development-workflows/rpi/.claude/commands/rpi/research.md` | 深度研究 Command 实现 |
| `videos/claude-dex-mlops-community-24-mar-26.md` | Dex 演讲全文：从 RPI 到 Crispy 方法论 |
