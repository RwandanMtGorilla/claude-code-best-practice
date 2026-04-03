# Claude Code: 使用量、速率限制和额外使用量

了解 Claude Code 中的使用限制如何工作，以及在触及限制时如何继续工作。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

## 概览

Claude Code 的订阅计划（Pro、Max 5x、Max 20x）有在滚动窗口重置的使用限制。三个内置斜杠命令可以帮助你监控和管理使用量：

| 命令 | 描述 | 适用于 |
|---------|-------------|--------------|
| `/usage` | 检查计划限制和速率限制状态 | Pro、Max 5x、Max 20x |
| `/extra-usage` | 配置触及限制时的按量溢出计费 | Pro、Max 5x、Max 20x |
| `/cost` | 显示当前会话的令牌使用和支出 | API 密钥用户 |

---

## `/usage` —— 检查你的限制

显示你当前计划的使用限制和速率限制状态。有助于在触及限制之前查看还剩多少容量。

---

## `/extra-usage` —— 超过限制时继续工作

`/extra-usage` 命令配置**按量溢出计费**，使 Claude Code 在触及计划速率限制时无缝继续工作，而非阻止你。

### 工作原理

1. 你触及了计划的速率限制（限制每 5 小时重置一次）
2. 如果启用了额外使用量且有可用余额，Claude Code 将继续不中断
3. 溢出令牌按**标准 API 费率**计费，与订阅费分开

### 设置方法

CLI 中的 `/extra-usage` 命令会引导你完成配置。你也可以在 claude.ai 的**设置 > 用量**网页上配置：

1. 启用额外使用量
2. 添加付款方式
3. 设置**每月支出上限**（或选择无限制）
4. 可选添加**预付费资金**，在余额低于阈值时自动充值

### 关键细节

| 详情 | 值 |
|--------|-------|
| 每日兑换限额 | $2,000/天 |
| 计费 | 与订阅分开，按标准 API 费率 |
| 限制重置窗口 | 每 5 小时 |

### 已知问题

截至 2026 年 2 月，`/extra-usage` CLI 命令[未记录文档](https://github.com/anthropics/claude-code/issues/12396)，可能会打开一个登录窗口但没有清晰的配置选项。目前通过 **claude.ai 网页界面**配置是更可靠的路径。

---

## `/cost` —— 会话支出（API 用户）

对于使用 API 密钥（而非订阅计划）进行身份验证的用户，`/cost` 显示：

- 当前会话的总花费
- API 持续时间和墙壁时间
- 令牌使用细目
- 代码更改次数

此命令不适用于 Pro/Max 订阅用户。

---

## 快速模式与额外使用量

快速模式（`/fast`）使用 Claude Opus 4.6 并以更快的速度输出。它与额外使用量有特殊的计费关系：

- 快速模式的使用**始终计入额外使用量**，从第一个令牌开始
- 即使你的订阅计划仍有剩余使用量也是如此
- 快速模式不消耗你计划的包含速率限制

这意味着你需要启用并充额外使用量才能使用 `/fast`。

---

## CLI 启动标志

两个与使用预算相关的启动标志（仅限 API 密钥用户，打印模式）：

| 标志 | 描述 |
|------|-------------|
| `--max-budget-usd <金额>` | API 调用在停止前的最大美元金额 |
| `--max-turns <数字>` | 限制 agent 回合数 |

完整列表参见 [CLI 启动标志参考](claude-cli-startup-flags.md)。

---

## 来源

- [Extra usage for paid Claude plans — Claude Help Center](https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans)
- [Using Claude Code with your Pro or Max plan — Claude Help Center](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- [/extra-usage slash command is undocumented — GitHub Issue #12396](https://github.com/anthropics/claude-code/issues/12396)
- [Claude Code CLI Reference](https://code.claude.com/docs/en/cli-reference)
