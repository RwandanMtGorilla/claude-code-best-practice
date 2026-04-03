# Claude 高级工具使用模式

API 级别的功能（现已正式发布）可减少令牌消耗、延迟，并提高工具准确性。随 Opus/Sonnet 4.6 发布。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## 目录

1. [概览](#概览)
2. [程序化工具调用 (PTC)](#程序化工具调用-ptc)
3. [Web 搜索/获取的动态过滤](#web-搜索获取的动态过滤)
4. [工具搜索工具](#工具搜索工具)
5. [工具使用示例](#工具使用示例)
6. [与 Claude Code 的关联](#与-claude-code-的关联)

---

## 概览

| 功能 | 解决的问题 | 令牌节省 | 可用性 |
|---------|---------------|---------------|--------------|
| 程序化工具调用 | 多步骤 agent 循环在多轮往返中消耗令牌 | 减少约 37% | API、Foundry（正式发布） |
| 动态过滤 | Web 搜索/获取结果将无关内容撑大上下文 | 减少约 24% 输入令牌 | API、Foundry（正式发布） |
| 工具搜索工具 | 过多工具定义撑大上下文 | 减少约 85% | API、Foundry（正式发布） |
| 工具使用示例 | 仅靠 schema 无法表达使用模式 | 72% → 90% 准确率 | API、Foundry（正式发布） |

所有功能已于 2026 年 2 月 18 日**正式发布**。

**策略分层** —— 从最大瓶颈开始：
- 工具定义导致的上下文膨胀 → 工具搜索工具
- 大型中间结果 → 程序化工具调用
- Web 搜索噪声 → 动态过滤
- 参数错误 → 工具使用示例

---

## 程序化工具调用 (PTC)

<img src="assets/programmatic-tool-calling-diagram.svg" alt="PTC 图 — 传统 vs 程序化工具调用" width="100%" />

### 范式转变

**之前（传统工具调用）：**
```
用户提示 → Claude → 工具调用 1 → 响应 1 → Claude → 工具调用 2 → 响应 2 → Claude → 工具调用 3 → 响应 3 → Claude → 最终答案
```
每次工具调用都需要一次完整的模型往返。3 个工具 = 3 次推理传递。

**之后（程序化工具调用）：**
```
用户提示 → Claude → 编写 Python 脚本 → 脚本内部调用工具 1、工具 2、工具 3 → stdout → Claude → 最终答案
```
Claude 编写代码来协调所有工具。只有最终的 `stdout` 进入上下文窗口。3 个工具 = 1 次推理传递。

### 工作原理

1. 你定义工具时使用 `allowed_callers: ["code_execution_20250825"]`
2. Claude 编写的 Python 在沙箱中将这些工具作为异步函数调用
3. 当调用工具函数时，沙箱暂停，API 返回一个 `tool_use` 块
4. 你提供工具结果 —— 它进入**运行中的代码**，而非 Claude 的上下文
5. 代码恢复执行，处理结果，根据需要调用更多工具
6. 只有最终执行的 `stdout` 到达 Claude

### 关键配置

```json
{
  "tools": [
    {
      "type": "code_execution_20250825",
      "name": "code_execution"
    },
    {
      "name": "query_database",
      "description": "执行 SQL 查询。返回行 JSON 对象，包含字段：id (str), name (str), revenue (float)。",
      "input_schema": {
        "type": "object",
        "properties": {
          "sql": { "type": "string", "description": "要执行的 SQL 查询" }
        },
        "required": ["sql"]
      },
      "allowed_callers": ["code_execution_20250825"]
    }
  ]
}
```

### `allowed_callers` 字段

| 值 | 行为 |
|-------|----------|
| `["direct"]` | 仅传统工具调用（省略时的默认值） |
| `["code_execution_20250825"]` | 仅可从 Python 沙箱调用 |
| `["direct", "code_execution_20250825"]` | 两种模式均可用 |

**建议：** 每个工具选择一种模式，而非两者兼用。这给 Claude 提供更清晰的指引。

### 响应中的 `caller` 字段

每个工具使用块都包含 `caller` 字段，方便你知道调用方式：

```json
// 直接（传统方式）
{ "caller": { "type": "direct" } }

// 程序化（来自代码执行）
{ "caller": { "type": "code_execution_20250825", "tool_id": "srvtoolu_abc123" } }
```

### 高级模式

**批处理** —— 在一次推理传递中处理 N 个项目：
```python
regions = ["West", "East", "Central", "North", "South"]
results = {}
for region in regions:
    data = await query_database(f"SELECT SUM(revenue) FROM sales WHERE region='{region}'")
    results[region] = data[0]["revenue"]

top = max(results.items(), key=lambda x: x[1])
print(f"Top region: {top[0]} with ${top[1]:,}")
```

**提前终止** —— 一旦满足成功标准就停止：
```python
endpoints = ["us-east", "eu-west", "apac"]
for endpoint in endpoints:
    status = await check_health(endpoint)
    if status == "healthy":
        print(f"Found healthy endpoint: {endpoint}")
        break
```

**条件工具选择：**
```python
file_info = await get_file_info(path)
if file_info["size"] < 10000:
    content = await read_full_file(path)
else:
    content = await read_file_summary(path)
print(content)
```

**数据过滤** —— 减少 Claude 看到的内容：
```python
logs = await fetch_logs(server_id)
errors = [log for log in logs if "ERROR" in log]
print(f"Found {len(errors)} errors")
for error in errors[-10:]:
    print(error)
```

### 模型兼容性

| 模型 | 支持 |
|-------|-----------|
| Claude Opus 4.6 | 是 |
| Claude Sonnet 4.6 | 是 |
| Claude Sonnet 4.5 | 是 |
| Claude Opus 4.5 | 是 |

### 约束

| 约束 | 详情 |
|-----------|--------|
| **不支持 Bedrock/Vertex** | 仅限 API 和 Foundry |
| **不支持 MCP 工具** | MCP 连接器工具无法程序化调用 |
| **不支持 Web 搜索/获取** | Web 工具不支持 PTC |
| **不支持结构化输出** | `strict: true` 工具不兼容 |
| **不支持强制工具选择** | `tool_choice` 无法强制 PTC |
| **容器生命周期** | 约 4.5 分钟后过期 |
| **ZDR** | 不受零数据保留保护 |
| **工具结果为字符串** | 验证外部结果以防范代码注入风险 |

### 何时使用 PTC

| 适合的使用场景 | 不太理想 |
|----------------|------------|
| 处理需要聚合的大型数据集 | 简单响应的单个工具调用 |
| 3 个以上依赖工具的顺序调用 | 需要即时用户反馈的工具 |
| 在 Claude 看到之前过滤/转换结果 | 非常快的操作（开销 > 收益） |
| 跨多个项目的并行操作 | |
| 基于中间结果的条件逻辑 | |

### 令牌效率

- 程序化工具调用的结果**不会添加到 Claude 的上下文**中 —— 只有最终 `stdout` 会
- 中间处理发生在代码中，而非模型令牌中
- 10 次程序化调用 ≈ 10 次直接调用令牌量的 1/10

---

## Web 搜索/获取的动态过滤

### 问题

Web 搜索和获取工具将完整的 HTML 页面转储到 Claude 的上下文窗口中。大部分内容是无关的 —— 导航、广告、模板文字。Claude 随后对所有内容进行推理，浪费令牌并降低准确性。

### 解决方案

Claude 现在**编写并执行 Python 代码来过滤 Web 结果**，然后才进入上下文窗口。Claude 不再推理原始 HTML，而是在沙箱中过滤、解析和提取仅相关内容。

### 工作原理

**之前：**
```
查询 → 搜索结果 → 获取 N 个页面的完整 HTML → 所有内容进入上下文 → Claude 对所有内容进行推理
```

**之后：**
```
查询 → 搜索结果 → Claude 编写过滤代码 → 代码仅提取相关内容 → 过滤后的结果进入上下文
```

### API 配置

使用更新后的工具类型版本，加一个 beta 头：

```json
{
  "model": "claude-opus-4-6",
  "max_tokens": 4096,
  "tools": [
    {
      "type": "web_search_20260209",
      "name": "web_search"
    },
    {
      "type": "web_fetch_20260209",
      "name": "web_fetch"
    }
  ]
}
```

**需要请求头：** `anthropic-beta: code-execution-web-tools-2026-02-09`

**默认启用** —— 当使用新工具类型版本与 Sonnet 4.6 和 Opus 4.6 时。

### 基准测试结果

**BrowseComp**（在网站上查找特定信息）：

| 模型 | 无过滤 | 有过滤 | 改进 |
|-------|-------------------|----------------|-------------|
| Sonnet 4.6 | 33.3% | **46.6%** | +13.3 pp |
| Opus 4.6 | 45.3% | **61.6%** | +16.3 pp |

**DeepsearchQA**（多步骤研究，F1 分数）：

| 模型 | 无过滤 | 有过滤 | 改进 |
|-------|-------------------|----------------|-------------|
| Sonnet 4.6 | 52.6% | **59.4%** | +6.8 pp |
| Opus 4.6 | 69.8% | **77.3%** | +7.5 pp |

**令牌效率：** 平均减少 24% 输入令牌。Sonnet 4.6 成本降低；Opus 4.6 可能因过滤代码更复杂而略有增加。

### 使用场景

- 筛选技术文档
- 跨多个来源验证引用
- 交叉引用搜索结果
- 多步骤研究查询
- 在大型页面中找特定数据点

---

## 工具搜索工具

### 问题

一开始加载所有工具定义会浪费上下文。如果你有 50 个 MCP 工具，每个约 1.5K 令牌，那就是 75K 令牌 —— 用户甚至还没提问。

### 解决方案

将不常用工具标记为 `defer_loading: true`。它们初始不包含在上下文中。Claude 通过工具搜索工具按需发现它们。

### 配置

```json
{
  "tools": [
    {
      "type": "mcp_toolset",
      "mcp_server_name": "google-drive",
      "default_config": { "defer_loading": true },
      "configs": {
        "search_files": { "defer_loading": false }
      }
    }
  ]
}
```

### 最佳实践

- 保持 3-5 个最常用工具始终加载，其余延迟
- 编写清晰、描述性的工具名称和描述（搜索依赖于此）
- 系统提示中说明可用能力

### 何时使用

- 工具定义消耗超过 10K 令牌
- 10 个以上工具可用
- 多个 MCP 服务器
- 选项过多导致工具选择准确性问题

### 令牌节省

工具定义令牌减少约 85%（在 Anthropic 基准测试中 77K → 8.7K）。

### Claude Code 等价物

Claude Code 内置 **MCP 工具搜索自动模式**（自 v2.1.7 起默认启用）。当 MCP 工具描述超过上下文的 10% 时，它们将被延迟并通过 `MCPSearch` 发现。可通过 `ENABLE_TOOL_SEARCH=auto:N` 配置阈值，其中 N 是上下文百分比（0-100）。

---

## 工具使用示例

### 问题

JSON schema 定义结构但无法表达：
- 何时包含可选参数
- 哪些参数组合有意义
- 格式约定（日期格式、ID 模式）
- 嵌套结构使用

### 解决方案

在工具定义中添加 `input_examples` —— 超越 schema 的具体使用模式。

### 配置

```json
{
  "name": "create_ticket",
  "description": "创建支持票据",
  "input_schema": {
    "type": "object",
    "properties": {
      "title": { "type": "string" },
      "priority": { "type": "string", "enum": ["low", "medium", "high", "critical"] },
      "assignee": { "type": "string" },
      "labels": { "type": "array", "items": { "type": "string" } }
    },
    "required": ["title"]
  },
  "input_examples": [
    {
      "title": "登录页面返回 500 错误",
      "priority": "critical",
      "assignee": "oncall-team",
      "labels": ["bug", "auth", "production"]
    },
    {
      "title": "添加深色模式支持",
      "priority": "low",
      "labels": ["feature-request", "ui"]
    },
    {
      "title": "更新 v2 端点的 API 文档"
    }
  ]
}
```

### 最佳实践

- 使用**真实数据**，而非 "example_value" 这类占位符字符串
- 展示**多样性**：最小、部分和完整规范
- 保持简洁：**每个工具 1-5 个示例**
- 聚焦于解决歧义 —— 以行为明确性为目标，而非 schema 完整性
- 展示参数关联性（例如 `priority: "critical"` 通常有 `assignee`）

### 结果

Anthropic 基准测试中，复杂参数处理的准确率从 72% 提升至 90%。

---

## 与 Claude Code 的关联

### 直接适用于 Claude Code 用户的功能

| 功能 | Claude Code 状态 | 操作 |
|---------|-------------------|--------|
| 工具搜索 | v2.1.7 起内置为 MCPSearch 自动模式 | 拥有大量 MCP 工具时调整 `ENABLE_TOOL_SEARCH=auto:N` |
| 动态过滤 | CLI 中不可用（API 级 Web 工具） | 与做 Web 研究的 Agent SDK 用户相关 |
| PTC | CLI 中不可用 | 与构建自定义 agent 的 Agent SDK 用户相关 |
| 工具使用示例 | CLI 中不可配置 | 与自定义 MCP 服务器作者相关 |

### 面向 Agent SDK 开发者

如果你正在用 `@anthropic-ai/claude-agent-sdk` 构建 agent，PTC 是立即可行的：

1. 在工具数组中添加 `code_execution_20250825`
2. 在受益于批量/过滤的工具上设置 `allowed_callers`
3. 实现工具结果循环（暂停 → 提供结果 → 恢复）
4. 从工具返回结构化数据（JSON）以便更容易进行程序化解析

### 面向 MCP 服务器作者

如果你正在构建自定义 MCP 服务器，工具使用示例可以改善 Claude 使用你的工具的方式：
- 在工具 schema 中添加 `input_examples`
- 在描述中清晰文档化返回格式（PTC 需要要解析它们）

---

## 来源

- [Anthropic Engineering: 高级工具使用](https://www.anthropic.com/engineering/advanced-tool-use)
- [程序化工具调用的文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling)
- [代码执行工具文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)
- [Improved Web Search with Dynamic Filtering](https://claude.com/blog/improved-web-search-with-dynamic-filtering)
