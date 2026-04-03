# Claude Agent SDK vs Claude CLI：系统提示和输出一致性

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

![SDK vs CLI 系统提示图](assets/sdk-vs-cli-diagram.svg)

---

## 执行摘要

当通过 **Claude Agent SDK** 和 **Claude CLI（Claude Code）** 发送同一消息（例如 "What is the capital of Norway?"）时，伴随这些消息的系统提示根本不同。CLI 使用**模块化系统提示架构**（基础约 269 令牌，附加内容根据功能条件加载），而 SDK 默认使用最简提示。**即使配置相同，也无法保证输出完全一致**，因为没有种子参数且 Claude 架构本身存在固有的非确定性。

---

## 1. 系统提示对比

### Claude CLI（Claude Code）

Claude CLI 使用**模块化系统提示架构**，基础提示约 269 令牌，附加内容条件加载：

| 组件 | 描述 | 加载方式 |
|-----------|-------------|---------|
| **基础系统提示** | 核心指令和行为 | 始终加载（约 269 令牌） |
| **工具指令** | 18+ 内置工具（Write, Read, Edit, Bash, TodoWrite 等） | 始终加载 |
| **编码指南** | 代码风格、格式化规则、安全实践 | 始终加载 |
| **安全规则** | 拒绝规则、注入防御、伤害防御 | 始终加载 |
| **响应风格** | 语气、冗长程度、解释深度、Emoji 使用 | 始终加载 |
| **环境上下文** | 工作目录、git 状态、平台信息 | 始终加载 |
| **项目上下文** | CLAUDE.md 内容、设置、hooks 配置 | 条件加载 |
| **子 agent 提示** | 计划模式、Explore agent、Task agent | 条件加载 |
| **安全审查** | 扩展安全指令（约 2,610 令牌） | 条件加载 |

**关键特征：**
- **模块化架构**，110+ 个系统提示字符串条件加载
- 基础提示较为适中（约 269 令牌），总量根据激活的功能而变化
- 包含广泛的安全和注入防御层
- 自动加载工作目录中的 CLAUDE.md 文件
- 交互模式下会话持久上下文

### Claude Agent SDK

Agent SDK 默认使用**极简系统提示**，包含：

| 组件 | 描述 | 令牌影响 |
|-----------|-------------|--------------|
| **必要工具指令** | 仅显式提供的工具 | 最小 |
| **基本安全** | 最小安全指令 | 最小 |

**关键特征：**
- 默认没有编码指南或风格偏好
- 没有项目上下文，除非显式配置
- 没有广泛的工具描述
- 需要显式配置才能匹配 CLI 行为

---

## 2. 每个接口发送的内容

### 示例："What is the capital of Norway?"

#### 通过 Claude CLI

```
System Prompt: [模块化，约 269+ 基础令牌]
├── 基础系统提示（约 269 令牌）
├── 工具指令（Write, Read, Edit, Bash, Grep, Glob 等）
├── Git 安全协议
├── 代码引用指南
├── 专业客观指令
├── 安全和注入防御规则
├── 环境上下文（操作系统、目录、日期）
├── CLAUDE.md 内容（如果存在）[条件]
├── MCP 工具描述（如果配置）[条件]
├── 计划/Explore 模式提示 [条件]
└── 会话/对话上下文

User Message: "What is the capital of Norway?"
```

#### 通过 Claude Agent SDK（默认）

```
System Prompt: [极简]
├── 必要工具指令（如果提供了工具）
└── 基本操作上下文

User Message: "What is the capital of Norway?"
```

#### 通过 Agent SDK（使用 `claude_code` preset）

```typescript
const response = await query({
  prompt: "What is the capital of Norway?",
  options: {
    systemPrompt: {
      type: "preset",
      preset: "claude_code"
    }
  }
});
```

```
System Prompt: [模块化，匹配 CLI]
├── 完整的 Claude Code 系统提示
├── 工具指令
├── 编码指南
└── 安全规则

// 注意：仍然不包含 CLAUDE.md，除非设置了 settingSources
```

---

## 3. 自定义方法

### Claude CLI 自定义

