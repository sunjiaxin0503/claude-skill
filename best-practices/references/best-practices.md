# Claude Code 最佳实践与工作流

## 目录
1. [功能选型决策树](#功能选型决策树)
2. [核心理念](#核心理念)
3. [高效工作流模式](#高效工作流模式)
4. [Prompt 技巧](#prompt-技巧)
5. [上下文管理](#上下文管理)
6. [并行与自动化](#并行与自动化)
7. [常见错误模式](#常见错误模式)
8. [常用工作流食谱](#常用工作流食谱)

---

## 功能选型决策树

Claude Code 提供了多种扩展机制，很多用户分不清该用哪个。以下是决策指南：

### "我想让 Claude 在某个时机自动做某事" → **Hooks**

Hooks 是确定性的、保证执行的自动化。适合：
- 每次文件编辑后自动 lint/format
- 阻止危险命令（如 `rm -rf`）
- 会话开始时自动加载环境变量
- 提交前自动运行测试

**关键特征：** 不是"建议"而是"强制"。Claude 不能忽略 hook。

### "我想让 Claude 知道某些规则/偏好" → **CLAUDE.md**

CLAUDE.md 是建议性指令，每次会话都加载。适合：
- 代码风格偏好（缩进、命名规范）
- 项目特定的构建/测试命令
- 工作流约定（PR 格式、分支命名）
- Claude 无法从代码中推断的信息

**关键特征：** 总是加载，但 Claude 可能在长会话中"忘记"。保持 200 行以内。

### "我想让 Claude 按需获取某领域知识" → **Skills**

Skills 按需加载，不污染每次会话。适合：
- 特定领域的参考知识（API 规范、设计系统）
- 可复用的工作流（修 issue、创建 PR、代码审查）
- 需要附带脚本或模板的复杂任务

**关键特征：** 只在需要时加载。可以手动调用（`/skill-name`）或让 Claude 自动判断。

### "我想让 Claude 把复杂任务委托给专门助手" → **Subagents**

Subagents 在独立上下文中运行。适合：
- 不想污染主会话上下文的探索任务
- 需要特定模型或工具集的专业审查
- 并行处理多个独立子任务
- 需要隔离权限的操作

**关键特征：** 独立上下文 + 独立工具集。结果摘要返回主会话。

### "我想把多个功能打包分享" → **Plugins**

Plugins 将 skills + hooks + subagents + MCP 服务器打包。适合：
- 团队共享的工具链
- 社区分发的功能包

### 快速选型表

| 需求 | 推荐功能 | 理由 |
|------|---------|------|
| 每次编辑后自动格式化 | Hook (PostToolUse) | 需要确定性执行 |
| "测试用 pytest 不要用 unittest" | CLAUDE.md | 全局偏好，每次都需要 |
| REST API 设计规范文档 | Skill | 只在写 API 时需要 |
| "帮我审查安全漏洞" | Subagent | 需要独立上下文深度审查 |
| 阻止向 migrations/ 写入 | Hook (PreToolUse) | 需要强制阻止 |
| 项目特定的部署流程 | Skill (disable-model-invocation: true) | 手动触发的工作流 |
| 代码库探索不污染主会话 | Subagent (Explore) | 隔离上下文 |
| "提交时消息格式必须是 conventional commits" | Hook (PreToolUse on Bash) 或 CLAUDE.md | Hook 更可靠，CLAUDE.md 更简单 |

### ⚠️ 常见选型错误

| 错误做法 | 为什么不好 | 正确做法 |
|---------|----------|---------|
| 在 CLAUDE.md 写"每次编辑后运行 eslint" | CLAUDE.md 是建议性的，Claude 可能忘记 | 用 PostToolUse hook |
| 把 500 行 API 文档放进 CLAUDE.md | 每次会话都加载，浪费上下文 | 做成 skill，按需加载 |
| 用 skill 存放代码风格偏好 | 可能不会被自动加载 | 放进 CLAUDE.md |
| 让主会话读几百个文件做调研 | 填满上下文 | 委托给 subagent |

---

## 核心理念

Claude Code 是一个 agentic 编码环境——它不是聊天机器人，而是能读取文件、运行命令、修改代码并自主解决问题的 AI 代理。

**最重要的一条规则：** Claude 的上下文窗口是最需要管理的资源。上下文填满后性能会下降。你可以通过自定义 status line 来实时追踪上下文使用量（`/statusline`）。

---

## 高效工作流模式

### 1. 给 Claude 验证自身工作的方式（最高杠杆的做法）

Claude 在能自我验证时表现显著更好。提供测试、截图或预期输出，让 Claude 可以检查自己。

| 策略 | 不好的写法 | 好的写法 |
|------|----------|---------|
| 提供验证标准 | "实现一个验证邮箱的函数" | "写一个 validateEmail 函数，测试用例：user@example.com → true, invalid → false, user@.com → false。实现后运行测试" |
| 视觉验证 UI | "让仪表盘更好看" | "[贴截图] 按这个设计实现，完成后截图对比原图，列出差异并修复" |
| 定位根因 | "构建失败了" | "构建失败，报错信息：[贴错误]。修复并验证构建成功，定位根本原因而非压制错误" |

### 2. 先探索，再规划，再编码

使用 Plan Mode 把研究和执行分开：

**第一步：探索**（Plan Mode）
```
读取 /src/auth 目录，理解我们如何处理会话和登录
```

**第二步：规划**（Plan Mode）
```
我想添加 Google OAuth。需要改哪些文件？会话流程是什么？制定一个计划
```

按 `Ctrl+G` 在编辑器中直接编辑计划。

**第三步：实现**（Normal Mode）
```
按照你的计划实现 OAuth 流程。为回调处理器写测试，运行测试套件并修复失败
```

**第四步：提交**
```
用描述性的消息提交，并创建 PR
```

**何时跳过规划：** 如果你能用一句话描述 diff（修错别字、添加日志行、重命名变量），直接让 Claude 做就好。

### 3. 提供具体上下文

| 策略 | 不好的写法 | 好的写法 |
|------|----------|---------|
| 限定范围 | "给 foo.py 加测试" | "给 foo.py 写测试，覆盖用户未登录的边界情况，不要用 mock" |
| 指向来源 | "为什么 ExecutionFactory 的 API 这么奇怪" | "查看 ExecutionFactory 的 git 历史，总结它的 API 是怎么演变的" |
| 参考现有模式 | "添加日历组件" | "看看首页现有 widget 的实现模式，HotDogWidget.php 是好例子。按照这个模式实现日历组件" |
| 描述症状 | "修复登录 bug" | "用户报告会话超时后登录失败。检查 src/auth/ 的 auth 流程，特别是 token 刷新。写一个复现问题的失败测试，然后修复" |

### 4. 提供丰富的输入

- **用 `@` 引用文件**：`@src/utils/auth.js` 直接加载文件内容
- **贴图片**：直接拖拽或 Ctrl+V 粘贴
- **给 URL**：Claude 可以访问文档和 API 参考
- **管道输入**：`cat error.log | claude`
- **让 Claude 自己获取**：告诉它用 Bash、MCP 工具或读文件来拉取上下文

---

## Prompt 技巧

### 让 Claude 采访你

对于较大的功能，让 Claude 先采访你：

```
我想构建 [简要描述]。用 AskUserQuestion 工具详细采访我。

询问技术实现、UI/UX、边界情况、顾虑和权衡。不要问显而易见的问题，深入我可能没想到的难点。

一直采访到覆盖所有方面，然后写一份完整的 spec 到 SPEC.md。
```

完成后开新会话执行 spec——新会话有干净的上下文，专注于实现。

### 使用「ultrathink」关键词

在 Opus 4.6 和 Sonnet 4.6 上，在 prompt 中包含「ultrathink」会将该轮的推理深度设为 high，适合需要深度思考的一次性任务。

### 用 `/btw` 做侧边快速提问

```
/btw 这个函数的返回类型是什么？
```

答案以浮层展示，不进入会话历史，不消耗上下文。

---

## 上下文管理

> 具体的命令和快捷键见 `claude-code-basics.md` 的「交互模式快捷键」章节。这里只讲策略。

**核心观点：** 上下文窗口是 Claude Code 最稀缺的资源。你管理上下文的能力直接决定了 Claude 的输出质量。

### 三条黄金法则

1. **不相关任务之间必须 `/clear`** —— 混杂的上下文是 Claude "变笨"的头号原因
2. **探索任务交给 subagent** —— 让 subagent 读几百个文件，只返回摘要，主会话保持干净
3. **纠正两次仍不对就 `/clear` 重来** —— 失败尝试堆积在上下文中只会让情况更糟，用更好的初始 prompt 重新开始

### 实用技巧

- **`/btw` 做侧边提问**：答案以浮层展示，不进入上下文历史
- **`/compact` 主动压缩**：可以带指令，如 `/compact 保留所有修改过的文件路径`
- **在 CLAUDE.md 中写压缩规则**：`当压缩时，始终保留修改文件的完整列表和所有测试命令`
- **用 `/statusline` 追踪上下文用量**：实时可见，避免被动触发自动压缩

---

## 并行与自动化

### 非交互模式（脚本/CI）

```bash
# 一次性查询
claude -p "解释这个项目做什么"

# JSON 输出用于脚本解析
claude -p "列出所有 API 端点" --output-format json

# 流式处理
claude -p "分析这个日志文件" --output-format stream-json
```

### 多会话并行

三种方式：

1. **桌面应用**：管理多个本地会话，每个有独立 worktree
2. **Web（claude.ai/code）**：在 Anthropic 云基础设施上的隔离 VM 中运行
3. **Agent Teams**：多会话自动协调，共享任务和消息

### Writer/Reviewer 模式

| 会话 A（Writer） | 会话 B（Reviewer） |
|-----------------|------------------|
| "为 API 端点实现速率限制器" | |
| | "审查 @src/middleware/rateLimiter.ts 中的速率限制器实现，查找边界情况、竞态条件" |
| "这是审查反馈：[B 的输出]。处理这些问题" | |

### 批量处理（Fan-out）

```bash
# 生成任务列表后批量处理
for file in $(cat files.txt); do
  claude -p "将 $file 从 React 迁移到 Vue。返回 OK 或 FAIL。" \
    --allowedTools "Edit,Bash(git commit *)"
done
```

### 作为 Unix 工具使用

```bash
# 作为 linter
claude -p '你是一个 linter。检查 vs main 的变更，报告拼写错误相关的问题'

# 管道处理
cat build-error.txt | claude -p '简洁解释这个构建错误的根因' > output.txt

# 加入 package.json scripts
"lint:claude": "claude -p '检查 vs main 的变更，报告问题'"
```

---

## 常见错误模式

### 1. 厨房水槽会话
**症状**：一个会话中跳跃多个不相关任务，上下文充满无关信息。
**解决**：任务之间用 `/clear`。

### 2. 反复纠正
**症状**：Claude 做错了，你纠正，还是错，再纠正。上下文充满失败的尝试。
**解决**：纠正两次后，`/clear` 并用更好的初始 prompt 重新开始。

### 3. CLAUDE.md 过于详细
**症状**：CLAUDE.md 太长，Claude 忽略了其中一半的规则。
**解决**：无情精简。如果 Claude 不写这条规则也做得对，就删掉。或者转换为 hook。

### 4. 信任-验证的鸿沟
**症状**：Claude 产出看起来合理但未处理边界情况的实现。
**解决**：始终提供验证方式（测试、脚本、截图）。不能验证就不要上线。

### 5. 无限探索
**症状**：让 Claude "调查"某事但不限定范围，它读了几百个文件。
**解决**：限定调查范围，或使用 subagents。

---

## 常用工作流食谱

### 理解新代码库

```
给我一个这个代码库的概览
```
然后深入：
```
解释主要的架构模式
关键的数据模型是什么？
认证是如何处理的？
```

### 修复 Bug

```
我运行 npm test 时出现错误 [贴错误信息]
建议几种修复方式
应用你建议的修复，运行测试验证
```

### 重构代码

```
找到代码库中已弃用的 API 用法
建议如何重构 utils.js 使用现代 JavaScript 特性
重构，同时保持相同的行为
运行测试验证
```

### 写测试

```
找到 NotificationsService.swift 中没有测试覆盖的函数
为通知服务添加测试
添加边界条件的测试用例
运行新测试并修复失败
```

### 创建 PR

```
总结我对认证模块的更改
创建 PR
增强 PR 描述，添加更多关于安全改进的上下文
```

### 使用 Plan Mode

```bash
# 启动 Plan Mode
claude --permission-mode plan

# 或在 headless 模式使用
claude --permission-mode plan -p "分析认证系统并建议改进"
```

---

## 官方文档链接

- 最佳实践：https://code.claude.com/docs/en/best-practices
- 常用工作流：https://code.claude.com/docs/en/common-workflows
- 工作原理：https://code.claude.com/docs/en/how-claude-code-works
