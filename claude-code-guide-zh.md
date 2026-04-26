# Claude Code 完全指南

> 原文作者：[cogsec (@affaanmustafa)](https://x.com/affaanmustafa)
> 原文链接：https://x.com/affaanmustafa/status/2012378465664745795
> 发布日期：2026年1月17日

---

以下是我经过10个月日常使用后的完整配置：技能、钩子、子代理、MCP、插件，以及哪些真正有效。

自从2月份 Claude Code 实验性推出以来，我一直是忠实用户，并与 @DRodriguezFX 一起使用 Claude Code 完全开发了 Zenith，在 Anthropic x Forum Ventures 黑客马拉松中获胜。

## 技能和命令

技能类似于规则，限定在特定范围和工作流内。当你需要执行特定工作流时，它们是提示的快捷方式。

经过长时间使用 Opus 4.5 编码后，想要清理无用代码和散落的 .md 文件？运行 `/refactor-clean`。需要测试？`/tdd`、`/e2e`、`/test-coverage`。技能和命令可以在单个提示中链式组合。

我可以创建一个在检查点更新 codemap 的技能——让 Claude 能够快速导航你的代码库，而不用在探索上消耗上下文。

`~/.claude/skills/codemap-updater.md`

命令是通过斜杠命令执行的技能。它们有重叠但存储方式不同：

- **技能：** `~/.claude/skills` - 更广泛的工作流定义
- **命令：** `~/.claude/commands` - 快速可执行的提示

## 钩子

钩子是基于触发器的自动化，在特定事件时触发。与技能不同，它们限定在工具调用和生命周期事件内。

**钩子类型**

1. **PreToolUse** - 工具执行前（验证、提醒）
2. **PostToolUse** - 工具完成后（格式化、反馈循环）
3. **UserPromptSubmit** - 当你发送消息时
4. **Stop** - 当 Claude 完成响应时
5. **PreCompact** - 上下文压缩前
6. **Notification** - 权限请求时

**示例：** 在长时间运行命令前的 tmux 提醒

```json
{
  "PreToolUse": [
    {
      "matcher": "tool == \"Bash\" && tool_input.command matches \"(npm|pnpm|yarn|cargo|pytest)\"",
      "hooks": [
        {
          "type": "command",
          "command": "if [ -z \"$TMUX\" ]; then echo '[Hook] Consider tmux for session persistence' >&2; fi"
        }
      ]
    }
  ]
}
```

**专业提示：** 使用 `hookify` 插件以对话方式创建钩子，而不是手动编写 JSON。运行 `/hookify` 并描述你想要的内容。

## 子代理

子代理是你的编排器（主 Claude）可以委派任务的进程，具有有限的作用域。它们可以在后台或前台运行，为主代理释放上下文。

子代理与技能配合得很好——能够执行你技能子集的子代理可以被委派任务并自主使用这些技能。它们也可以通过特定的工具权限进行沙盒化配置。

```bash
# 子代理结构示例
~/.claude/agents/
  planner.md           # 功能实现规划
  architect.md         # 系统设计决策
  tdd-guide.md         # 测试驱动开发
  code-reviewer.md     # 质量/安全审查
  security-reviewer.md # 漏洞分析
  build-error-resolver.md
  e2e-runner.md
  refactor-cleaner.md
```

为每个子代理配置允许的工具、MCP 和权限，以实现正确的范围控制。

## 规则和记忆

你的 `.rules` 文件夹包含 `.md` 文件，其中包含 Claude 应始终遵循的最佳实践。两种方法：

1. **单一 CLAUDE.md** - 所有内容在一个文件中（用户或项目级别）
2. **规则文件夹** - 按关注点分组的模块化 `.md` 文件

**规则示例：**

```bash
~/.claude/rules/
  security.md      # 强制安全检查
  coding-style.md  # 不可变性、文件大小限制
  testing.md       # TDD、80% 覆盖率
  git-workflow.md  # 约定式提交
  agents.md        # 子代理委派规则
  patterns.md      # API 响应格式
  performance.md   # 模型选择（Haiku vs Sonnet vs Opus）
  hooks.md         # 钩子文档
```

示例规则：
- 代码库中不使用 emoji
- 前端避免使用紫色调
- 部署前始终测试代码
- 优先使用模块化代码而非大文件
- 永远不要提交 console.log

## MCP（模型上下文协议）

MCP 将 Claude 直接连接到外部服务。不是 API 的替代品——它是围绕 API 的提示驱动包装器，在导航信息时提供更多灵活性。

**示例：** Supabase MCP 让 Claude 可以拉取特定数据，直接在上游运行 SQL，无需复制粘贴。数据库、部署平台等也是如此。

**Chrome in Claude：** 是一个内置的插件 MCP，让 Claude 可以自主控制你的浏览器——点击查看事物如何运作。

**关键：上下文窗口管理**

对 MCP 要挑剔。我把所有 MCP 都放在用户配置中，但禁用所有未使用的。导航到 `/plugins` 并向下滚动，或运行 `/mcp`。

启用太多工具后，你的 200k 上下文窗口在压缩前可能只有 70k。性能会显著下降。

**经验法则：** 配置中保留 20-30 个 MCP，但保持不到 10 个启用 / 不到 80 个工具活跃。

## 插件

插件将工具打包以便于安装，而不是繁琐的手动设置。插件可以是技能 + MCP 的组合，或是钩子/工具的捆绑。

**安装插件：**

```bash
# 添加市场
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep

# 打开 Claude，运行 /plugins，找到新市场，从那里安装
```

**LSP 插件：** 如果你经常在编辑器外运行 Claude Code，特别有用。语言服务器协议为 Claude 提供实时类型检查、跳转定义和智能补全，无需打开 IDE。

与 MCP 一样的警告——注意你的上下文窗口。

## 技巧和窍门

**键盘快捷键**

- `Ctrl+U` - 删除整行（比反复按退格更快）
- `!` - 快速 bash 命令前缀
- `@` - 搜索文件
- `/` - 启动斜杠命令
- `Shift+Enter` - 多行输入
- `Tab` - 切换思考显示
- `Esc Esc` - 中断 Claude / 恢复代码

**并行工作流**

- `/fork` - 分叉对话以并行执行不重叠的任务，而不是发送排队消息
- **Git Worktrees** - 用于不冲突的并行 Claude 重叠操作。每个 worktree 是一个独立的检出

```bash
git worktree add ../feature-branch feature-branch
# 现在在每个 worktree 中运行独立的 Claude 实例
```

**tmux 用于长时间运行的命令：** 流式传输和监视 Claude 运行的日志/bash 进程。

```bash
tmux new -s dev
# Claude 在这里运行命令，你可以分离和重新连接
tmux attach -t dev
```

**mgrep > grep：** `mgrep` 是对 ripgrep/grep 的重大改进。通过插件市场安装，然后使用 `/mgrep` 技能。支持本地搜索和网络搜索。

```bash
mgrep "function handleSubmit"  # 本地搜索
mgrep --web "Next.js 15 app router changes"  # 网络搜索
```

**其他有用命令**

- `/rewind` - 回到之前的状态
- `/statusline` - 自定义分支、上下文 %、待办事项
- `/checkpoints` - 文件级撤销点
- `/compact` - 手动触发上下文压缩

**GitHub Actions CI/CD**

使用 GitHub Actions 设置 PR 的代码审查。配置后，Claude 可以自动审查 PR。

**沙盒化**

对危险操作使用沙盒模式——Claude 在受限环境中运行，不会影响你的实际系统。（使用 `--dangerously-skip-permissions` 则相反，让 Claude 自由运行，如果不小心可能会造成破坏。）

## 关于编辑器

虽然不需要编辑器，但它可以正面或负面地影响你的 Claude Code 工作流。虽然 Claude Code 可以在任何终端中工作，但搭配一个有能力的编辑器可以解锁实时文件跟踪、快速导航和集成命令执行。

**Zed（我的偏好）**

我使用 Zed——一个基于 Rust 的编辑器，轻量、快速且高度可定制。

**为什么 Zed 与 Claude Code 配合得很好：**

- **Agent Panel 集成** - Zed 的 Claude 集成让你可以实时跟踪 Claude 编辑时的文件变化。无需离开编辑器即可跳转到 Claude 引用的文件
- **性能** - 用 Rust 编写，即时打开，处理大型代码库无延迟
- **CMD+Shift+R 命令面板** - 快速访问所有自定义斜杠命令、调试器和可搜索 UI 中的工具
- **最小资源使用** - 在重度操作期间不会与 Claude 竞争系统资源
- **Vim 模式** - 完整的 vim 键绑定

**分屏技巧：**
1. 分屏——一边是运行 Claude Code 的终端，另一边是编辑器
2. `Ctrl + G` - 快速在 Zed 中打开 Claude 当前正在处理的文件
3. 自动保存 - 启用自动保存，这样 Claude 的文件读取始终是最新的
4. Git 集成 - 使用编辑器的 git 功能在提交前审查 Claude 的更改
5. 文件监视器 - 大多数编辑器会自动重新加载更改的文件，确保此功能已启用

**VSCode / Cursor**

这也是一个可行的选择，与 Claude Code 配合良好。你可以使用终端格式，通过 `\ide` 与编辑器自动同步启用 LSP 功能（现在与插件有些重复）。或者你可以选择扩展版本，它与编辑器集成更紧密，有匹配的 UI。

## 我的配置

**插件**

已安装：（我通常一次只启用其中 4-5 个）

```markdown
ralph-wiggum@claude-code-plugins       # 循环自动化
frontend-design@claude-code-plugins    # UI/UX 模式
commit-commands@claude-code-plugins    # Git 工作流
security-guidance@claude-code-plugins  # 安全检查
pr-review-toolkit@claude-code-plugins  # PR 自动化
typescript-lsp@claude-plugins-official # TS 智能
hookify@claude-plugins-official        # 钩子创建
code-simplifier@claude-plugins-official
feature-dev@claude-code-plugins
explanatory-output-style@claude-code-plugins
code-review@claude-code-plugins
context7@claude-plugins-official       # 实时文档
pyright-lsp@claude-plugins-official    # Python 类型
mgrep@Mixedbread-Grep                  # 更好的搜索
```

**MCP 服务器**

配置（用户级别）：

```json
{
  "github": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"] },
  "firecrawl": { "command": "npx", "args": ["-y", "firecrawl-mcp"] },
  "supabase": {
    "command": "npx",
    "args": ["-y", "@supabase/mcp-server-supabase@latest", "--project-ref=YOUR_REF"]
  },
  "memory": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-memory"] },
  "sequential-thinking": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
  },
  "vercel": { "type": "http", "url": "https://mcp.vercel.com" },
  "railway": { "command": "npx", "args": ["-y", "@railway/mcp-server"] },
  "cloudflare-docs": { "type": "http", "url": "https://docs.mcp.cloudflare.com/mcp" },
  "cloudflare-workers-bindings": {
    "type": "http",
    "url": "https://bindings.mcp.cloudflare.com/mcp"
  },
  "cloudflare-workers-builds": { "type": "http", "url": "https://builds.mcp.cloudflare.com/mcp" },
  "cloudflare-observability": {
    "type": "http",
    "url": "https://observability.mcp.cloudflare.com/mcp"
  },
  "clickhouse": { "type": "http", "url": "https://mcp.clickhouse.cloud/mcp" },
  "AbletonMCP": { "command": "uvx", "args": ["ableton-mcp"] },
  "magic": { "command": "npx", "args": ["-y", "@magicuidesign/mcp@latest"] }
}
```

按项目禁用（上下文窗口管理）：

```markdown
# 在 ~/.claude.json 的 projects.[path].disabledMcpServers 下
disabledMcpServers: [
  "playwright",
  "cloudflare-workers-builds",
  "cloudflare-workers-bindings",
  "cloudflare-observability",
  "cloudflare-docs",
  "clickhouse",
  "AbletonMCP",
  "context7",
  "magic"
]
```

关键是——我配置了 14 个 MCP，但每个项目只启用约 5-6 个。保持上下文窗口健康。

**关键钩子**

```json
{
  "PreToolUse": [
    { "matcher": "npm|pnpm|yarn|cargo|pytest", "hooks": ["tmux 提醒"] },
    { "matcher": "Write && .md file", "hooks": ["除非 README/CLAUDE 否则阻止"] },
    { "matcher": "git push", "hooks": ["打开编辑器审查"] }
  ],
  "PostToolUse": [
    { "matcher": "Edit && .ts/.tsx/.js/.jsx", "hooks": ["prettier --write"] },
    { "matcher": "Edit && .ts/.tsx", "hooks": ["tsc --noEmit"] },
    { "matcher": "Edit", "hooks": ["grep console.log 警告"] }
  ],
  "Stop": [
    { "matcher": "*", "hooks": ["检查修改的文件中的 console.log"] }
  ]
}
```

**自定义状态栏**

显示用户、目录、带脏指示器的 git 分支、剩余上下文 %、模型、时间和待办事项数量。

**规则结构**

```bash
~/.claude/rules/
  security.md      # 不硬编码密钥、验证输入
  coding-style.md  # 不可变性、文件组织
  testing.md       # TDD 工作流、80% 覆盖率
  git-workflow.md  # 提交格式、PR 流程
  agents.md        # 何时委派给子代理
  performance.md   # 模型选择、上下文管理
```

**子代理**

```markdown
~/.claude/agents/
  planner.md           # 分解功能
  architect.md         # 系统设计
  tdd-guide.md         # 先写测试
  code-reviewer.md     # 质量审查
  security-reviewer.md # 漏洞扫描
  build-error-resolver.md
  e2e-runner.md        # Playwright 测试
  refactor-cleaner.md  # 无用代码删除
  doc-updater.md       # 保持文档同步
```

## 关键要点

1. **不要过度复杂化** - 把配置当作微调，而不是架构
2. **上下文窗口是宝贵的** - 禁用未使用的 MCP 和插件
3. **并行执行** - 分叉对话，使用 git worktrees
4. **自动化重复任务** - 用钩子处理格式化、lint、提醒
5. **限定子代理范围** - 有限的工具 = 专注的执行

## 参考资料

- [插件参考](https://code.claude.com/docs/en/plugins-reference)
- [钩子文档](https://code.claude.com/docs/en/hooks)
- [检查点](https://code.claude.com/docs/en/checkpointing)
- [交互模式](https://code.claude.com/docs/en/interactive-mode)
- [记忆系统](https://code.claude.com/docs/en/memory)
- [子代理](https://code.claude.com/docs/en/sub-agents)
- [MCP 概述](https://code.claude.com/docs/en/mcp-overview)

---

**注意：** 这只是部分内容。如果大家感兴趣，我可能会发布更多关于具体细节的帖子。
