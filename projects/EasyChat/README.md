---
type: project
status: 已完成
created: 2026-08-22
updated: 2026-11-22
tech:
  - Java 21
  - Spring Boot 4.0.7
  - Spring Cloud 2025.1.0
  - Spring Cloud Alibaba 2025.1.0.0
  - Apache Dubbo 3.3.5
  - Nacos 2.4.3
  - Alibaba Sentinel
  - Netty 4.2
  - Kafka / Redpanda
  - MyBatis-Plus 3.5.16
  - Flyway
  - Redis 7.4
  - MySQL 8.4
  - Vue 3.5
  - TypeScript
  - Tailwind CSS 4
  - Pinia 4
  - Electron 35
repo: D:\workspace\EasyChat
remote: https://github.com/www12354jkl/EasyChat
---

# EasyChat

## 一句话简介

基于 Spring Boot 4、Spring Cloud Alibaba、Netty WebSocket、Kafka、Apache Dubbo 与 Vue 3 / Electron 构建的微服务分布式即时通讯系统（Web + 桌面双端）。

## 解决什么问题

- **统一网关鉴权与长连接动态路由**：`easychat-gateway`（WebFlux `:8080`）统一校验 Bearer JWT、`token:blacklist`、`token:revoke-before` 与一次性 `ws:ticket`，通过 `WsLoadBalancer` 读取 Nacos 实例元数据 `ws-port` 将 `/ws` 升级流量路由至 `easychat-chat` 的 Netty `:8083` 端口。
- **可水平扩展的多实例 WebSocket 实时推送**：`easychat-chat` 支持 `local`（单机直写 Channel）与 `kafka`（全实例独立消费组广播，本地命中活跃 Channel 后下发）双模式投递，并结合 Redis `ws:online` 心跳续期与 Lua CAS 断开清理维持在线状态一致性。
- **跨服务拆分、流量治理与消息防重**：`easychat_user` 与 `easychat_chat` 物理分库（各自内置 Flyway 增量迁移脚本 `V1~V7`），通过 Dubbo 3.3 RPC（配 Failfast 降级）协同；同时集成 Sentinel 网关/接口限流、Redis 注解 `@RateLimit` 防刷、`client_msg_id` 唯一索引发信幂等、2 分钟限时撤回与成员维度 `cleared_at` 单边清空记录。
- **Web 与 Electron 桌面双端体验**：前端一套 Vue 3 + Tailwind CSS + Pinia 代码同步支持浏览器 Web 与 Electron 35 桌面端（无边框窗口、登录/主窗体尺寸模式自动切换、`contextMenu`、IndexedDB 会话本地缓存与 401 单飞刷新）。

## 技术栈

- **后端基础**：Java 21、Spring Boot 4.0.7、Spring Cloud 2025.1.0、Spring Cloud Alibaba 2025.1.0.0（Nacos 服务发现 / Sentinel 流控）、SpringDoc OpenAPI 3.0.3
- **RPC 与实时通信**：Apache Dubbo 3.3.5（Interface 级注册）、Netty 4.2.16（WebSocket `:8083`）、Spring Kafka / Redpanda
- **数据与缓存**：MySQL 8.4（`easychat_user` + `easychat_chat` 双库）、MyBatis-Plus 3.5.16 + Druid 1.2.28 + Flyway、Redis 7.4
- **安全与外部服务**：JJWT 0.13.0、腾讯云 COS SDK（头像与聊天文件）、QQ SMTP 邮箱验证码
- **前端与桌面端**：Vue 3.5.40、Vite 8.1.5、TypeScript、Pinia 4.0.2、Vue Router 5.2.0、Tailwind CSS 4.3.3、Electron 35.7.5 + `electron-builder` 26.0.12

## 快速运行

> [!TIP] 项目状态：已完成
> 后端四模块与前端 Web / Electron 桌面端均已完整实现（仓库含 Docker 编排、Windows 批处理脚本、GitHub Actions CI 及历史端到端验证截图）。

1. **Docker 全栈一键启动**：在 `D:\workspace\EasyChat\deploy` 按 `.env.example` 配置环境变量后执行 `docker compose up -d --build`，拉起 MySQL、Redis、Nacos、Kafka 及三个微服务容器。
2. **本地 IDE / Windows 脚本运行**：确认 MySQL(`3306`)、Redis(`6379`)、Nacos(`8848`)、Kafka(`9092`) 就绪后，在 `D:\workspace\EasyChat\backend` 运行 `start-all.bat` 依次启动 `easychat-user`（`:8081`）、`easychat-chat`（HTTP `:8082` / Netty WS `:8083`）与 `easychat-gateway`（`:8080`）。
3. **启动前端或桌面端**：在 `D:\workspace\EasyChat\frontend\easychat` 安装依赖后运行 `npm run dev`（Web 端，代理 `/api` 与 `/ws` 至 `:8080`）或 `npm run electron:dev`（Electron 桌面端）。详情见 [[deploy/部署说明]]。

## 界面实测截图

![[assets/easychat-chat-window.png]]

![[assets/easychat-login.png]]

## 相关文档

- [[思维导图]]
- 架构：[[architecture/架构说明]]（[[architecture/html/easychat-architecture.html|交互式系统架构图]]）
- 数据库：[[architecture/数据库设计]]（[[architecture/html/easychat-erd.html|数据库 E-R 图]]）
- 缓存、队列与流控：[[architecture/Redis与Kafka设计]]
- 流程图：[[flows/easychat-ws-kafka-push-flow.html|WebSocket 票据握手与 Kafka 多节点推送时序图]]
- 部署：[[deploy/部署说明]]
- 决策记录：[[decisions/001-内部通信从OpenFeign迁移至Dubbo接口级注册|001 Dubbo 接口级注册]] ｜ [[decisions/002-多实例WebSocket广播引入Kafka独立消费组与Redis-CAS在线态|002 Kafka WS 广播与 CAS]] ｜ [[decisions/003-WebSocket握手改用30秒一次性Ticket与统一鉴权契约|003 WS Ticket 握手契约]] ｜ [[decisions/004-双Token黑名单迁移非阻塞网关校验并采用Redis故障Fail-Closed策略|004 双 Token 黑名单与 Fail-Closed]] ｜ [[decisions/005-在线状态异步广播采用按userId分片单线程条带池|005 在线状态广播条带池]]
- 开发日志：[[devlog/2026-11-20|2026-11-20 开发日志]] ｜ [[devlog/2026-11-22|2026-11-22 开发日志]]
