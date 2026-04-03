# 全面浏览器自动化 MCP 对比报告

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## 执行摘要

在广泛研究的基础上，我分析了你截图中的两个工具，加上第三个主要竞争者。以下是全面的分解，以帮助你选择最佳选项用于自动化测试工作。

---

## 1. 三个竞争者

### **A. Chrome DevTools MCP**（截图 #1）
- **来源：** 官方 Google Chrome 团队
- **发布时间：** 2025 年 9 月公开预览
- **架构：** 建立在 Chrome DevTools Protocol (CDP) + Puppeteer
- **令牌使用：** ~19.0k 令牌（上下文的 9.5%）
- **工具数量：** 6 个类别共 26 个专业工具

### **B. Claude in Chrome**（截图 #2）
- **来源：** 官方 Anthropic 扩展
- **发布时间：** 公测，逐步面向所有付费计划（Pro、Max、Team、Enterprise）推出
- **架构：** 浏览器扩展，具有计算机使用能力
- **令牌使用：** ~15.4k 令牌（上下文的 7.7%）
- **工具数量：** 16 个工具，包括计算机使用能力

### **C. Playwright MCP**（强力竞争者）
- **来源：** Microsoft（官方 + 社区实现）
- **架构：** 基于无障碍树的自动化
- **令牌使用：** ~13.7k 令牌（上下文的 6.8%）
- **工具数量：** 21 个工具

---

## 2. 详细功能对比

| 功能 | Chrome DevTools MCP | Claude in Chrome | Playwright MCP |
|---------|---------------------|------------------|----------------|
| **主要用途** | 调试与性能 | 通用浏览器自动化 | UI 测试与端到端 |
| **浏览器支持** | 仅 Chrome | 仅 Chrome | Chromium、Firefox、WebKit |
| **令牌效率** | 19.0k（9.5%） | 15.4k（7.7%） | 13.7k（6.8%） |
| **元素选择** | CSS/XPath 选择器 | 视觉 + DOM | 无障碍树（语义） |
| **性能追踪** | ✅ 优秀 | ❌ 否 | ⚠️ 有限 |
| **网络检查** | ✅ 深入分析 | ⚠️ 基础 | ⚠️ 基础 |
| **控制台日志** | ✅ 完整访问 | ✅ 完整访问 | ⚠️ 有限 |
| **跨浏览器** | ❌ 否 | ❌ 否 | ✅ 是 |
| **CI/CD 集成** | ✅ 优秀 | ❌ 不佳（需要登录） | ✅ 优秀 |
| **无头模式** | ✅ 是 | ❌ 否 | ✅ 是 |
| **身份验证** | 需要配置 | 使用你的会话 | 需要配置 |
| **定时任务** | ❌ 否 | ✅ 是 | ❌ 否 |
| **费用** | 免费 | 需要付费计划 | 免费 |
| **本地设置** | 需要 Node.js | 浏览器扩展 | 需要 Node.js |

---

## 3. 工具细目

### Chrome DevTools MCP（26 个工具）

```
输入自动化（8）：    click, drag, fill, fill_form, handle_dialog,
                   hover, press_key, upload_file

导航（6）：        close_page, list_pages, navigate_page,
                  new_page, select_page, wait_for

模拟（2）：        emulate, resize_page

性能（3）：        performance_analyze_insight,
                  performance_start_trace, performance_stop_trace

网络（2）：        get_network_request, list_network_requests

调试（5）：        evaluate_script, get_console_message,
                  list_console_messages, take_screenshot,
                  take_snapshot
```

### Claude in Chrome（16 个工具）

```
浏览器控制：       navigate, read_page, find, computer
                  (click, type, scroll)

表单交互：         form_input, javascript_tool

媒体：             upload_image, get_page_text, gif_creator

标签管理：         tabs_context_mcp, tabs_create_mcp

开发：             read_console_messages, read_network_requests

实用工具：         shortcuts_list, shortcuts_execute,
                  resize_window, update_plan
```

### Playwright MCP（21 个工具）

