# Ideal Cafe 工作计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

## 里程碑索引

| 编号 | 名称 | 计划文件 | 后端 | C端 | S/A端 | 测试 | 部署 | 合计 |
|------|------|---------|------|-----|-------|------|------|------|
| M1 | 项目脚手架 & 数据库 | [00-scaffold.md](00-scaffold.md) | 8 | 4 | 4 | 1 | 3 | 20 |
| M2 | 认证模块 | [01-auth.md](01-auth.md) | 11 | 4 | 3 | 1 | - | 19 |
| M3 | 菜单模块 | [02-menu.md](02-menu.md) | 7 | 3 | 3 | 3 | - | 16 |
| M4 | 门店模块 | [03-shop.md](03-shop.md) | 7 | 2 | 3 | 1 | - | 13 |
| M5 | 订单模块 | [04-order.md](04-order.md) | 15 | 5 | 3 | 3 | - | 26 |
| M6 | 通知/轮询 | [05-notification.md](05-notification.md) | - | 4 | 3 | 1 | - | 8 |
| M7 | 部署上线 | [06-deployment.md](06-deployment.md) | - | - | - | 1 | 5 | 6 |
| | **合计** | | **48** | **22** | **19** | **11** | **8** | **108** |

## 依赖关系

```
M1(脚手架) → M2(认证) → M3(菜单) ──→ M5(订单) → M6(通知) → M7(部署)
                    └→ M4(门店) ──┘
```

- M3 和 M4 可并行开发
- M5 依赖 M3（订单引用商品）+ M4（下单需校验营业状态）
- M6 依赖 M5（轮询订单状态）
- M7 依赖全部完成

## 复杂度定义

| 级别 | 含义 | 预估时间 |
|------|------|---------|
| S | 逻辑简单，套路代码 | 0.5-1 天 |
| M | 有业务逻辑或交互 | 1-2 天 |
| L | 复杂业务逻辑或多组件联动 | 2-3 天 |

## 任务 ID 命名规则

`M{里程碑}-{角色}{序号}`

- B = 后端 (Backend)
- C = C端前端
- S = S/A端前端
- T = 测试
- D = 部署

示例：`M2-B04` = 里程碑2 后端第4个任务

## 参考文档

- [PRD 文档](../prd/)
- [设计文档](../design/)
  - [00-architecture](../design/00-architecture.md) — 架构、技术栈、项目结构
  - [01-database](../design/01-database.md) — 8 张表、字段定义、索引
  - [02-api-auth](../design/02-api-auth.md) — 认证模块 API
  - [03-api-menu](../design/03-api-menu.md) — 菜单模块 API
  - [04-api-order](../design/04-api-order.md) — 订单模块 API
  - [05-api-shop](../design/05-api-shop.md) — 门店模块 API
  - [06-api-notification](../design/06-api-notification.md) — 通知轮询策略
