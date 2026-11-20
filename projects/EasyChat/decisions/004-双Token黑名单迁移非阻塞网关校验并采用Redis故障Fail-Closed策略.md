---
type: decision
date: 2026-11-15
status: 已采纳
---

# 004 - 双 Token 黑名单迁移非阻塞网关校验并采用 Redis Fail-Closed 策略

## 背景：遇到了什么问题
项目初期在复盘登出与令牌吊销链路时发现多处缺口：
1. `AuthServiceImpl.blacklistToken` 曾捕获并吞掉 Redis/JWT 解析异常，导致登出接口在黑名单未写入时仍返回成功；
2. 网关层 `GatewayAuthFilter` 曾使用阻塞式 Redis 客户端，存在 WebFlux EventLoop 线程阻塞隐患，且与下游 `JwtAuthFilter` 在 Bearer 裁剪、重复 `Authorization` 头及路径白名单边界（如 `;`、`..`、URL 编码绕过）上口径不一致；
3. 当 Redis 短时不可用时，简单吞错降级会让已登出或改密作废的旧令牌被错误放行；
4. 前端登出按钮在 Access Token 过期触发 401 自动刷新并发重放时，闭包里提前捕获的旧 `refreshToken` 会使服务端拉黑失准。

## 备选方案
1. **方案 A：仅保留无状态 JWT 短有效期，Redis 异常时 Fail-Open 放行以优先保可用性**
   - 优点：系统对 Redis 依赖更弱。
   - 缺点：主动登出和修改密码后无法保证旧令牌即时失效，且在 Redis 异常期间形成鉴权旁路。
2. **方案 B：网关非阻塞全量校验 + 下游二次校验 + Redis 不可用 Fail-Closed（503）+ 前端动态求值重放**
   - 优点：`GatewayAuthFilter` 改用 `ReactiveStringRedisTemplate`，与下游 `JwtAuthFilter`、共享 `TokenUtils` 对齐 Bearer 提取/引号归一化、段边界白名单与路径遍历拦截，校验 `token:blacklist`、`token:revoke-before` 与拒绝 `refresh` 令牌越权访问；Redis 出现 `DataAccessException` 时明确返回 HTTP `503`（不误报 `401` 引发前端错误清会话）；前端 `request.ts` 支持 `body` 工厂函数在 401 单飞刷新后重放时动态读取最新 `refreshToken`。
   - 缺点：Redis 成为强依赖安全组件，故障期间已登录流量会被拒绝为 `503`。

## 最终选择与理由
选择**方案 B**。对于即时通讯系统而言，登出后/改密后旧会话可被继续访问属于高风险安全缺陷，因此必须坚持“登出失败显式暴露、Redis 故障 Fail-Closed（503）、网关与下游双层防御”，并用覆盖网关/过滤器/服务/前端登出全链路的单元测试兜底。

## 影响与后续注意点
- 相关模块：`common-security`、`easychat-gateway`、`easychat-user`、前端 `request.ts` / `auth.ts` / `HomeView.vue`，设计细节见 [[../architecture/架构说明|架构说明]] 与 [[../architecture/Redis与Kafka设计|Redis 与 Kafka 设计]]。
- `token:blacklist:{token}` 与 `token:revoke-before:{userId}` 在网关与下游服务之间共享同一套 Redis 键前缀与校验口径，任何修改都须双端同步（见 [[../devlog/2026-11-20|2026-11-20 开发日志]]）。
