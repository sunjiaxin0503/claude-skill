# 世游通行证（Account）- 系统架构

> 来源：世游通行证 - 系统架构.docx

## 架构概述

世游通行证整体涉及以下几部分（按依赖关系自底向上）：
- 第三方服务
- 中间件
- 后端服务
- 通行证网站
- 客户端 SDK

## 第三方服务

| 服务 | 说明 |
|-----|------|
| 网络游戏防沉迷实名认证系统 | 中宣部系统，合规要求必接 |
| 本机号码一键登录 | 如创蓝云智的闪验认证 |
| 短信验证码 | 如腾讯云短信 SMS；需一主一备两家供应商 |
| 实名认证 | 如腾讯云人脸核身；用于防沉迷实名认证的补充 |
| 人机验证 | 如极验（GEETEST）行为验证 4.0；对抗黑产恶意流量 |

## 中间件

| 中间件 | 版本/类型 | 云服务 | 说明 |
|-------|---------|-------|------|
| MySQL | 8.0 | Aurora MySQL v3 (Serverless v2) | 持久化数据，读写分离，至少一主一从 |
| Redis | 7.0 | ElastiCache for Redis (Cluster Mode) | 缓存、session、登录 token |
| SQS | FIFO Queues | Amazon SQS | 系统内部异步处理 |

**数据安全要点**：
- Aurora MySQL 强制开启 TLS（require_secure_transport = ON）
- 不使用 MySQL 内置加解密功能
- 总是使用 client-side envelope encryption
- Encryption key 为服务级别，使用 AWS KMS 管理
- 手机号、真实姓名、身份证号为 PII 数据，必须加密存储

## 后端服务

所有后端服务均用 Golang 实现，无状态。对内服务采用 gRPC，对外服务采用 HTTP + JSON。

### 对内服务

| 服务 | 职责 |
|-----|------|
| **account-core** | 通行证核心功能，通行证数据的 owner。只有此服务能读写通行证帐号数据和访问明文敏感数据。其他服务只能获取掩码后的敏感数据 |
| **account-person** | 对接第三方实名认证服务；维护公司统一实名信息数据库；对接中宣部防沉迷系统(wlc)。通过消息队列合并用户上下线行为数据后批量上报 |
| **account-otp** | 发送和验证 One-Time Password (OTP)，对接多个短信服务提供商 |

### 对外服务

| 服务 | 职责 | 部署地址示例 |
|-----|------|-----------|
| **account-game-api** | 面向游戏客户端的注册、登录 API | https://account-game-api.seayoo.com/ |
| **account-web-api** | 面向通行证网站 Web 前端的 API | 通行证网站域名下 |

两个对外服务均需支持请求限流，其他后端服务不应调用这两个服务的 API。

## 通行证网站

- 前后端分离，流量入口为 Amazon CloudFront
- 部署地址示例：https://account.seayoo.com
- 动态流量：CloudFront → ALB → traefik → account-web-api（不做缓存，仅动态加速）
- 静态资源：CloudFront → S3（常规缓存策略）
- 同域名访问 API，避免 CORS

### OIDC Provider

通行证网站作为 Identity Provider (IdP)，通过 OpenID Connect (OIDC) 协议为世游其他面向玩家的 Web 系统提供用户身份认证。潜在 OIDC Client 包括：游戏官网、预约站、运营活动、交易平台。

## 客户端 SDK

包括 iOS、Android、Unity 三个平台的通行证 SDK。通行证 SDK 被 Combo SDK 聚合，游戏客户端对接 Combo SDK 而非直接对接通行证 SDK。通行证 SDK 仅与 account-game-api 交互。
