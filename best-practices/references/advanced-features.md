# Claude Code 高级功能指南

> **时效性说明（2026-04）：** MCP 服务器列表、Plugin 生态、Agent Teams 等功能更新较快。涉及具体配置命令时建议联网确认最新文档。Hooks 的事件类型和 JSON 格式、Skill 的 frontmatter 字段等通常是稳定的。自 2026 年 4 月起，Claude CLI 已改为原生平台二进制（不再通过 Node.js 运行）。

## 目录
1. [Hooks（生命周期钩子）](#hooks)
2. [MCP（Model Context Protocol）](#mcp)
3. [Subagents（子代理）](#subagents)
4. [Skills（技能）](#skills)
5. [Computer Use（CLI 端屏幕操作）](#computer-use)
6. [Monitor 工具（后台事件监听）](#monitor-工具)
7. [Ultraplan（云端规划）](#ultraplan)
8. [Routines（模板化云端 Agent）](#routines)
9. [Agent Teams](#agent-teams)
10. [Plugins（插件）](#plugins)
11. [Claude Managed Agents API](#managed-agents)
12. [GitHub Actions / GitLab CI](#cicd-集成)

---

## Hooks

Hooks 是在 Claude Code 生命周期特定点自动执行的用户定义命令。与 CLAUDE.md 中的指令（建议性的）不同，hooks 是确定性的，保证会执行。

### Hook 类型

| 类型 | 说明 | 超时 |
|------|------|------|
| `command` | 运行 shell 命令 | 600s |
| `http` | 发送 POST 请求到 URL | 30s |
| `prompt` | 向 Claude 发送单轮提示评估 | 30s |
| `agent` | 生成可使用工具的 subagent | 60s |

### 常用 Hook 事件

| 事件 | 何时触发 | 能否阻止 |
|------|---------|---------|
| `SessionStart` | 会话开始/恢复 | 否 |
| `UserPromptSubmit` | 用户提交 prompt | 是 |
| `PreToolUse` | 工具执行前 | 是（允许/拒绝/修改） |
| `PostToolUse` | 工具成功执行后 | 否 |
| `Stop` | Claude 完成响应时 | 是（可要求继续） |
| `Notification` | 需要用户注意时 | 否 |
| `SessionEnd` | 会话结束 | 否 |

### 配置位置

| 位置 | 作用域 |
|------|--------|
| `~/.claude/settings.json` | 所有项目 |
| `.claude/settings.json` | 当前项目（团队共享） |
| `.claude/settings.local.json` | 当前项目（个人） |

### 示例：文件编辑后自动 lint

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx eslint --fix \"$CLAUDE_PROJECT_DIR\"/src"
          }
        ]
      }
    ]
  }
}
```

### 示例：阻止危险命令

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/block-rm.sh"
          }
        ]
      }
    ]
  }
}
```

```bash
#!/bin/bash
# .claude/hooks/block-rm.sh
COMMAND=$(jq -r '.tool_input.command')
if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "rm -rf 被阻止"
    }
  }'
else
  exit 0
fi
```

### 示例：桌面通知（macOS）

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude Code 需要你的注意\" with title \"Claude Code\"'"
          }
        ]
      }
    ]
  }
}
```

### 退出码含义

- **0**：成功，处理 JSON 输出
- **2**：阻止操作（stderr 中的消息会显示给用户）
- **其他**：非阻止错误，继续执行

### 让 Claude 帮你写 Hook

```
写一个 hook，每次文件编辑后运行 eslint
写一个 hook，阻止向 migrations 文件夹写入
```

用 `/hooks` 查看已配置的所有 hooks。

---

## MCP

MCP（Model Context Protocol）是连接 AI 工具到外部数据源的开放标准。通过 MCP，Claude Code 可以访问 Google Drive、Jira、Slack、Figma、数据库等外部服务。

### 添加 MCP 服务器

```bash
# 添加 stdio 类型服务器
claude mcp add my-server npx -y @some/mcp-server

# 添加 HTTP 类型服务器
claude mcp add my-http-server --transport http https://example.com/mcp

