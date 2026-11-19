---
type: project
status: 进行中
created: 2026-11-19
tech:
  - Java 21
  - Spring Boot 3.3.5
  - Spring Cloud 2023.0.3
  - Spring Cloud Alibaba 2023.0.3.2
  - Apache Dubbo 3.3.6
  - Nacos 2.3.2
  - MyBatis-Plus 3.5.9
  - Redis 7.2
  - RabbitMQ 3.13
  - MySQL 8.0
  - Caffeine
  - Fastjson2
repo: D:\workspace\OneGate-Manual
---

# OneGate-Manual

## 一句话简介

高并发 AI 流量网关与实时 Token 计费平台（基于 Java 21、Apache Dubbo、Redis Lua、RabbitMQ 与 MySQL）。

## 解决什么问题

- **高并发配额防超卖**：通过 Redis Lua 原子脚本实现单次请求幂等校验、分钟级 RPM 限流与配额预冻结。
- **准确的多退少补结算与超时兜底**：请求完成后按实际 Token 消耗退还或补扣差额；调用异常或超时未结算时，通过 RabbitMQ 5 分钟 TTL 延迟队列与 DLX 死信队列触发全额回滚。
- **多租户鉴权与多模型渠道路由**：为网关层提供租户 API Key 校验、欠费短路保护及多渠道加权负载均衡配置能力（网关模块待建设）。

## 技术栈

- **语言与基础框架**：Java 21（开启虚拟线程）、Spring Boot 3.3.5、Spring Cloud 2023.0.3、Spring Cloud Alibaba 2023.0.3.2
- **微服务与注册中心**：Apache Dubbo 3.3.6（Triple 协议）、Nacos 2.3.2
- **缓存与原子计算**：Redis 7.2（Lua 脚本）、Caffeine 3.1.8
- **消息队列与数据库**：RabbitMQ 3.13（DLX 死信延迟）、MySQL 8.0.36、MyBatis-Plus 3.5.9
- **工具库**：Fastjson2 2.0.53、Lombok 1.18.30

## 快速运行

> [!NOTE] 当前仓库状态
> 目前仅完成了基础设施编排、`onegate-common` 与 `onegate-billing-service` 的核心 RPC/Lua/MQ 拓扑配置。`onegate-billing-service` 尚缺 Spring Boot 启动类与 MQ 消费者，`onegate-gateway` 模块尚未创建，暂未形成完整端到端运行链路。

1. 在源码目录 `D:\workspace\OneGate-Manual` 启动中间件（MySQL、Redis、RabbitMQ、Nacos），详见 [[deploy/部署说明]]。
2. 在 `D:\workspace\OneGate-Manual\backend` 编译现有模块（`onegate-common`、`onegate-billing-service`）。

## 相关文档

- [[思维导图]]
- 架构：[[architecture/架构说明]]
- 中间件：[[architecture/Redis与RabbitMQ设计]]
- 数据库：[[architecture/数据库设计]]（[[architecture/html/onegate-erd.html|E-R 图]]）
- 流程图：[[flows/redis-rabbitmq-billing-flow.html|Redis 与 RabbitMQ 计费及死信回滚流程]]
- 部署：[[deploy/部署说明]]
