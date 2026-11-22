---
type: architecture
updated: 2026-11-22
---

# Redis、Kafka 与流控设计

关联文档：[[../README|项目 README]] ｜ [[架构说明]] ｜ [[数据库设计]] ｜ [[../flows/easychat-ws-kafka-push-flow.html|WS 与 Kafka 推送时序图]]

## 一、Redis Key 空间与核心机制

EasyChat 在网关、安全组件、用户服务与聊天服务间共用 Redis，承担鉴权吊销、WebSocket 票据与在线注册表及频控职责（不写入明文密码，连接配置见 [[../deploy/部署说明]]）：

| Key 模式 | 数据结构 / 值 | TTL / 过期策略 | 读写方与核心用途 |
|---|---|---|---|
| `token:blacklist:{token}` | String `"1"` | 令牌剩余有效期（ms） | **写入**：`AuthServiceImpl`（`logout` 时拉黑 Access/Refresh Token；轮换刷新时拉黑旧 Refresh Token）<br>**校验**：`GatewayAuthFilter` & `JwtAuthFilter` |
| `token:revoke-before:{userId}` | String 毫秒时间戳 | `refreshExpiration`（默认 7 天） | **写入**：`TokenBlacklistServiceImpl.revokeUserTokens`（忘记密码、重置密码时触发）<br>**校验**：签发时间 `iat <= revokeBefore` 的历史 Access/Refresh Token 一律作废 |
| `ws:ticket:{ticket}` | String `{userId}` | `30s` | **写入**：`WsTicketService.issue`（`POST /api/chat/ws-ticket`）<br>**预检与核销**：`GatewayAuthFilter` 校验存在；Netty `AuthWsHandler` 调用 `getAndDelete` 一次性原子消费 |
| `ws:online:{userId}` | String `{wsInstanceId}` | `5m` | **写入与续期**：Netty 上线写入，收到客户端每 25s 一次 `PING` 执行 `REFRESH_SCRIPT` Lua CAS 续期<br>**断开清理**：`WsSessionRegistry` 执行 `OFFLINE_SCRIPT` Lua CAS 脚本（仅当 Redis 值等于本节点 `instanceId` 时续期/删除，避免旧连接断开误操作重连新节点会话） |
| `email:code:{email}` | String 6 位数字验证码 | `5m` | `EmailUtil` 发送邮箱验证码时写入，注册 / 找回 / 重置密码校验成功立即删除 |
| `email:code:limit:{email}` | String `"1"` | `60s` | `AuthServiceImpl` / `UserServiceImpl` 防止 60 秒内重复发送邮件验证码 |
| `rate:{methodName}:{identifier}` | String 原子计数器 | 注解参数 `seconds`（默认 60s） | `RateLimitAspect`（`ratelimit.enabled=true` 时生效），支持固定 key 或方法首字符串参数限流 |

### 断开在线状态 Lua CAS 脚本（`WsSessionRegistry.OFFLINE_SCRIPT`）

```lua
if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('del', KEYS[1])
else
  return 0
end
```

---

## 二、Kafka 多实例 WebSocket 推送设计

### 1. 投递策略开关（`easychat.ws.delivery`）

- **`local`（默认，无外部 MQ 依赖）**：`LocalWsPushRouter` 直接通过 `WsChannelWriter` 在当前 `easychat-chat` 进程的 `WsSessionManager` 查找目标用户的 Netty `Channel` 并下发。
- **`kafka`（多实例水平扩展，Docker 编排默认开启）**：
  - `KafkaWsPushRouter` 将消息/会话通知包装为 `WsPushEvent(toUserId, type, payloadJson, traceId)` 经 `KafkaWsPushPublisher` 发往主题 `easychat-ws-push`（以 `toUserId` 作为 Kafka 消息 Key）。
  - `WsDeliveryConfig` 为每个 `easychat-chat` 实例生成唯一 Bean `wsInstanceId` 与独立消费者组名：`${spring.application.name}-ws-push-${wsInstanceId}`（支持环境变量覆盖）。
  - 因此多台 `easychat-chat` 实例处于**不同 Consumer Group**，均可全量收到 `easychat-ws-push` 事件；各实例内 `WsPushKafkaConsumer` 检查本机 `WsSessionManager.getChannel(toUserId)` 是否存在，仅持有目标长连接的实例向前端写入 `TextWebSocketFrame`，实现无状态横向扩容与 `traceId` 跨线程透传。
- **在线状态异步广播调度（`WsOnlineStatusBroadcaster`）**：上线/下线触发的 `ONLINE_STATUS` 广播按 `userId` 哈希进入 `16` 路有界单线程条带池（每路队列 `64`），保障同一用户在 Kafka `toUserId` 分区上的上下线顺序一致，队满淘汰最旧任务时显式打印告警日志。

### 2. WebSocket 推送事件类型

前端 `ws.ts` 与后端服务已打通 11 类事件信封 `{ "type": "...", "payload": ... }`：

- 消息与会话：`NEW_MESSAGE`、`MESSAGE_UPDATE`（撤回/邀请卡片状态更新）、`CONVERSATION_UPDATE`、`NEW_CONVERSATION`、`CONVERSATION_REMOVED`
- 在线与连接保活：`ONLINE_STATUS`（上/下线异步通知会话好友/群友）、`PONG`
- 社交与群审批：`FRIEND_REQUEST`、`FRIEND_REQUEST_HANDLED`、`GROUP_JOIN_REQUEST`、`GROUP_JOIN_REQUEST_HANDLED`

---

## 三、Sentinel 网关与微服务限流基线

三个可执行模块均同时配置了本地 JSON 规则（兜底）与 Nacos `SENTINEL_GROUP` 动态数据源，限流触发时统一返回 JSON `{"code":429,"message":"请求过于频繁，请稍后再试"}`（其中 Gateway 在 `application.yml` 返回 HTTP `429`，`easychat-user` / `easychat-chat` 经 `SentinelWebMvcConfig` 返回 HTTP `200` + JSON `code:429`）：

| 模块 | 配置文件 / DataId | 已配置资源与阈值 |
|---|---|---|
| `easychat-gateway` | `sentinel/gw-flow.json` / `easychat-gateway-gw-flow` | 路由级限流：`easychat-user` QPS `200`、`easychat-chat` QPS `200`（`intervalSec: 1`） |
| `easychat-user` | `sentinel/flow.json` / `easychat-user-service-flow` | `/api/auth/login`（QPS `20`）、`/api/auth/register`（QPS `10`）、`/api/auth/send-verify-code`（QPS `5`） |
| `easychat-chat` | `sentinel/flow.json` / `easychat-chat-service-flow` | `/api/chat/message`（QPS `100`）、`/api/chat/upload`（QPS `20`） |