# 从配置文件加载
claude --mcp-config ./mcp.json
```

### MCP 配置文件格式

```json
{
  "mcpServers": {
    "memory": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    },
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "your-token"
      }
    }
  }
}
```

### 使用 MCP 资源

在 prompt 中用 `@server:resource` 格式引用：
```
显示 @github:repos/owner/repo/issues 的数据
```

### 常见使用场景

- 从 Jira/Linear 读取 issue 详情
- 查询数据库
- 集成 Figma 设计
- 访问 Slack 消息
- 读取 Google Drive 文档

---

## Subagents

Subagents 是在独立上下文窗口中运行的专门 AI 助手，拥有自定义系统提示、特定工具访问和独立权限。

### 内置 Subagents

| Agent | 模型 | 用途 |
|-------|------|------|
| Explore | Haiku（快速） | 只读代码搜索和探索 |
| Plan | 继承主会话 | Plan Mode 下的代码研究 |
| general-purpose | 继承主会话 | 复杂多步任务 |
| Claude Code Guide | Haiku | 回答关于 Claude Code 功能的问题 |

### 创建自定义 Subagent

**交互式创建：**
```
/agents
→ Create new agent
→ 选择 Personal（所有项目可用）或 Project（当前项目）
```

**手动创建文件：**

```markdown
# .claude/agents/code-reviewer.md
---
name: code-reviewer
description: 审查代码质量和最佳实践
tools: Read, Grep, Glob, Bash
model: sonnet
memory: project
---

你是高级代码审查员。被调用时，分析代码并提供关于质量、安全和最佳实践的具体可操作反馈。
```

### Subagent Frontmatter 字段

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | 是 | 唯一标识（小写+连字符） |
| `description` | 是 | 何时委托给此 agent |
| `tools` | 否 | 可用工具白名单 |
| `disallowedTools` | 否 | 工具黑名单 |
| `model` | 否 | sonnet/opus/haiku/inherit（Opus 4.7 现已可用） |
| `permissionMode` | 否 | 权限模式 |
| `maxTurns` | 否 | 最大轮次 |
| `skills` | 否 | 启动时预加载的 skills |
| `mcpServers` | 否 | MCP 服务器配置 |
| `hooks` | 否 | 生命周期 hooks |
| `memory` | 否 | 持久记忆范围：user/project/local |
| `background` | 否 | 是否总是后台运行 |
| `isolation` | 否 | 设为 `worktree` 在隔离 worktree 中运行 |
| `effort` | 否 | 推理深度：low/medium/high/xhigh/max（xhigh 为 Opus 4.7 默认） |

### 使用 Subagent

**自动委托：** Claude 根据任务自动选择合适的 subagent。

**显式调用：**
```
用 code-reviewer subagent 审查认证模块
```

**@-mention 强制使用：**
```
@"code-reviewer (agent)" 看看 auth 的变更
```

**整个会话作为 agent 运行：**
```bash
claude --agent code-reviewer
```

### 常用模式

**隔离高输出量操作：**
```
用 subagent 运行测试套件，只报告失败的测试及其错误信息
```

**并行研究：**
```
用独立的 subagents 并行研究认证、数据库和 API 模块
```

**链式 subagent：**
```
用 code-reviewer subagent 找到性能问题，然后用 optimizer subagent 修复
```

---

## Skills

Skills 扩展 Claude 的知识，创建可复用的工作流。与 CLAUDE.md（每次加载）不同，skills 按需加载。

### 创建 Skill

```bash
mkdir -p .claude/skills/api-conventions
```

```yaml
# .claude/skills/api-conventions/SKILL.md
---
name: api-conventions
description: REST API 设计规范
---