| 方法 | 命令 | 效果 |
|--------|---------|--------|
| **追加到提示** | `claude -p "..." --append-system-prompt "..."` | 在保留默认值的同时添加指令 |
| **替换提示** | `claude -p "..." --system-prompt "..."` | 完全替换系统提示 |
| **项目上下文** | CLAUDE.md 文件 | 自动加载，持久 |
| **输出风格** | `/output-style [name]` | 应用预定义响应风格 |

### Agent SDK 自定义

| 方法 | 配置 | 效果 |
|--------|---------------|--------|
| **自定义提示** | `systemPrompt: "..."` | 完全替换默认值（丢失工具） |
| **预设 + 追加** | `systemPrompt: { type: "preset", preset: "claude_code", append: "..." }` | 保留 CLI 功能 + 自定义指令 |
| **CLAUDE.md 加载** | `settingSources: ["project"]` | 加载项目级指令 |
| **输出风格** | `settingSources: ["user"]` 或 `settingSources: ["project"]` | 加载保存的输出风格 |

### 配置对比表

| 功能 | CLI 默认 | SDK 默认 | SDK 带预设 |
|---------|-------------|-------------|-----------------|
| 工具指令 | ✅ 完整 | ❌ 极简 | ✅ 完整 |
| 编码指南 | ✅ 是 | ❌ 否 | ✅ 是 |
| 安全规则 | ✅ 是 | ❌ 基本 | ✅ 是 |
| CLAUDE.md 自动加载 | ✅ 是 | ❌ 否 | ❌ 否* |
| 项目上下文 | ✅ 自动 | ❌ 否 | ❌ 否* |

*需要显式 `settingSources: ["project"]` 配置

---

## 4. 输出一致性保证

### 关键发现：不保证确定性

**Claude 消息 API 不提供用于可重复性的种子参数。** 这是根本的架构限制。

### 防止相同输出的因素

| 因素 | 描述 | 可控？ |
|--------|-------------|---------------|
| **不同的系统提示** | CLI 和 SDK 有不同的默认值 | ✅ 是（通过配置） |
| **浮点运算** | 并行硬件特性 | ❌ 否 |
| **MoE 路由** | Mixture-of-Experts 架构变化 | ❌ 否 |
| **批处理/调度** | 云基础设施差异 | ❌ 否 |
| **数值精度** | 推理引擎变化 | ❌ 否 |
| **模型快照** | 版本更新/变化 | ❌ 否 |

### 温度和采样

即使设置 `temperature=0.0`（贪婪解码）：
- 完全确定性**不保证**
- 基础设施因素仍可能导致微小差异
- 已知 bug：[Claude CLI 对相同输入产生非确定性输出](https://github.com/anthropics/claude-code/issues/3370)

---

## 5. 实现最大程度的一致性

要使 SDK 和 CLI 之间的输出**尽可能**一致：

### Agent SDK 配置

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

// 方案 1：使用 claude_code 预设
const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1024,
  // 尽可能匹配 CLI 系统提示
  system: "Your exact system prompt matching CLI",
  messages: [
    { role: "user", content: "What is the capital of Norway?" }
  ],
  // 用贪婪解码实现最大程度的一致性
  temperature: 0
});

// 方案 2：使用 Agent SDK query 函数
import { query } from "@anthropic-ai/agent-sdk";

for await (const message of query({
  prompt: "What is the capital of Norway?",
  options: {
    systemPrompt: {
      type: "preset",
      preset: "claude_code"
    },
    temperature: 0,
    model: "claude-sonnet-4-20250514",
    // 像 CLI 一样加载项目上下文
    settingSources: ["project"]
  }
})) {
  // 处理响应
}
```

### CLI 配置

```bash
# 尽可能匹配 SDK 配置
claude -p "What is the capital of Norway?" \
  --model claude-sonnet-4-20250514 \
  --temperature 0
