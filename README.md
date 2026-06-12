# Hermes Agent 源码学习

> Hermes Agent 源码系统学习指南 — 12 个模块覆盖完整架构

## 📚 目录

| # | 模块 | 说明 |
|---|------|------|
| 01 | [项目总览与架构](docs/01-项目总览与架构.md) | 代码规模、技术栈、目录结构、核心模块 |
| 02 | [Agent Loop 核心循环](docs/02-Agent-Loop-核心循环.md) | 主循环、对话管理、流式输出 |
| 03 | [工具调用系统](docs/03-工具调用系统.md) | 工具注册、动态 schema、handle_function_call |
| 04 | [Skills 系统框架](docs/04-Skills-系统框架.md) | 渐进式披露、生命周期管理、安全扫描 |
| 05 | [Memory 记忆系统](docs/05-Memory-记忆系统.md) | 持久化知识、Provider 架构、容量管理 |
| 06 | [Cron 定时任务系统](docs/06-Cron-定时任务系统.md) | 无人值守执行、调度器、脚本模式 |
| 07 | [Context Compression](docs/07-Context-Compression.md) | 上下文压缩、摘要策略、保护区间 |
| 08 | [LLM 调用层](docs/08-LLM-调用层.md) | Provider 生态、call_llm、多层异常处理 |
| 09 | [Gateway 网关系统](docs/09-Gateway-网关系统.md) | 20+ 平台适配、消息路由、Agent 缓存 |
| 10 | [自学习系统](docs/10-自学习系统.md) | Background Review、Curator、Skill 生命周期 |
| 11 | [多Agent系统](docs/11-多Agent系统.md) | Delegation、Sub-Agent、并行执行、安全隔离 |
| 12 | [省Token优化](docs/12-省Token优化.md) | 20 种省 Token 机制、缓存利用、模型调用层优化 |

## 🏗️ 架构概览

```
Hermes Agent
├── Agent Loop（模块2） — 主循环引擎
├── Tool Calling（模块3） — 工具调用系统
├── Skills（模块4） — 程序性记忆
├── Memory（模块5） — 声明性记忆
├── Cron（模块6） — 定时任务
├── Context Compression（模块7） — 上下文压缩
├── LLM Layer（模块8） — 大模型调用
├── Gateway（模块9） — 多平台网关
├── Self-Learning（模块10） — 自学习
├── Delegation（模块11） — 多 Agent
└── Token Optimization（模块12） — 成本优化
```

## 📖 来源

- 源码仓库：Hermes Agent (Nous Research)
- 学习方式：逐模块阅读源码 → 提炼核心设计 → 结构化文档
- 同步飞书知识库：Hermes Agent 源码学习