```
导航：             navigate, goBack, goForward, reload

交互：             click, fill, select, hover, press,
                  drag, uploadFile

元素查询：          getElement, getElements, waitForSelector

断言：             assertVisible, assertText, assertTitle

页面状态：          screenshot, getAccessibilityTree,
                  evaluateScript

浏览器管理：        newPage, closePage
```

---

## 4. 自动化测试用例分析

### **Chrome DevTools MCP 最适合：**

✅ **性能测试**
- 录制性能追踪与 Core Web Vitals
- 识别渲染瓶颈和布局偏移
- 内存泄漏检测和 CPU 分析

✅ **深入调试**
- 网络请求检查（头部、负载、时序）
- 控制台错误分析和堆栈追踪
- 实时 DOM 检查

✅ **CI/CD 管线**
- 无头执行支持
- 稳定的基于脚本的自动化
- 无认证状态依赖

**理想工作流：** "找一下为什么这个页面慢" 或 "调试这个 API 调用"

---

### **Claude in Chrome 最适合：**

✅ **手动测试协助**
- 在已登录账户中测试
- 探索性测试，具有视觉上下文
- 录制的可重放工作流

✅ **快速验证**
- 设计验证（比较 Figma 到输出）
- 新功能抽查
- 开发过程中读取控制台错误

✅ **定时浏览器任务**
- 计划自动化检查
- 多标签工作流管理
- 从你录制的方法中学习

**理想工作流：** "检查我的改看起来对吗" 或 "用我的登录测试这个表单"

---

### **Playwright MCP 最适合：**

✅ **端到端测试自动化**
- 跨浏览器测试（Chrome、Firefox、Safari）
- 生成可重用的测试脚本
- 页面对象模型生成

✅ **可靠的 UI 测试**
- 无障碍树 = 无脆弱选择器
- 确定性交互
- 不易受 UI 变化影响而中断

✅ **CI/CD 集成**
- 管线中的无头模式
- 用自然语言生成 Playwright 测试文件
- 与测试管理工具集成

**理想工作流：** "为这个用户流写端到端测试" 或 "跨浏览器测试"

---

## 5. 令牌效率分析

| 工具 | 令牌使用 | 上下文占比 | 效率评级 |
|------|-------------|--------------|-------------------|
| Playwright MCP | ~13.7k | 6.8% | ⭐⭐⭐⭐⭐ 最佳 |
| Claude in Chrome | ~15.4k | 7.7% | ⭐⭐⭐⭐ 好 |
| Chrome DevTools MCP | ~19.0k | 9.5% | ⭐⭐⭐ 可接受 |

**影响：** 使用 200k 令牌上下文时：
- Playwright 剩余 186.3k 令牌用于你的工作
- Claude in Chrome 剩余 184.6k 令牌
- Chrome DevTools 剩余 181k 令牌

Playwright 和 Chrome DevTools 之间约 5.3k 令牌的差异在具有大量代码上下文的复杂会话中可能很重要。

---

## 6. 安全考虑

### Chrome DevTools MCP
- ✅ 默认隔离的浏览器配置文件
- ✅ 无云依赖
- ✅ 完全本地控制
- ⚠️ 远程调试端口安全（使用隔离配置文件）

### Claude in Chrome
- ⚠️ **无缓解措施时 23.6% 攻击成功率**（有防御措施时降至 11.2%）
- ⚠️ 使用你实际的浏览器会话（有 Cookie 泄露风险）
- ⚠️ 被阻止访问金融/成人/盗版网站
- ⚠️ 仍处于公测阶段，存在已知漏洞

### Playwright MCP
- ✅ 隔离浏览器上下文- ✅ 无云依赖
- ✅ 成熟的安全模型（Microsoft 支持）
- ✅ 可安全处理身份验证

---

## 7. 安装命令

### Chrome DevTools MCP

```bash
claude mcp add chrome-devtools npx chrome-devtools-mcp@latest
```

### Claude in Chrome

```
从 Chrome Web Store 安装（需要 Pro/Max/Team/Enterprise 计划）
```

### Playwright MCP（推荐）

