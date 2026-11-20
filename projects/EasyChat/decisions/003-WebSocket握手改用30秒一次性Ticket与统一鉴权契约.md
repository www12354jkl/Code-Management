---
type: decision
date: 2026-10-21
status: 已采纳
---

# 003 - WebSocket 握手改用 30 秒一次性 Ticket 与统一鉴权契约

## 背景：遇到了什么问题
浏览器原生 `WebSocket` API 无法在握手阶段自定义 `Authorization` 请求头。若直接把长期有效的 Access Token（默认 30 分钟）拼接在 `/ws?token=...` 查询参数中，令牌会明文出现在网关日志、代理访问日志与浏览器历史中，增大泄露风险。另一方面，前端 `request.ts` 拦截器在早期就约定了业务接口遇未授权时返回 HTTP `200` + JSON `{"code":401,"message":...}` 再触发无感刷新，而 WebSocket 握手失败则要求标准 HTTP `401` 拒绝连接升级。

## 备选方案
1. **方案 A：WebSocket 建立后再发首帧 Auth 报文，或直接复用 Cookie 鉴权**
   - 优点：URL 上不带凭证。
   - 缺点：连接未认证前会先占满 Netty 升级与会话资源，且需处理前后端跨越同源/Electron `file://` 本地加载时的 Cookie 跨域与 `SameSite` 兼容问题。
2. **方案 B：引入 30 秒一次性 Redis `ws:ticket` + 按升级请求区分 HTTP/WS 错误响应契约**
   - 优点：客户端先携带 Bearer Token 请求 `POST /api/chat/ws-ticket`，换取 32 位随机票据写入 Redis `ws:ticket:{ticket}`（TTL `30s`）；握手时仅携带 `/ws?ticket=...`，由网关 `GatewayAuthFilter` 预检存在性并注入 `X-Ws-Ticket`，到达 Netty `AuthWsHandler` 后调用 Redis `GETDEL` 原子消费；普通 HTTP 接口维持 HTTP `200 + code:401` 契约，检测到 `Upgrade` 头时返回标准 HTTP `401`。
   - 缺点：建立长连接前多一次轻量 `POST /api/chat/ws-ticket` 往返。

## 最终选择与理由
选择**方案 B**。一次性 30 秒票据即使在反向代理日志中出现也无法二次复用，兼顾了浏览器与 Electron 桌面端兼容性，并在网关层即可挡掉非法或未携票据的连接升级请求。

## 影响与后续注意点
- 相关代码：`WsTicketService`、`GatewayAuthFilter`、`AuthWsHandler`、前端 `ws.ts` 与 `request.ts`，流程详见 [[../flows/easychat-ws-kafka-push-flow.html|WS 与 Kafka 推送时序图]]。
- `ws:ticket` 在客户端收到 `401` 重连时必须通过 `refreshSession()` 刷新 Access Token 后重新申请，不能复用旧票据。
- 修改 `GatewayAuthFilter` 响应格式时必须保留“普通 HTTP 返回 `200 + code:401`、WS 升级失败返回 HTTP `401`”的契约，否则会破坏前端无感刷新拦截逻辑。
