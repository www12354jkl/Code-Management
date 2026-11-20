---
type: decision
date: 2026-09-06
status: 已采纳
---

# 002 - 多实例 WebSocket 推送引入 Kafka 独立消费组与 Redis Lua CAS 在线态

## 背景：遇到了什么问题
`easychat-chat` 通过 Netty（`:8083`）在进程内 `ConcurrentHashMap<String, Channel>` 中维护用户长连接。当服务水平扩容为多实例时，用户 A 连在实例 1、用户 B 连在实例 2，REST 发信请求落在实例 1 时本地找不到 B 的 `Channel`，消息无法跨实例实时下发。同时，若只用简单的 Redis `DEL ws:online:{userId}` 处理断线，网络抖动导致旧连接晚于新连接关闭时，旧连接会误删新建立的在线状态甚至触发错误的下线广播。

## 备选方案
1. **方案 A：Redis 记录 `userId -> host:wsPort`，发送方按路由定向直连目标实例投递**
   - 优点：精确单播，减少无效消息投递。
   - 缺点：需要维护实例间点对点通信通道，实例动态扩缩容或网络变更时重试、缓冲和实例不可达兜底复杂度高。
2. **方案 B：抽象 `WsPushRouter`（`local` / `kafka` 双模式）+ Kafka 每实例独立 Consumer Group 广播 + Redis Lua CAS**
   - 优点：单机开发默认 `local`（零 MQ 依赖）；生产多实例开启 `kafka` 后每个实例以 `${spring.application.name}-ws-push-${wsInstanceId}` 作为独立消费组全量接收事件，各自判断本机是否持有 `Channel` 再写出并透传 `traceId`；在线状态键 `ws:online:{userId}` 的续期与清理通过 Lua CAS 校对 `instanceId` 实现原子操作。
   - 缺点：多实例规模极大时，每条推送事件会在各 `chat` 实例产生一次轻量本地查找过滤开销。

## 最终选择与理由
选择**方案 B**。以当前微服务规模而言，`WsSessionManager.getChannel(toUserId)` 的内存查找成本极低，但换来了网关到 Netty 节点完全无状态转发、免维护实例间内部 RPC 网格、以及单机/集群无缝切换的巨大运维优势。在此基础上于 `2026-11-20` 进一步补齐：在 `WsSessionManager` 内部按 `userId` 增加 64 分段锁串行化 `register` 与 `unregister`，锁外关闭被顶替的旧 `Channel`，且仅当 `channels.remove(userId, channel)` 返回 `true` 时才执行 `markOffline` 与下线广播，彻底堵住同实例快速重连时的状态竞态；并将 `WsSessionRegistry.refresh` 同样改为 Lua CAS 原子续期。

## 影响与后续注意点
- 相关设计见 [[../architecture/Redis与Kafka设计|Redis 与 Kafka 设计]] 及 [[../flows/easychat-ws-kafka-push-flow.html|WS 与 Kafka 推送时序图]]。
- 多实例部署 `easychat-chat` 时必须保持 `WS_DELIVERY=kafka` 且每个实例拥有不同的 `wsInstanceId`（未显式配置时由 `WsDeliveryConfig` 生成 UUID），否则多个实例会落入同一消费组导致漏推。
- 在容器化环境启动 Kafka/Redpanda 时须确保 `KAFKA_ADVERTISED_LISTENERS` 对业务实例网络可达（见 [[../deploy/部署说明|部署说明]]）。