```bash
# 首先，安装浏览器
npx playwright install

# 然后添加到 Claude Code（用户作用域 = 所有项目）
claude mcp add playwright -s user -- npx @playwright/mcp@latest
```

---

## 8. 建议

### **对于你的自动化测试工作流：**

#### 🥇 **主要工具：Playwright MCP**

**用于：** 日常端到端测试、跨浏览器验证、生成测试脚本

**原因：**
- 最低令牌使用（更多上下文可用于你的代码）
- 跨浏览器支持（Chrome、Firefox、Safari）
- 无障碍树方法 = 更可靠的选择器
- 优秀的 CI/CD 集成
- 可以生成实际的 Playwright 测试文件
- 免费，无需订阅

#### 🥈 **次要工具：Chrome DevTools MCP**

**用于：** 性能调试、网络分析、Core Web Vitals

**原因：**
- 性能追踪和调试无与伦比
- 深入的网络请求检查
- Google 官方工具，长期支持
- 需要回答 "为什么这么慢？" 时很重要

#### 🥉 **情境性：Claude in Chrome**

**用于：** 快速手动验证（已登录状态）、探索性测试、设计验证

**原因：**
- 开发过程中快速视觉检查很好用
- 可以读取你的登录状态
- 对 "看起来对吗？" 验证有用
- CI/CD 或正式测试自动化的跳过

---

## 9. 推荐设置

```bash
# 同时安装 Playwright 和 Chrome DevTools MCP
npx playwright install
claude mcp add playwright -s user -- npx @playwright/mcp@latest
claude mcp add chrome-devtools -s user -- npx chrome-devtools-mcp@latest
```

### 建议工作流

```
1. 开发        → Claude Code（终端）
2. 测试        → Playwright MCP（端到端、跨浏览器）
3. 调试        → Chrome DevTools MCP（性能、网络）
4. 验证        → Claude in Chrome（快速视觉检查）
5. CI/CD       → Playwright MCP（无头、自动化）
```

---

## 10. 最终结论

| 如果你需要... | 用这个 |
|----------------|----------|
| 跨浏览器端到端测试 | **Playwright MCP** |
| 性能分析 | **Chrome DevTools MCP** |
| 网络调试 | **Chrome DevTools MCP** |
| 快速视觉验证 | **Claude in Chrome** |
| CI/CD 自动化 | **Playwright MCP** |
| 测试脚本生成 | **Playwright MCP** |
| 最低令牌使用 | **Playwright MCP** |
| 已登录状态测试 | **Claude in Chrome** |
| 控制台日志调试 | **Chrome DevTools MCP** |

### **TL;DR 建议：**

**同时安装 Playwright MCP 和 Chrome DevTools MCP。** 将 Playwright 用作主要测试工具（令牌效率更高、跨浏览器、更适合端到端）。在需要深入性能分析或网络调试时使用 Chrome DevTools。仅在需要登录状态进行快速手动验证时使用 Claude in Chrome。

---

## 来源

- [Chrome DevTools MCP - GitHub](https://github.com/ChromeDevTools/chrome-devtools-mcp)
- [Anthropic - Piloting Claude in Chrome](https://claude.com/blog/claude-for-chrome)
- [Claude in Chrome Help Center](https://support.claude.com/en/articles/12012173-getting-started-with-claude-in-chrome)
- [Playwright MCP - GitHub](https://github.com/microsoft/playwright-mcp)
- [Simon Willison - Using Playwright MCP with Claude Code](https://til.simonwillison.net/claude-code/playwright-mcp-claude-code)
- [Testomat.io - Playwright MCP Claude Code](https://testomat.io/blog/playwright-mcp-claude-code/)
- [MCP Integration Guide - Scrapeless](https://www.scrapeless.com/en/blog/mcp-integration-guide)
- [Chrome DevTools MCP Guide - Vladimir Siedykh](https://vladimirsiedykh.com/blog/chrome-devtools-mcp-ai-browser-debugging-complete-guide-2025)
- [Addy Osmani - Give your AI eyes](https://addyosmani.com/blog/devtools-mcp/)

---

*本报告由 Claude Code 使用 Opus 4.5 模型生成于 2025 年 12 月 19 日。*
