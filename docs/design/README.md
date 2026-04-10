# 详细设计文档

Ideal Cafe 项目的详细设计文档，基于 [PRD](../prd/) 输出。

## 文档索引

| 文档 | 内容 | 状态 |
|------|------|------|
| [00-architecture](00-architecture.md) | 项目架构、技术栈、项目结构、认证架构、部署架构 | ✅ |
| [01-database](01-database.md) | 数据库设计：8 张表、字段定义、索引、关系 | ✅ |
| [02-api-auth](02-api-auth.md) | 认证模块：微信登录、账号密码登录、用户/店员管理 | ✅ |
| [03-api-menu](03-api-menu.md) | 菜单模块：分类 CRUD、商品 CRUD、上下架、文件上传 | ✅ |
| [04-api-order](04-api-order.md) | 订单模块：下单、状态流转、取消/拒绝、再来一单 | ✅ |
| [05-api-shop](05-api-shop.md) | 门店模块：门店信息、营业状态（手动）、公告 | ✅ |
| [06-api-notification](06-api-notification.md) | 通知模块：轮询策略、C端状态检测、S端新订单检测 | ✅ |

## 设计决策记录

1. **C端与 S/A端拆为独立前端项目**：页面结构、认证方式、编译目标完全不同，合并成本 > 维护两个项目
2. **价格存整数（分）**：避免浮点精度问题，展示时转换
3. **营业时间为展示文本**：营业状态完全由管理员手动控制，不做时间自动判定，简化实现
4. **订单商品快照含 product_id**：订单创建时保存商品 ID + 名称 + 价格，商品删除后 product_id 自动置 NULL（FK ON DELETE SET NULL），支持"再来一单"精确匹配在架商品
5. **MVP 轮询方案**：C端订单详情 5 秒轮询状态、订单列表 10 秒轮询当前订单，S端 3 秒轮询新订单数量，后续迭代切 SSE/WebSocket
6. **店员删除 = 软删除**：设置 is_active = FALSE，保留历史关联数据；Token 验证时检查 is_active，软删除即时失效
7. **双 Token + refresh_token 黑名单**：access_token 2 小时 + refresh_token 30 天，登出/改密码时将 refresh_token 加入数据库黑名单表
8. **A端仅查看订单，不可操作**：与 PRD 权限矩阵一致，admin 不接单/不操作订单状态
9. **文件上传 MVP 存本地**：MVP 阶段存服务器磁盘，后续迁移到 OSS
10. **users 表预留 unionid + session_key**：为后续跨端扩展和微信敏感数据解密预留字段
11. **管理员可通过 API 创建**：MVP 阶段即允许管理员创建其他管理员和店员账号，不限制 role 字段

## 数据库表一览

| 表名 | 说明 | 关联 |
|------|------|------|
| users | C端用户 | → orders |
| staff | 店员/管理员 | - |
| categories | 商品分类 | → products |
| products | 商品 | → categories |
| shop | 门店信息（单行） | - |
| orders | 订单 | → users, → order_items |
| order_items | 订单商品快照（含 product_id） | → orders |
| token_blacklist | Token 黑名单 | - |
