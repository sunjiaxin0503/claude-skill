# 活动玩法配置参考

本文档包含 event 服务中所有玩法类型的配置结构，供生成需求文档时作为配置示例的参考。

## 目录
- [通用玩法框架](#通用玩法框架)
- [奖励配置](#奖励配置)
- [道具类型](#道具类型)
- [各玩法配置详情](#各玩法配置详情)

---

## 通用玩法框架

每个玩法（Feature）在系统中的通用属性：

| 字段 | 类型 | 说明 |
|-----|------|------|
| type | enum | 玩法类型标识 |
| cycle | enum | 参与周期：1-无周期 2-每日 3-每周 4-每月 |
| cycle_limit | int | 单周期内最大参与次数，0 为无限制 |
| limit | int | 用户总参与次数上限，0 为无限制 |
| engage_account | enum | 参与维度：1-用户ID 2-角色ID |
| since / until | datetime | 玩法起止时间 |
| config | json | 玩法专属配置（按类型不同结构不同） |
| rewards | json | 参与奖励配置 |

**周期刷新说明：**
- 周刷新：每周一 00:00:00 ~ 每周日 23:59:59
- 月刷新：每月 1 日 00:00:00 ~ 当月最后一天 23:59:59
- 配置周期刷新时，活动起止时间应覆盖完整周期

---

## 奖励配置

### 每次参与都发放（WHENEVER）
```json
{
  "type": "ENGAGEMENT_REWARD_TYPE_WHENEVER",
  "whenever": {
    "items": [
      { "item_id": 1, "item_amount": 2 }
    ]
  }
}
```

### 达到指定参与次数发放（REGULAR）
```json
{
  "type": "ENGAGEMENT_REWARD_TYPE_REGULAR",
  "regular": {
    "details": [
      {
        "engagement_count": 1,
        "items": [{ "item_id": 1, "item_amount": 2 }]
      },
      {
        "engagement_count": 3,
        "items": [{ "item_id": 2, "item_amount": 1 }]
      }
    ]
  }
}
```

---

## 道具类型

| 类型值 | 标识 | 说明 | 特殊处理 |
|-------|------|------|---------|
| 1 | EVENT_ITEM | 活动道具 | 活动内产出消耗，存入 user_items 表 |
| 2 | GAME_ITEM | 游戏道具 | 通过 GM 命令发放到游戏内，需配置 gm_reward_type 和 gm_reward_params |
| 3 | PHYSICAL_ITEM | 实物 | 需收集收货地址 |
| 4 | WEIXIN_HONGBAO | 微信红包 | 需用户绑定微信 OpenID，通过 payment 服务发放 |
| 5 | GIFT_CODE | 礼包码 | 支持唯一码（UNIQUE）和通兑码（UNIVERSAL） |
| 6 | LOTTERY_TICKET | 抽奖券 | 用于开奖玩法 |
| 127 | VOID_ITEM | 空道具 | "谢谢参与"场景 |

---

## 各玩法配置详情

### 预约（preregister）

用户点击预约按钮即视为参与成功。可能携带邀请者信息。

```json
{
  "fake_ratio": 0.1,  // 前端展示预约数的放大系数
  "fake_base": 1000   // 前端展示预约数的假定基数
}
```

需求文档需确认：预约数展示的放大策略。

---

### 抽奖（lottery）

用户消耗道具进行随机抽奖。

```json
{
  "consume_item_id": 2,      // 消耗的道具 ID
  "consume_item_amount": 1,  // 每次消耗数量
  "lottery_items": [
    {
      "item_id": 8,
      "item_amount": 1,
      "random_weight": 1000,     // 随机权重
      "guaranteed_count": 0,     // 保底次数，0 为无保底
      "item_stock": 1000,        // 库存上限，0 为不限
      "daily_limit": 0,          // 每日限量，0 为不限
      "per_user_limit": 0,       // 每人限量，0 为不限
      "total_min_limit": 0,      // 总用户最低抽取次数
      "per_user_min_limit": 0    // 单用户最低抽取次数
    }
  ]
}
```

需求文档需确认：消耗道具、奖池内容、各奖品权重和限量策略。

---

### 问卷（survey）

用户完成第三方问卷平台（如问卷星）的问卷。

```json
{
  "platform_name": "wjx",         // 平台名称
  "survey_id": "12345",           // 问卷 ID
  "survey_url": "https://...",    // 问卷地址
  "receive_method": "active"      // 发奖方式：game-通知游戏 / active-活动自行发放
}
```

需求文档需确认：问卷平台、发奖方式。

---

### 邀请（invite）

分享邀请链接给好友，好友通过链接完成登录即算邀请成功。

```json
{
  "share_url": "https://..."  // 分享的邀请链接
}
```

需求文档需确认：邀请达标人数、邀请奖励阶梯。

---

### 分享（share）

用户分享链接或图片到社交平台。

```json
{
  "share_platform": "wechat",     // 分享平台
  "jump_url": "https://...",      // 分享跳转地址
  "icon_url": "https://..."       // 分享图标
}
```

---

### 关注社媒（follow）

用户点击关注官方社交媒体账号。

```json
{
  "platform_name": "bilibili",    // 平台名称
  "platform_icon": "https://...", // 平台图标
  "platform_url": "https://...",  // 平台地址
  "qr_code_url": "https://...",   // 二维码链接
  "platform_desc": "关注B站官方号" // 平台描述
}
```

---

### 微信订阅（weixin_subscribe）

用户订阅微信消息通知。

```json
{
  "weixin_appid": "wx...",
  "weixin_template_ids": ["template_id_1", "template_id_2"]
}
```

---

### 评论/弹幕（comment）

用户发布评论或弹幕，经易盾敏感词过滤后存储。

```json
{
  "preset_comments": ["预设弹幕1", "预设弹幕2"],  // 预设评论
  "send_rate": 10  // 弹幕发送速率
}
```

---

### 投票（vote）

用户为预设选项投票。支持官方上传、用户投稿、投票入围三种选项来源。

```json
{
  "vote_feature_id": [1, 2],          // 关联的投票玩法 ID
  "submission_feature_id": [3],       // 关联的投稿玩法 ID
  "finalists_amount": 5,             // 入围数量
  "consume_item_id": 17,             // 消耗道具 ID（可选）
  "consume_item_amount": 100,        // 消耗数量
  "source": "system",                // 选项来源：system/submission/finalists
  "options": [
    {
      "sn": "option_001",
      "name": "选项名称",
      "imgs": ["https://..."],
      "videos": [],
      "describes": ["描述"],
      "jump_url": "",
      "fake_base": 0,
      "fake_ratio": 1
    }
  ],
  "rewards": [
    { "item_id": 1, "item_amount": 1 }  // 入围奖励
  ]
}
```

---

### 组队（team）

玩家组队共同完成任务进度。

```json
{
  "max_members": 5,                // 最大队员数
  "min_members": 2,                // 最小队员数
  "teams_per_recommendation": 10   // 推荐队伍数量
}
```

---

### 任务（quest）

一系列系统设定的目标任务，支持个人和组队模式。

**任务目标类型（objective）：**

| 枚举值 | 说明 | 数据来源 |
|-------|------|---------|
| player_active_points | 累计游戏内活跃值 | gamer/data |
| player_login_days | 累计登录人天 | gamer/data |
| order_total_amount | 累计充值金额 | combo/order |
| order_total_count | 累计充值次数 | combo/order |
| order_in_app_total_amount | 累计游戏内充值金额 | combo/order |
| order_seayoo_web_total_amount | 累计支付中心充值金额 | combo/order |
| community_posts | 社区发帖数量 | gamer/community |
| community_comments | 社区评论数量 | gamer/community |
| community_post_likes | 社区帖子点赞数量 | gamer/community |
| player_match_counts | 游戏对局次数 | gamer/data |
| player_match_days | 游戏对局天数 | gamer/data |
| player_level | 账号等级目标 | gamer/data |
| event_items | 活动道具数量 | gamer/event |
| game_task | 游戏内任务 | gamer/data |
| team_size | 组队进度 | gamer/event |

**个人任务示例：**
```json
{
  "objective": "player_active_points",
  "completion_value": 200,
  "enable_time_range": true,
  "since": 1736996056,
  "until": 1736996056
}
```

**组队任务示例（sum 算法）：**
```json
{
  "objective": "player_login_days",
  "completion_value": 3,
  "team": {
    "feature_id": 73,
    "completion_value": 10,
    "progress_algorithm": "sum"
  },
  "enable_time_range": false
}
```

**组队任务示例（top_n 算法）：**
```json
{
  "objective": "player_level",
  "completion_value": 0,
  "team": {
    "feature_id": 73,
    "completion_value": 10,
    "progress_algorithm": "top_n",
    "top_n": 2
  }
}
```

**特殊任务目标配置：**

- `community_posts` / `community_post_likes`：可指定 `topic_id`
- `player_match_counts`：可配置 `required_players`（要求玩家数量）和 `required_host`（是否要求房主）
- `player_match_days`：可配置 `required_players`、`required_matches`（每日完成次数）和 `required_host`
- `event_items`：需配置 `event_item_id`（活动道具 ID）
- `game_task`：需配置 `task_id` 和 `progress_algorithm`

需求文档需确认：任务目标类型、完成值、是否组队、周期刷新策略。

---

### 开奖（lottery_draw）

用户使用抽奖券进行开奖，随机获得奖励。

```json
{
  "ticket_item_id": 1,            // 抽奖券道具 ID
  "draw_not_before": 1736996056,  // 最早开奖时间
  "draw_not_after": 1736996056,   // 最晚开奖时间
  "lottery_items": [
    {
      "item_id": 1,
      "item_count": 1,
      "weight": 100,
      "stock": 50
    }
  ]
}
```

---

### 礼包码（gift_code）

用户领取可在游戏内兑换道具的 CDK。

```json
{
  "item_id": 1,                       // 礼包码道具 ID
  "mp_url": "https://...",            // 社媒平台链接
  "mp_qrcode_url": "https://..."     // 社媒二维码图片链接
}
```

---

### 充值返还（cashback）

根据用户充值金额按规则返还奖励。

```json
{
  "order_start_time": "1727712000",
  "order_end_time": "1730390399",
  "claim_rewards_start_time": "1730390400",
  "claim_rewards_end_time": "1732896000",
  "game_item_id": 1,
  "game_item_exchange_rate": 10,
  "cashback_rules": [
    {
      "amount_left_open_bound": 0,
      "amount_right_closed_bound": 100000,
      "cashback_percent": 300
    }
  ]
}
```

---

## 活动级配置（EventConfig）

```json
{
  "budget_id": "预算 ID（仅微信红包奖励需要配置）",
  "email_title": "邮件标题",
  "email_content": "邮件内容",
  "sender": "发件人",
  "email_ttl": 604800   // 邮件有效期（秒），7天 = 604800
}
```

- budget_id：仅在活动涉及发放「微信红包」奖励时需要配置，其他奖励类型不需要
- email_title / email_content / sender / email_ttl：涉及游戏道具通过 GM 命令发放时需要配置
