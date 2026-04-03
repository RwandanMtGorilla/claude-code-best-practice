# 理解 Claude 技能发现在大型单体仓库中

在大型单体仓库中使用 Claude Code 时，理解技能如何被发现和加载到上下文中对于组织项目特定能力至关重要。

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## 与 CLAUDE.md 的重要区别

**技能不具有与 CLAUDE.md 文件相同的加载行为。** CLAUDE.md 文件会向上遍历目录树（祖先加载），而技能使用不同的发现机制，专注于项目内的嵌套目录。

## 技能如何被发现

### 1. 标准技能位置

技能根据作用域从以下固定位置加载：

| 位置 | 路径 | 适用于 |
|----------|------|------------|
| 企业 | 管理设置 | 组织中的所有用户 |
| 个人 | `~/.claude/skills/<skill-name>/SKILL.md` | 你的所有项目 |
| 项目 | `.claude/skills/<skill-name>/SKILL.md` | 仅此项目 |
| 插件 | `<plugin>/skills/<skill-name>/SKILL.md` | 启用插件的位置 |

### 2. 嵌套目录的自动发现

当你在子目录中使用文件时，Claude Code 会在嵌套的 `.claude/skills/` 目录中自动发现技能。例如，如果你在 `packages/frontend/` 中编辑文件，Claude Code 还会在 `packages/frontend/.claude/skills/` 中寻找技能。

这支持单体仓库中包拥有自己的技能的情况。

## 典型单体仓库结构

考虑典型的具有分离包的单体仓库：

```
/mymonorepo/
├── .claude/
│   └── skills/
│       └── shared-conventions/SKILL.md    # 项目级技能
├── packages/
│   ├── frontend/
│   │   ├── .claude/
│   │   │   └── skills/
│   │   │       └── react-patterns/SKILL.md  # 前端特定技能
│   │   └── src/
│   │       └── App.tsx
│   ├── backend/
│   │   ├── .claude/
│   │   │   └── skills/
│   │   │       └── api-design/SKILL.md      # 后端特定技能
│   │   └── src/
│   └── shared/
│       ├── .claude/
│       │   └── skills/
│       │       └── utils-patterns/SKILL.md  # 共享工具技能
│       └── src/
```

## 场景 1：刚在根目录启动 Claude（尚未编辑文件）

当你从 `/mymonorepo/` 运行 Claude Code 且尚未编辑任何文件时：

```bash
cd /mymonorepo
claude
# 刚启动 -- 尚未编辑文件
```

| 技能 | 在上下文中？ | 原因 |
|-------|-------------|--------|
| `shared-conventions` | **是** | 根目录 `.claude/skills/` 中的项目级技能 |
| `react-patterns` | **否** | 未被发现 —— 尚未处理 `packages/frontend/` 中的文件 |
| `api-design` | **否** | 未被发现 —— 尚未处理 `packages/backend/` 中的文件 |
| `utils-patterns` | **否** | 未被发现 —— 尚未处理 `packages/shared/` 中的文件 |

## 场景 2：在包中编辑文件后

在你请求 Claude 编辑 `packages/frontend/src/App.tsx` 之后：

| 技能 | 在上下文中？ | 原因 |
|-------|-------------|--------|
| `shared-conventions` | **是** | 根目录 `.claude/skills/` 中的项目级技能 |
| `react-patterns` | **是** | 在编辑 `packages/frontend/` 中的文件时被发现 |
| `api-design` | **否** | 仍未被发现 —— 尚未处理 `packages/backend/` 中的文件 |
| `utils-patterns` | **否** | 仍未被发现 —— 尚未处理 `packages/shared/` 中的文件 |

**关键洞察**：嵌套技能在你处理这些目录中的文件时**按需发现**。它们不会在会话启动时预加载。

## 关键行为：描述 vs 完整内容

技能描述被加载到上下文中，Claude 知道有什么可用，但**仅在调用时才加载完整技能内容**。这是一个重要的优化：

- **描述**：始终在上下文中（在字符预算范围内）
- **完整内容**：调用时按需加载

注意：预加载技能的子代理工作方式不同 —— 完整技能内容在启动时注入。

## 优先级顺序（当技能共享名称时）

当技能在跨层级共享相同名称时，优先级较高的位置获胜：

| 优先级 | 位置 | 作用域 |
|----------|----------|-------|
| 1（最高） | 企业 | 组织范围 |
| 2 | 个人（`~/.claude/skills/`） | 你的所有项目 |
| 3（最低） | 项目（`.claude/skills/`） | 仅此项目 |

插件技能使用 `plugin-name:skill-name` 命名空间，因此不会与其他层级冲突。

## 为什么这个设计对单体仓库有效

- **包特定技能保持隔离** —— 在 `packages/frontend/` 工作的前端开发者获得前端特定技能，而不会被后端技能 clutter 上下文。

- **自动发现减少配置** —— 无需显式注册包级技能；在你处理这些目录时自动发现。

- **上下文进行了优化** —— 仅初始加载技能描述，嵌套技能按需发现。

- **各团队可以维护自己的技能** —— 每个包团队可以定义特定于其领域的技能，无需与其他团队协调。

## 字符预算考虑

技能描述加载到上下文中时受字符预算限制（默认 15,000 字符）。在拥有大量包和技能的大型单体仓库中，你可能会触达这个限制。

- 运行 `/context` 检查是否有技能被排除的警告
- 设置 `SLASH_COMMAND_TOOL_CHAR_BUDGET` 环境变量来增加限制

## 最佳实践

1. **将共享工作流放在根目录 `.claude/skills/`** —— 仓库范围的约定、提交工作流和共享模式。

2. **将包特定技能放在包 `.claude/skills/` 中** —— 框架特定模式、组件约定、该包独有的测试工具。

3. **对危险技能使用 `disable-model-invocation: true`** —— 部署或破坏性技能应要求显式用户调用。

4. **保持技能描述简洁** —— 描述始终在上下文中（最多到字符预算），因此冗长的描述会浪费上下文空间。

5. **使用技能名称命名空间** —— 考虑用包名前缀（例如 `frontend-review`、`backend-deploy`）来避免混淆。

## 对比：技能 vs CLAUDE.md 加载

| 行为 | CLAUDE.md | 技能 |
|----------|-----------|--------|
| 祖先加载（UP 目录树） | 是 | 否 |
| 嵌套/后代发现（DOWN 目录树） | 是（惰性） | 是（自动发现） |
| 全局位置 | `~/.claude/CLAUDE.md` | `~/.claude/skills/` |
| 项目位置 | `.claude/` 或仓库根目录 | `.claude/skills/` |
| 内容加载 | 完整内容 | 仅描述（调用时完整） |

---

## 来源

- [Claude Code 文档 - Extend Claude with Skills](https://code.claude.com/docs/en/skills)
- [Claude Code 文档 - 嵌套目录的自动发现](https://code.claude.com/docs/en/skills#automatic-discovery-from-nested-directories)
