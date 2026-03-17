# game-event-requirement

A Claude skill for writing game operation event requirement documents（游戏运营活动需求文档编写技能）。

---

## 简介

`game-event-requirement` 是一个专为游戏运营团队设计的 Claude Skill。它通过**分阶段交互式提问**，引导运营策划、产品经理、研发或测试人员，逐步完成一份完整的游戏运营活动需求文档，输出为标准 Markdown 格式。

### 适用场景

- 撰写新版本预热活动、节日活动、拉新促活等运营活动的需求文档
- 需要快速整理活动玩法（预约、抽奖、签到、任务、投稿、组队等）及奖励发放逻辑
- 涉及世游（Seayou）/ Gamer / event 服务 / SDK 等内部系统的活动配置

### 触发关键词

以下关键词会自动触发本 Skill：

- 活动需求、活动策划文档、运营活动需求文档
- 活动方案、写一个活动需求、帮我写活动、新活动需求
- 帮我出个活动方案、策划一下 xx 活动
- 世游、Gamer、event 服务、活动玩法配置

---

## 功能特性

- **5 阶段交互式信息收集**：基础信息 → 平台与用户范围 → 活动玩法设计 → 道具与奖励 → 技术与补充说明
- **自动识别玩法类型**：根据用户的自然语言描述，自动匹配系统玩法（预约 / 抽奖 / 签到 / 任务 / 邀请 / 分享等 16+ 种）
- **联运用户处理**：自动检测是否需要 Combo ID 关联流程
- **配置项完整性检查**：对每个玩法逐一确认通用配置和专属配置
- **奖励联动校验**：自动检查道具来源与消耗的完整性，避免遗漏
- **输出标准文档**：最终输出结构完整、可直接用于评审的 Markdown 需求文档

---

## 安装方法

### 前提条件

- 已安装 [Claude 桌面版（Cowork）](https://claude.ai/) 或 Claude Code CLI
- 具备导入自定义 Skill 的权限

### 步骤

1. **下载或克隆本仓库**：

   ```bash
   git clone https://github.com/piacere0503/claude-skill.git
   ```

2. **将 `game-event-requirement` 文件夹放入你的 Skills 目录**：

   - Cowork 桌面版：在 Cowork 界面中选择「导入 Skill 文件夹」，选择 `game-event-requirement` 目录即可。
   - Claude Code CLI：将文件夹放入 `.claude/skills/` 目录下，或在 Settings 中指定 Skills 路径。

3. **重启 Claude** 使 Skill 生效。

4. **验证安装**：在对话框中输入「帮我写一个活动需求」，如果 Claude 开始分阶段提问活动信息，即安装成功。

---

## 文件结构

```
game-event-requirement/
├── SKILL.md                        # Skill 主配置与工作流程说明
├── README.md                       # 本文件
└── references/
    └── feature_configs.md          # 各玩法详细配置项参考
```

---

## 输出文档结构

生成的需求文档包含以下章节：

1. 活动概述（名称、背景、目标、时间、负责人）
2. 平台与用户范围（登录方式、参与用户类型、Combo ID 关联流程）
3. 活动玩法设计（每个玩法的配置详情）
4. 道具与奖励设计（道具类型、发放规则、奖励明细）
5. 技术说明与特殊逻辑（数据埋点、边界情况处理等）

---

## License

MIT
