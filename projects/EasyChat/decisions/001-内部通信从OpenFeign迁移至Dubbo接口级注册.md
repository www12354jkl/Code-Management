---
type: decision
date: 2026-09-06
status: 已采纳
---

# 001 - 内部通信从 OpenFeign 迁移至 Apache Dubbo（接口级注册）

## 背景：遇到了什么问题
EasyChat 早期采用 OpenFeign 在 `easychat-user` 与 `easychat-chat` 之间调用 `Internal*Controller` 完成用户查询、好友校验与删好友清会话。这种方案需要额外维护内部 HTTP 接口与密钥拦截器（`InternalApiBlockFilter`），外部网关路由与内部接口安全边界耦合；同时在接入 Dubbo 后发现，若采用默认应用级服务发现，Dubbo 会占用与 Spring Boot 应用同名的 Nacos 服务名并注册 `20880`/`20881` 端口，导致 Spring Cloud Gateway 把 HTTP 流量错误转发到 Dubbo RPC 端口。

## 备选方案
1. **方案 A：继续保留 OpenFeign + 内部 HTTP Secret 过滤**
   - 优点：复用 Spring MVC 接口与网关负载体系，上手成本低。
   - 缺点：需要同步维护内部路由隔离与签名密钥过滤，序列化和长连接复用不如 RPC 紧凑，且缺少跨 RPC 链路装饰的原生滤波器模型。
2. **方案 B：Apache Dubbo 3.3 + `common-api` 契约 + 接口级注册（`register-mode: interface` / `FORCE_INTERFACE`）**
   - 优点：跨库接口直接收敛到 `common-api`（`UserQueryService`、`FriendRequestService`、`ConversationDeleteService`），配合 `TraceIdDubboFilter` 透传 MDC `traceId`，接口级注册天然避开 Nacos HTTP 同名实例端口冲突；配合 `failfast` 与显式 Fallback 精细控制失败策略。
   - 缺点：需要显式管理 Dubbo 网卡绑定（`DUBBO_HOST`）并避免 Consumer 降级静默吞错掩盖上游不可用。

## 最终选择与理由
选择**方案 B**。跨服务同步调用全部收敛到 `common-api` 与 Dubbo `register-mode: interface`（`FORCE_INTERFACE`），移除了 `Feign` 客户端与 `Internal*Controller`。针对降级语义：删好友清会话（`ConversationDeleteServiceFallback`）与关键准入校验（`UserQueryServiceFallback` 于 `2026-11-20` 统一改为抛出 `AppException.serviceUnavailable` 并在 `GlobalExceptionHandler` 映射为 HTTP `503`），保证关键写路径与校验不将上游宕机误报为 `404`/`400`；列表头像昵称、消息撤回文案补全等非关键读路径则通过 `UserQuerySupport` 显式捕获 `503` 后优雅降级。

## 影响与后续注意点
- 相关模块：`common-api`、`easychat-user`、`easychat-chat`、[[../architecture/架构说明|架构说明]]。
- 变更配置时必须保留 `dubbo.application.register-mode: interface` 与 `service-discovery.migration: FORCE_INTERFACE`。
- 本机多网卡环境下若遇到 RPC 超时，须在不入库的 `application-local.yml` 中固定 `DUBBO_HOST`（见 [[../deploy/部署说明|部署说明]]）。