编写 API 端点时：
- URL 路径用 kebab-case
- JSON 属性用 camelCase
- 列表端点必须包含分页
- URL 路径中版本化 (/v1/, /v2/)
```

### Skill Frontmatter 字段

| 字段 | 说明 |
|------|------|
| `name` | 技能名称，即 `/slash-command` |
| `description` | Claude 用来判断何时使用此 skill |
| `disable-model-invocation` | `true` = 只能手动调用 |
| `user-invocable` | `false` = 只有 Claude 能调用 |
| `allowed-tools` | 此 skill 激活时可用的工具 |
| `model` | 使用的模型 |
| `context` | `fork` = 在 subagent 中运行 |
| `agent` | context: fork 时使用的 agent 类型 |

### 变量替换

| 变量 | 说明 |
|------|------|
| `$ARGUMENTS` | 调用时传入的所有参数 |
| `$ARGUMENTS[N]` / `$N` | 按索引访问参数 |
| `${CLAUDE_SKILL_DIR}` | skill 所在目录 |
| `${CLAUDE_SESSION_ID}` | 当前会话 ID |

### 动态上下文注入

用 `` !`command` `` 在 skill 发送给 Claude 之前运行命令：

```yaml
---
name: pr-summary
context: fork
agent: Explore
---

## PR 上下文
- 差异: !`gh pr diff`
- 评论: !`gh pr view --comments`
- 变更文件: !`gh pr diff --name-only`

## 任务
总结这个 PR...
```

### 内置 Skills

| Skill | 用途 |
|-------|------|
| `/batch <指令>` | 大规模并行变更（每个单元一个 worktree） |
| `/claude-api` | 加载 Claude API 参考 |
| `/debug [描述]` | 调试当前会话 |
| `/loop [间隔] <prompt>` | 按间隔重复执行 prompt（省略间隔时自适应节奏） |
| `/simplify [焦点]` | 审查并简化最近的代码变更 |
| `/ultrareview` | 云端代码审查：将分支分发给并行审查员，经过对抗性批评后返回验证报告 |
| `/team-onboarding` | 将你的开发环境设置打包为可重放的入职指南 |
| `/autofix-pr` | 从终端启用 PR 自动修复 |

---

## Computer Use

> **2026-04 新增（研究预览）：** Claude Code CLI 现在支持 Computer Use——Claude 可以在你的终端中打开原生应用、点击 UI 元素、输入文本、截图验证，无需离开终端。

### 适用场景

- 测试原生应用的 UI 交互（打开浏览器验证页面渲染）
- 调试视觉问题（截图对比、检查布局）
- 自动化 GUI-only 工具（只有图形界面的工具）
- 验证自身代码变更的视觉效果

### 启用方式

```bash
# CLI 启用
claude --computer-use

