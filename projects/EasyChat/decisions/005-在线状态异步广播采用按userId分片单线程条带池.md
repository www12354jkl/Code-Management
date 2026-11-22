---
type: decision
date: 2026-11-22
status: 已采纳
---

# 005 - 在线状态异步广播采用按 userId 分片单线程条带池

## 背景：遇到了什么问题

`WsSessionManager` 在每次 Netty 连接 `register` 与 `unregister` 时通过 64 路 `userLocks` 分段锁串行维护本地 `Channel` 与 Redis 在线态后，会调用 `WsOnlineStatusBroadcaster.broadcastAsync(userId, online)` 向好友/群友推送 `ONLINE_STATUS`。

演进过程中暴露出两代线程池问题：
1. **第一代（无参 `CompletableFuture.runAsync`）**：任务直接提交给 `ForkJoinPool.commonPool()`，`broadcast()` 中的 `conversationMemberMapper.selectPeerIds` 数据库查询与 Kafka 发送会占用公共计算池；在 Spring Boot Fat JAR 下，`commonPool` 线程还会丢失 `LaunchedClassLoader` 上下文类加载器，同时缺少异常兜底与停机清理。
2. **第二代（共享多线程 `ThreadPoolExecutor(2, 8, LinkedBlockingQueue(512), DiscardOldestPolicy)`）**：虽然脱离了公共池并绑定了类加载器，但同一用户极短时间“断线 → 重连”提交的 `(userId, false)` 与 `(userId, true)` 会被不同工作线程并发执行，可能以反序调用 `pushRouter.route(peer, ...)`（Kafka 模式按接收端 `peer` 分区，入队反序会直接把对端 UI 状态覆盖成离线）；同时 `core=2` 配 `512` 容量队列难以及时扩容、`DiscardOldestPolicy` 静默丢弃无日志、单好友推送失败会中断循环且丢失 `MDC traceId`。

## 备选方案

1. **方案 A：继续使用单个共享多线程 `ThreadPoolExecutor` + 业务层版本号/Redis 状态校验**
   - 优点：线程池实例数最少。
   - 缺点：仅靠单次校验无法避免校验后至 `pushRouter.route` 之间的时间窗竞争；若回查 Redis/SessionManager 还会引入额外网络开销或循环依赖。
2. **方案 B：按 `userId` 分片的 `16` 路单线程条带执行器（Striped Single-Thread `ThreadPoolExecutor`）**
   - 优点：同一 `userId` 的全部上下线事件固定哈希到同一条带（`1` 线程 FIFO 队列），天然消除同用户状态广播乱序，与 `WsSessionManager` 的 64 路分段锁形成端到端单用户有序闭环；不同用户跨条带天然并发，且每路有界队列（`64`）+ 自定义淘汰告警 + `appClassLoader` 绑定 + `MDC traceId` 透传 + 5 秒共享截止时间优雅停机。
   - 缺点：维护 16 个守护线程（内存与调度开销在百万级并发以下可忽略）。

## 最终选择与理由

选择**方案 B**。实时通讯在线状态推送最关键的是“同一用户最终状态不能因并发交叉而被旧离线事件覆写”，按 `userId` 分片单线程执行器从并发结构上直接消除了竞争，同时通过逐个 `peer` `try/catch` 隔离、可观测的队列淘汰日志与 `WsOnlineStatusBroadcasterTest` 确定性并发单测保障长期稳定性。

## 影响与后续注意点

- 相关代码：`WsOnlineStatusBroadcaster`、`WsOnlineStatusBroadcasterTest`、[[002-多实例WebSocket广播引入Kafka独立消费组与Redis-CAS在线态|ADR-002]]、[[../architecture/架构说明|架构说明]]、[[../architecture/Redis与Kafka设计|Redis 与 Kafka 设计]]。
- 每路条带当前队列上限为 `64`、总条带数为 `16`；若单实例突发同时上下线连接数明显超过 `1024`，可通过日志关键词 `在线状态广播队列已满，丢弃最旧任务` 监控并上调 `EXECUTOR_STRIPES` 或 `STRIPE_QUEUE_CAPACITY`。
- 禁止在 `WsOnlineStatusBroadcaster` 中直接注入 `WsSessionManager` 以免形成双向循环依赖（`WsSessionManager` 已构造注入 `WsOnlineStatusBroadcaster`）。
