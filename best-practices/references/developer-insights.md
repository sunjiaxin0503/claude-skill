# Claude Code 开发者洞察

> **来源说明：** 本文档整理自 Claude Code 核心开发者在 Twitter 上分享的实战经验和设计思考。这些内容代表了构建者视角的深度洞察，是官方文档之外的重要补充。
>
> **时效性说明（2026-03）：** 这些开发者持续在 Twitter 上分享新的见解。如果用户提到最新的开发者建议或你在参考文档中找不到相关信息，建议搜索以下 Twitter 账号获取最新内容：
> - @trq212（Thariq）— "Lessons from Building Claude Code" 系列
> - @bcherny（Boris Cherny）— Claude Code 创作者的使用技巧
> - @RLanceMartin（Lance Martin）— Agent 设计和工作流

## 目录
1. [生产力最高杠杆：Worktree 并行](#worktree-并行)
2. [验证驱动开发的内部实践](#验证驱动开发)
3. [CLAUDE.md 的迭代哲学](#claudemd-迭代)
4. [Skill 设计的内部经验](#skill-设计)
5. [Plan Mode 的高级用法](#plan-mode-高级用法)
6. [被低估的日常技巧](#被低估的日常技巧)
7. [Agent 设计思维](#agent-设计思维)

---

## Worktree 并行

**来源：Boris Cherny (@bcherny) — Claude Code 团队内部 #1 生产力技巧**

Claude Code 团队内部认为**同时开 3-5 个 git worktree** 是"单一最大的生产力提升"。

### 具体做法

```bash
# 启动隔离的 worktree 会话
claude --worktree feature-auth
claude --worktree feature-api
claude --worktree analysis
```

### 团队内部模式

- 有人设置 shell alias（`za`, `zb`, `zc`）一键在 worktree 之间切换
- 有人保留一个专门的 "analysis" worktree，只用来读日志和跑查询，从不写代码
- Claude Desktop 桌面应用原生支持多 worktree 管理

### 为什么比"多个终端窗口"更好

每个 worktree 有独立的文件系统状态，Claude 的编辑不会相互干扰。这不只是并行——是**隔离**。

---

## 验证驱动开发

**来源：Boris Cherny (@bcherny) — "最重要的一条建议"**

> "给 Claude 一种验证自身工作的方式。如果 Claude 有了这个反馈回路，最终结果的质量会提升 2-3 倍。"

### 团队是怎么做的

- Claude 测试 Boris 提交到 Claude Code 仓库的**每一个**变更
- 验证可以很简单：运行 bash 命令、跑测试套件、在浏览器或手机模拟器中测试
- 投资让你的验证流程**坚不可摧**——这是最高回报的投入

### 实战示例

```
不好："实现邮箱验证功能"
好："写一个 validateEmail 函数。测试用例：user@example.com → true, invalid → false, user@.com → false。实现后运行测试。"
```

---

## CLAUDE.md 迭代

**来源：Boris Cherny (@bcherny) — 团队内部实践**

> "团队无情地编辑 CLAUDE.md，持续迭代，直到 Claude 的出错率可测量地下降。"

### 迭代循环

1. 观察 Claude 的错误
2. 在 CLAUDE.md 中添加规则
3. 观察是否生效
4. 如果 Claude 不遵守 → 规则可能措辞不清或文件太长
5. 如果 Claude 本来就做对了 → 删掉这条规则

### 核心原则

- CLAUDE.md 应该像代码一样维护——定期 review、定期 prune
- 加入 git 让团队共同受益和共同维护
- 如果一条规则是**必须**执行的（不能容忍遗忘），转成 Hook 而不是留在 CLAUDE.md

---

## Skill 设计

**来源：Thariq (@trq212) — "Lessons from Building Claude Code: How We Use Skills"**

### Anthropic 内部的 Skill 分类

| 类别 | 说明 | 示例 |
|------|------|------|
| **Library/API 参考** | 教 Claude 正确使用某个库或 CLI | SDK 用法 + 常见错误列表 |
| **产品验证** | 确保 Claude 的输出正确 | 截图对比、输出校验脚本 |
| **代码质量审查** | 执行组织的代码质量标准 | 确定性脚本 + Hook 或 GitHub Action |

### 关键洞察：Gotchas 是最有价值的

> "任何 skill 中最高信号的内容是 Gotchas 章节——记录 Claude 使用这个 skill 时常犯的错误，理想情况下持续更新。"

**实践建议：** 每个 skill 都应该有一个 Gotchas 区域，从真实使用中积累，而不是凭想象编写。

### 渐进式披露（Progressive Disclosure）

> "把整个文件系统看作上下文工程和渐进式披露的一种形式——告诉 Claude 有哪些文件存在，它会在合适的时机读取。"

**做法：** 在 SKILL.md 中列出引用文件的目录和简要说明，Claude 会根据需要自己去读详细内容，而不是把所有内容塞进一个文件。

### 验证 Skill 值得重点投入

> "验证类 skill 极其有用，值得让一个工程师花一整周专门把验证 skill 做到极致。"

如果你的团队在某个验证流程上反复出错（截图对比、性能测试、兼容性检查），做成 skill 是高杠杆投入。

---

## Plan Mode 高级用法

**来源：Boris Cherny (@bcherny) — 团队内部做法**

### 双 Claude 审查模式

一位团队成员的工作流：
1. 让 Claude A 在 Plan Mode 下制定实施计划
2. 开一个新会话 Claude B，以"资深工程师"身份审查 Claude A 的计划
3. 将 Claude B 的反馈喂回 Claude A 修改计划
4. 计划通过后再进入实施阶段

### Plan 自动清除上下文

接受计划后，Claude Code 会自动清除上下文，让实施阶段从干净的上下文开始。这显著提升了计划的遵循率。如果不想清除，可以在设置中关闭。

### Plan Mode 纪律

**方向错了就重新规划，不要硬推。** 很多人在执行过程中发现问题后选择继续修修补补，但更好的做法是停下来、回到 Plan Mode、重新制定计划。

---

## 被低估的日常技巧

**来源：Boris Cherny (@bcherny) 和团队**

### 语音输入 Prompt

macOS 上按两次 fn 键激活语音听写。你说话的速度大约是打字的 3 倍。对于描述性的长 prompt 特别有用。

### "每天做超过一次的事就变成 Skill"

团队成员举例：
- `/techdebt` — 快速识别和标记技术债
- 各种 context dump 命令
- 项目特定的部署和测试流程

### /insights 命令

运行 `/insights`，Claude 会读取你过去一个月的消息历史，总结你的项目、使用模式，并给出工作流优化建议。这个命令在了解自己的使用模式方面非常有帮助。

### 输出风格设置

在 `/config` 中启用 "Explanatory" 或 "Learning" 输出风格，让 Claude 解释它的改动背后的**为什么**。对于学习新代码库或不熟悉的技术栈特别有用。

### Subagent 用于计算隔离

不要只把 subagent 看作"并行处理工具"。更重要的用途是**计算隔离**——让 subagent 做大量读取和分析，只返回摘要，保持主 agent 的上下文干净。

---

## Agent 设计思维

**来源：Thariq (@trq212) 和 Lance Martin (@RLanceMartin)**

### 工具设计是一门艺术

> "构建 agent 工具链中最难的部分是设计其行动空间。为模型设计工具既是艺术也是科学，高度取决于你使用的模型、agent 的目标和它运行的环境。你应该经常实验、阅读输出、尝试新东西。"

### 代码编排 vs 工具调用

Lance Martin 提出的观点：与其让 Claude 逐个调用工具（每次都回传到上下文），不如**给 Claude 一台电脑，让它用代码编排工具**。这就是 Programmatic Tool Calling（PTC）的思路，在 Opus 4.6 和 Sonnet 4.6 中支持。

### 知识型 Skill 的设计原则

> "如果你发布的 skill 主要是关于知识的，尽量聚焦于那些能**推动 Claude 跳出常规思维方式**的信息。"

Claude 已经知道的东西不需要重复。Skill 应该补充 Claude 的盲区，而不是重复它的知识。

---

## 关键 Twitter 搜索关键词

当需要搜索最新的开发者洞察时，使用以下搜索词：
- `from:trq212 "Claude Code"` — Thariq 的最新分享
- `from:bcherny "Claude Code"` — Boris 的最新技巧
- `from:RLanceMartin Claude agent` — Lance 的 agent 设计见解
- `from:trq212 "Lessons from Building"` — Thariq 的系列文章