```

### 仍然不保证

即使配置完美匹配：
- 输出可能在运行之间不同
- 输出可能在 SDK 和 CLI 之间不同
- 不存在用于强制可重复性的种子参数

---

## 6. 实际影响

### 何时使用每个接口

| 用例 | 推荐接口 | 原因 |
|----------|----------------------|--------|
| 交互式开发 | Claude CLI | 完整工具套件，项目上下文 |
| 编程集成 | Agent SDK | 细粒度控制，嵌入 |
| 一致的 API 响应 | Agent SDK + 自定义提示 | 更好的系统提示控制 |
| 批处理 | Agent SDK | 更适合自动化管线 |
| 一次性任务 | Claude CLI | 设置更快，即时上下文 |

### 设计建议

1. **不要依赖比特级可重复性**
   - 构建能容忍微小输出变化的应用
   - 使用结构化输出和验证

2. **对于需要一致性的生产管线：**
   - 尽可能缓存结果
   - 使用带 JSON schema 验证的结构化输出
   - 结合确定性逻辑和验证
   - 考虑多次生成并用共识

3. **在 SDK 中匹配 CLI 行为：**
   ```typescript
   systemPrompt: {
     type: "preset",
     preset: "claude_code",
     append: "Your additional instructions"
   },
   settingSources: ["project", "user"]
   ```

---

## 7. 系统提示令牌影响

| 配置 | 架构 | 备注 |
|---------------|-------------|-------|
| SDK（极简） | 最小默认值 | 仅必要工具指令 |
| SDK（claude_code 预设） | 模块化（约 269+ 基础） | 匹配 CLI，根据功能变化 |
| CLI（默认） | 模块化（约 269+ 基础） | 附加内容条件加载 |
| CLI（带 MCP 工具） | 模块化 + MCP | MCP 工具描述增加大量令牌 |

**注意：** Claude Code 使用模块化架构，有 110+ 个系统提示字符串。基础提示约 269 令牌，单个组件从 18 到 2,610 令牌不等，取决于激活的功能。

**影响：** SDK 的极简默认值让你的实际任务有更多上下文，但代价是损失了 Claude Code 的完整能力。

---

## 8. 汇总表

| 方面 | Claude CLI | Agent SDK（默认） | Agent SDK（预设） |
|--------|------------|--------------------|--------------------|
| **系统提示** | 模块化（约 269+ 基础） | 极简 | 模块化（匹配 CLI） |
| **包含的工具** | 18+ 内置 | 仅当提供时 | 18+ 内置 |
| **CLAUDE.md 自动加载** | 是 | 否 | 否（需要配置） |
| **编码指南** | 是 | 否 | 是 |
| **安全规则** | 完整 | 基本 | 完整 |
| **温度控制** | 是 | 是 | 是 |
| **确定性保证** | 否 | 否 | 否 |
| **相同输出？** | 不适用 | 否（vs CLI） | 更接近，但不完全 |

---

## 9. 结论

**SDK 和 CLI 之间的系统提示有何不同？**

CLI 使用**模块化系统提示架构**，基础提示约 269 令牌，加上 110+ 个条件加载的组件（工具指令、编码指南、安全规则、项目上下文）。SDK 使用**极简默认值**，仅包含基本工具指令，但可以配置为使用 `claude_code` 预设来匹配 CLI 行为。

**是否保证相同的输出？**

**不。** 即使系统提示匹配、输入相同且 `temperature=0`，也无法保证输出完全一致，原因如下：
- Claude API 中不存在种子参数
- 浮点运算的变化
- 基础设施级别的非确定性
- 模型架构（Mixture-of-Experts）路由变化

**建议：** 设计系统时应该容忍输出变化，而不是依赖确定性行为。对于一致性要求高的应用，使用结构化输出、缓存和验证层。

---

## 来源

- [Modifying System Prompts - Agent SDK](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/sdk#modifying-system-prompts)
- [Claude Code CLI Reference](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/cli)
- [Claude Code Headless Mode](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/headless)
- [Claude Code Best Practices - Anthropic Engineering](https://www.anthropic.com/engineering/claude-code-best-practices)
- [Claude Messages API Reference](https://docs.anthropic.com/en/api/messages)
- [GitHub Issue #3370: Non-deterministic output](https://github.com/anthropics/claude-code/issues/3370)
- [Claude Code System Prompts Repository](https://github.com/Piebald-AI/claude-code-system-prompts) - 模块化提示架构分析
- [Why Deterministic Output from LLMs is Nearly Impossible](https://unstract.com/blog/understanding-why-deterministic-output-from-llms-is-nearly-impossible/)

---

*本报告由 Claude Code 使用 Opus 4.5 模型生成于 2026 年 2 月 3 日。*