# 或在会话中
/computer-use
```

**平台支持：** 目前仅 macOS。Claude 可以打开应用、点击、输入、滚动、截图，并根据截图内容自主决定下一步操作。

### 与验证驱动开发的结合

Computer Use 最大的价值在于让 Claude 获得了**视觉验证**能力。以前你需要手动截图喂给 Claude，现在它可以自己截图、自己对比、自己修复——形成完整的视觉反馈闭环。

**参考文档：** https://code.claude.com/docs/en/computer-use

---

## Monitor 工具

> **2026-04 新增：** Monitor 工具让 Claude 在后台监听事件并实时响应，无需暂停对话。

### 工作方式

Monitor 将后台事件流注入到对话中，Claude 可以 tail 日志、监听变化并实时反应。

### 适用场景

- 监听构建日志，构建失败时自动分析错误
- 监控测试运行输出，实时报告失败用例
- 监听文件变化，触发相应操作

### 相关：`/loop` 自适应间隔

`/loop` 命令现在支持省略间隔参数，Claude 会根据任务自动调节执行节奏。

---

## Ultraplan

> **2026-04 新增（Week 15）：** Ultraplan 将规划任务从本地 CLI 交给云端 Claude Code on the web 会话处理。

### 工作流程

1. 在本地 CLI 发起规划请求
2. 云端自动创建环境，在 Plan Mode 下生成计划
3. 在浏览器中打开计划，可对具体章节评论、要求修改
4. 选择在云端执行或拉回本地执行

### 使用方式

```bash
# 启动 ultraplan
claude ultraplan "重构认证模块，支持 OAuth 2.0"
```

### 优势

- 规划过程不阻塞本地终端，你可以继续其他工作
- 浏览器编辑器支持结构化评论，比终端中讨论计划更高效
- 首次运行会自动创建云端环境

**参考文档：** https://code.claude.com/docs/en/ultraplan

---

## Routines

> **2026-04 新增（Week 16）：** Routines 是模板化的云端 Agent，可通过事件触发，无需本地机器运行。

### 核心概念

在 Claude Code on the web 上定义一次：设置 prompt、指定可访问的仓库、配置需要的连接器，然后通过事件自动触发。

### 支持的触发事件

- **PR opened** — 新 PR 创建时自动触发（如自动代码审查）
- **Release published** — 发布新版本时触发（如自动生成 changelog）
- **Webhook** — 任意 webhook 事件触发

### 与其他功能的区别

| 功能 | 运行位置 | 触发方式 | 持续性 |
|------|---------|---------|--------|
| Hooks | 本地 CLI | 生命周期事件 | 仅当 CLI 运行时 |
| Scheduled Tasks | 本地桌面应用 | 定时 | 需要应用保持打开 |
| **Routines** | **云端** | **事件驱动** | **无需本地机器** |

**参考文档：** https://code.claude.com/docs/en/routines

---

## Agent Teams

Agent Teams 让多个 Claude 会话协调工作，各有独立上下文，通过共享任务和消息进行沟通。

适用场景：
- 需要持续并行处理的任务
- 超出单个上下文窗口的工作
- 多个独立但相关的工作流

```bash
# 使用 teammate 模式
claude --teammate-mode tmux
```

---

## Plugins

Plugins 将 skills、hooks、subagents 和 MCP 服务器打包为一个可安装的单元。

```
/plugin  # 浏览插件市场
```

对于类型语言，安装 code intelligence 插件可以给 Claude 精确的符号导航和编辑后自动错误检测。

---

## Managed Agents

> **2026-04 新增（公测）：** Claude Managed Agents 是 Anthropic 提供的托管 Agent 服务，用于运行长时间自主 Agent 任务。

### 核心概念

Managed Agents 是一套完全托管的 Agent 基础设施，包含安全沙箱、内置工具和 Server-Sent Events 流式传输。你通过 API 创建 Agent、配置容器、运行会话。

### API 组成

| API | 用途 |
|-----|------|
| Sessions API | 运行有状态的 Agent 会话（云端容器中） |
| Environments API | 配置 Agent 的容器模板 |
| Agent Memory | Agent 的跨会话记忆（公测中） |

### 适用场景

- 需要长时间运行的自主 Agent 任务
- 不想自建 Agent 基础设施
- 需要安全沙箱隔离的 Agent 执行环境

**注意：** 所有 API 请求需要 `managed-agents-2026-04-01` beta header。

**参考文档：** https://docs.anthropic.com/en/docs/agents/managed-agents

---

## CI/CD 集成

### GitHub Actions

```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review
on: pull_request

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Review with Claude
        run: |
          claude -p "审查这个 PR 的代码变更，关注安全和性能问题" \
            --output-format json > review.json
```

### GitLab CI/CD

Claude Code 也支持 GitLab CI/CD 集成，用于自动化代码审查和 issue 分类。

---

## 官方文档链接

- Hooks：https://code.claude.com/docs/en/hooks
- MCP：https://code.claude.com/docs/en/mcp
- Subagents：https://code.claude.com/docs/en/sub-agents
- Skills：https://code.claude.com/docs/en/skills
- Computer Use：https://code.claude.com/docs/en/computer-use
- Ultraplan：https://code.claude.com/docs/en/ultraplan
- Routines：https://code.claude.com/docs/en/routines
- Agent Teams：https://code.claude.com/docs/en/agent-teams
- Plugins：https://code.claude.com/docs/en/plugins
- Managed Agents：https://docs.anthropic.com/en/docs/agents/managed-agents
- 功能概览：https://code.claude.com/docs/en/features-overview
- GitHub Actions：https://code.claude.com/docs/en/github-actions
- What's New：https://code.claude.com/docs/en/whats-new
