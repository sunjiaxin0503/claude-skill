# Claude Code 基础指南

> **时效性说明（2026-03）：** 本文档中的 CLI 命令和 flags 可能随版本更新变化。如果用户问到具体命令格式，建议联网确认 `code.claude.com/docs/en/cli-reference` 的最新版本。安装方式、会话管理、CLAUDE.md 编写原则等内容通常是稳定的。

## 目录
1. [安装与更新](#安装与更新)
2. [核心 CLI 命令](#核心-cli-命令)
3. [常用 CLI Flags](#常用-cli-flags)
4. [会话管理](#会话管理)
5. [权限与安全](#权限与安全)
6. [设置与配置](#设置与配置)
7. [CLAUDE.md 记忆系统](#claudemd-记忆系统)
8. [IDE 集成](#ide-集成)
9. [交互模式快捷键](#交互模式快捷键)

---

## 安装与更新

### 原生安装（推荐，自动更新）

**macOS / Linux / WSL：**
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows PowerShell：**
```powershell
irm https://claude.ai/install.ps1 | iex
```

### 其他安装方式

- **Homebrew**：`brew install --cask claude-code`（需手动 `brew upgrade claude-code`）
- **WinGet**：`winget install Anthropic.ClaudeCode`（需手动 `winget upgrade`）

### 更新

```bash
claude update
```

### 启动

```bash
cd your-project
claude
```

首次使用会提示登录。

---

## 核心 CLI 命令

| 命令 | 说明 | 示例 |
|------|------|------|
| `claude` | 启动交互式会话 | `claude` |
| `claude "query"` | 带初始提示启动会话 | `claude "explain this project"` |
| `claude -p "query"` | 非交互模式（执行完退出） | `claude -p "explain this function"` |
| `cat file \| claude -p "query"` | 管道输入处理 | `cat logs.txt \| claude -p "explain"` |
| `claude -c` | 继续当前目录最近的会话 | `claude -c` |
| `claude -r "name" "query"` | 按名称或 ID 恢复会话 | `claude -r "auth-refactor" "继续"` |
| `claude update` | 更新到最新版本 | `claude update` |
| `claude auth login` | 登录（支持 `--sso`、`--console`） | `claude auth login --console` |
| `claude auth status` | 查看认证状态 | `claude auth status --text` |
| `claude agents` | 列出所有已配置的 subagents | `claude agents` |
| `claude mcp` | 管理 MCP 服务器 | `claude mcp add` |

---

## 常用 CLI Flags

### 模型与性能
| Flag | 说明 | 示例 |
|------|------|------|
| `--model` | 选择模型（sonnet/opus/完整名称） | `claude --model opus` |
| `--effort` | 推理深度：low/medium/high/max | `claude --effort high` |
| `--bare` | 精简模式：跳过自动发现，启动更快 | `claude --bare -p "query"` |

### 系统提示
| Flag | 说明 |
|------|------|
| `--system-prompt "text"` | 替换整个系统提示 |
| `--append-system-prompt "text"` | 在默认提示后追加内容 |
| `--system-prompt-file path` | 从文件加载替换系统提示 |
| `--append-system-prompt-file path` | 从文件追加系统提示 |

大多数情况下使用 `--append-system-prompt`，保留内置能力的同时添加你的要求。

### 权限控制
| Flag | 说明 |
|------|------|
| `--allowedTools "Tool1" "Tool2"` | 免提示权限的工具白名单 |
| `--disallowedTools "Tool1"` | 完全禁用的工具 |
| `--permission-mode plan` | 以指定权限模式启动 |
| `--dangerously-skip-permissions` | 跳过所有权限提示（谨慎使用） |

### 输出格式
| Flag | 说明 |
|------|------|
| `--output-format text` | 纯文本输出（默认） |
| `--output-format json` | JSON 格式输出 |
| `--output-format stream-json` | 流式 JSON（实时处理） |

### 会话管理
| Flag | 说明 |
|------|------|
| `--continue` / `-c` | 继续最近的会话 |
| `--resume` / `-r` | 恢复指定会话 |
| `--name` / `-n` | 给会话命名 |
| `--worktree` / `-w` | 在隔离的 git worktree 中工作 |
| `--fork-session` | 恢复时创建分叉（不修改原会话） |

### 预算控制
| Flag | 说明 |
|------|------|
| `--max-budget-usd 5.00` | 最大花费限制（仅 print 模式） |
| `--max-turns 3` | 最大轮次限制（仅 print 模式） |

### 其他实用 Flags
| Flag | 说明 |
|------|------|
| `--add-dir ../other-project` | 添加额外工作目录 |
| `--chrome` | 启用 Chrome 浏览器集成 |
| `--mcp-config ./mcp.json` | 加载 MCP 配置文件 |
| `--agent my-agent` | 以指定 agent 身份运行整个会话 |
| `--remote "task"` | 在 claude.ai 创建远程会话 |
| `--teleport` | 将远程会话拉到本地终端 |
| `--verbose` | 显示详细日志（包括思考过程） |
| `--debug` | 启用调试模式 |

---

## 会话管理

### 命名会话

```bash
# 启动时命名
claude -n "auth-refactor"

# 会话中重命名
/rename auth-refactor
```

### 恢复会话

```bash
# 继续最近的会话
claude --continue

# 按名称恢复
claude --resume auth-refactor

# 交互式选择
claude --resume
```

### 会话选择器快捷键

| 快捷键 | 操作 |
|--------|------|
| `↑`/`↓` | 导航会话 |
| `Enter` | 选择并恢复 |
| `P` | 预览会话内容 |
| `R` | 重命名会话 |
| `/` | 搜索过滤 |
| `A` | 切换当前目录/所有项目 |
| `B` | 按当前 git 分支过滤 |

### Git Worktrees（并行会话）

```bash
# 在隔离的 worktree 中启动
claude --worktree feature-auth

# 自动生成名称
claude --worktree
```

Worktrees 创建在 `<repo>/.claude/worktrees/<name>`，退出时无更改自动清理。

---

## 权限与安全

### 权限模式

通过 `Shift+Tab` 在会话中切换：

| 模式 | 说明 |
|------|------|
| Normal | 标准模式，需要逐一审批 |
| Auto-Accept | 自动接受文件编辑 |
| Plan | 只读分析模式，不做任何修改 |

### 配置权限白名单

在 `/permissions` 中添加安全命令：

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run lint)",
      "Bash(git commit *)",
      "Bash(npm test *)",
      "Read"
    ]
  }
}
```

### 沙箱模式

使用 `/sandbox` 启用 OS 级隔离，限制文件系统和网络访问。

---

## 设置与配置

### 配置作用域

| 作用域 | 位置 | 共享 |
|--------|------|------|
| Managed | 系统级（IT 部署） | 是（组织范围） |
| User | `~/.claude/settings.json` | 否（个人） |
| Project | `.claude/settings.json` | 是（提交到 git） |
| Local | `.claude/settings.local.json` | 否（gitignore） |

### 常用配置

```json
// .claude/settings.json
{
  "permissions": {
    "defaultMode": "plan",
    "allow": ["Bash(npm test *)", "Read"],
    "deny": ["Agent(Explore)"]
  },
  "autoMemoryEnabled": true
}
```

使用 `/config` 在交互模式中打开设置界面。

---

## CLAUDE.md 记忆系统

### 文件位置与作用域

| 位置 | 作用域 | 用途 |
|------|--------|------|
| `~/.claude/CLAUDE.md` | 所有项目 | 个人偏好 |
| `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 当前项目 | 团队共享规范 |
| 子目录 `CLAUDE.md` | 按需加载 | 模块级指令 |

### 快速生成

```bash
# 自动分析项目并生成
/init

# 交互式多阶段流程（需设置环境变量）
CLAUDE_CODE_NEW_INIT=true claude
```

### 编写要点

- **目标 200 行以内**：过长会降低遵循率
- **具体可验证**：「使用 2 空格缩进」而非「格式化代码」
- **只写 Claude 猜不到的**：Claude 能从代码推断的不需要写
- **定期精简**：像代码一样维护，删除过时规则

### 导入其他文件

```markdown
# CLAUDE.md
参考 @README.md 了解项目概况
参考 @package.json 了解可用命令

# 个人偏好（不提交到 git）
- @~/.claude/my-project-instructions.md
```

### 规则文件 .claude/rules/

将指令拆分为多个文件，支持按文件路径触发：

```markdown
# .claude/rules/api-design.md
---
paths:
  - "src/api/**/*.ts"
---

所有 API 端点必须包含输入验证
使用标准错误响应格式
```

### Auto Memory（自动记忆）

Claude 自动积累跨会话的知识，存储在 `~/.claude/projects/<project>/memory/`。

- 默认开启，用 `/memory` 查看和管理
- 仅前 200 行在每次会话开始时加载
- 可以手动编辑或删除记忆文件

---

## IDE 集成

### VS Code

安装「Claude Code」扩展，支持内联 diff、@-mentions、计划审查。

快捷键 `Cmd+Shift+P` → "Claude Code" → "Open in New Tab"

### JetBrains

安装 Claude Code 插件，支持 IntelliJ、PyCharm、WebStorm 等。

### 桌面应用

独立应用，支持可视化 diff 审查、多会话并行、定时任务、云端会话。

- macOS 和 Windows 可用
- 下载地址：https://claude.ai/api/desktop/

---

## 交互模式快捷键

| 快捷键 | 操作 |
|--------|------|
| `Esc` | 停止当前操作 |
| `Esc + Esc` | 打开回溯菜单 |
| `Shift+Tab` | 切换权限模式 |
| `Ctrl+G` | 在编辑器中打开计划 |
| `Ctrl+O` | 切换 verbose 模式（查看思考过程） |
| `Ctrl+B` | 将当前任务转为后台运行 |
| `Option+T` / `Alt+T` | 切换思考模式 |

### 常用斜杠命令

| 命令 | 说明 |
|------|------|
| `/init` | 生成 CLAUDE.md |
| `/clear` | 清空上下文 |
| `/compact [指令]` | 压缩上下文 |
| `/memory` | 查看/编辑记忆文件 |
| `/config` | 打开设置 |
| `/permissions` | 管理权限 |
| `/hooks` | 查看已配置的 hooks |
| `/agents` | 管理 subagents |
| `/resume` | 选择并恢复会话 |
| `/rename name` | 重命名当前会话 |
| `/rewind` | 回溯到检查点 |
| `/effort` | 调整推理深度 |
| `/model` | 切换模型 |
| `/btw question` | 侧边快速提问（不进入上下文） |
| `/batch instruction` | 大规模并行变更 |
| `/debug` | 调试当前会话 |
| `/statusline` | 配置状态栏 |

---

## 官方文档链接

- 概览：https://code.claude.com/docs/en/overview
- CLI 参考：https://code.claude.com/docs/en/cli-reference
- 快速开始：https://code.claude.com/docs/en/quickstart
- 设置：https://code.claude.com/docs/en/settings
- 疑难排查：https://code.claude.com/docs/en/troubleshooting
