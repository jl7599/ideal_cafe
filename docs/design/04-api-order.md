# 订单模块 API 设计

## 1. 概述

订单模块是核心业务模块，涉及三端：
- C端：下单、查看订单、取消订单、再来一单
- S端：接单、拒绝、状态流转
- A端：仅查看全量订单，不可操作

**状态机**：

```
pending → accepted → preparing → ready → completed
pending → cancelled（C端取消）
pending → rejected（S端拒绝）
```

**关键约束**：
- 订单金额以后台商品价格为准，不信任前端传入的金额
- 订单创建时保存商品快照（含 product_id）
- 使用幂等键防重复提交，同时保留短时间限流
- 门店打烊时拒绝下单，但进行中订单不受打烊影响
- A端仅查看，不操作订单状态（与 PRD 权限矩阵一致）

## 2. 接口列表

### 2.1 POST /api/v1/orders — 创建订单（C端）

**权限**：customer

**请求头**：

| Header | 必填 | 说明 |
|--------|------|------|
| Idempotency-Key | 是 | UUID。同一次提交重试必须使用同一个值 |

**请求体**：

```json
{
  "items": [
    {
      "product_id": "uuid",
      "quantity": 2
    },
    {
      "product_id": "uuid",
      "quantity": 1
    }
  ],
  "remark": "少冰"
}
```

**校验规则**：
- `Idempotency-Key`：必填，UUID。同一次提交重试必须使用同一个值
- items：非空数组，每项 product_id 必填，quantity 正整数；同一 product_id 不可重复出现，否则返回 400
- remark：选填，最大 200 字符
- 所有 product_id 必须存在且上架
- 校验门店营业状态，打烊时拒绝下单

**处理流程**：
1. 校验门店是否营业
2. 先按 `Idempotency-Key` 查询是否已存在成功创建的订单，若存在则直接返回已有订单
3. 校验所有商品是否存在且上架
4. 按后台商品价格计算 total_amount（不使用前端传入的价格）
5. 执行短时间限流，防止同一用户极短时间内连续创建多笔订单
6. 生成订单号 + 取餐码
7. 创建订单 + 订单商品快照（含 product_id、product_name、product_price）
8. 返回订单详情

**幂等与限流策略**：
- 幂等：客户端每次点击「提交订单」前生成 `Idempotency-Key`，同一次提交的重试请求必须复用该值；服务端按该键返回同一笔订单，避免重复创建
- 限流：同一用户 3 秒内最多创建 1 笔新订单；命中限流且幂等键不同，返回 429

**成功响应** 201：

```json
{
  "id": "uuid",
  "order_no": "202604091430258827",
  "status": "pending",
  "total_amount": 8400,
  "total_amount_display": "84.00",
  "remark": "少冰",
  "pickup_code": "5827",
  "items": [
    {
      "product_name": "燕麦拿铁",
      "product_price": 2800,
      "product_price_display": "28.00",
      "quantity": 2,
      "subtotal": 5600,
      "subtotal_display": "56.00"
    },
    {
      "product_name": "美式咖啡",
      "product_price": 2800,
      "product_price_display": "28.00",
      "quantity": 1,
      "subtotal": 2800,
      "subtotal_display": "28.00"
    }
  ],
  "created_at": "2026-04-10T14:30:25+08:00"
}
```

**错误响应**：
- 400：门店已打烊 / 商品已下架
- 429：提交过频，请稍后

### 2.2 GET /api/v1/orders — 订单列表

**权限**：customer、staff、admin

**查询参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| status | string | 否 | 按状态筛选，逗号分隔（如 `pending,accepted`） |
| start_date | date | 否 | 起始日期，格式 yyyy-MM-dd（A端用） |
| end_date | date | 否 | 结束日期，格式 yyyy-MM-dd（A端用） |
| page | integer | 否 | 页码，默认 1 |
| page_size | integer | 否 | 每页条数，默认 20，最大 50 |

**行为差异**：
- customer：仅返回自己的订单，按 created_at DESC
- staff/admin：返回全量订单，按 created_at DESC

**成功响应** 200：

```json
{
  "items": [
    {
      "id": "uuid",
      "order_no": "202604091430258827",
      "order_no_short": "258827",
      "status": "pending",
      "status_label": "待处理",
      "total_amount": 8400,
      "total_amount_display": "84.00",
      "product_summary": "燕麦拿铁 x2, 美式 x1",
      "user_nickname": "理想咖啡5827",
      "created_at": "2026-04-10T14:30:25+08:00"
    }
  ],
  "total": 50,
  "page": 1,
  "page_size": 20
}
```

**说明**：
- `order_no_short`：订单号后 6 位，列表展示用
- `status_label`：状态中文标签
- `product_summary`：商品摘要，如 "燕麦拿铁 x2, 美式 x1"
- `user_nickname`：下单用户昵称，customer 查看时省略

**status_label 映射**：

| status | status_label |
|--------|-------------|
| pending | 待处理 |
| accepted | 已接单 |
| preparing | 制作中 |
| ready | 待取餐 |
| completed | 已完成 |
| cancelled | 已取消 |
| rejected | 已拒绝 |

### 2.3 GET /api/v1/orders/{id} — 订单详情

**权限**：customer（仅自己的订单）、staff、admin

**成功响应** 200：

```json
{
  "id": "uuid",
  "order_no": "202604091430258827",
  "order_no_short": "258827",
  "status": "pending",
  "status_label": "待处理",
  "total_amount": 8400,
  "total_amount_display": "84.00",
  "remark": "少冰",
  "pickup_code": "5827",
  "cancel_reason": "",
  "reject_reason": "",
  "estimated_wait_minutes": null,
  "items": [
    {
      "product_name": "燕麦拿铁",
      "product_price": 2800,
      "product_price_display": "28.00",
      "quantity": 2,
      "subtotal": 5600,
      "subtotal_display": "56.00"
    }
  ],
  "user": {
    "id": "uuid",
    "nickname": "理想咖啡5827",
    "avatar_url": ""
  },
  "created_at": "2026-04-10T14:30:25+08:00",
  "updated_at": "2026-04-10T14:30:25+08:00"
}
```

**说明**：
- customer 查看时 `user` 字段可省略（就是自己）
- staff/admin 查看时包含 `user` 信息

**错误响应**：
- 404：订单不存在
- 403：customer 访问他人订单

### 2.4 PUT /api/v1/orders/{id}/cancel — 取消订单（C端）

**权限**：customer（仅自己的订单）

**请求体**：

```json
{
  "cancel_reason": "不想要了"
}
```

**校验规则**：
- cancel_reason：选填，最大 200 字符
- 仅 `pending` 状态可取消

**成功响应** 200：

```json
{
  "id": "uuid",
  "status": "cancelled",
  "status_label": "已取消",
  "cancel_reason": "不想要了"
}
```

**错误响应**：
- 409：订单状态不允许取消（非 pending）
- 403：不是自己的订单

### 2.5 PUT /api/v1/orders/{id}/accept — 接单（S端）

**权限**：staff

**请求体**：

```json
{
  "estimated_wait_minutes": 10
}
```

**校验规则**：
- estimated_wait_minutes：正整数，选填
- 仅 `pending` 状态可接单

**成功响应** 200：

```json
{
  "id": "uuid",
  "status": "accepted",
  "status_label": "已接单",
  "estimated_wait_minutes": 10
}
```

**错误响应**：
- 409：订单状态不允许接单

### 2.6 PUT /api/v1/orders/{id}/reject — 拒绝订单（S端）

**权限**：staff

**请求体**：

```json
{
  "reject_reason": "今日咖啡豆已用完"
}
```

**校验规则**：
- reject_reason：必填，最大 200 字符
- 仅 `pending` 状态可拒绝

**成功响应** 200：

```json
{
  "id": "uuid",
  "status": "rejected",
  "status_label": "已拒绝",
  "reject_reason": "今日咖啡豆已用完"
}
```

**错误响应**：
- 400：拒绝原因为空
- 409：订单状态不允许拒绝

### 2.7 PUT /api/v1/orders/{id}/prepare — 开始制作（S端）

**权限**：staff

**请求体**：无

**前置状态**：accepted

**成功响应** 200：

```json
{
  "id": "uuid",
  "status": "preparing",
  "status_label": "制作中"
}
```

**错误响应**：
- 409：订单状态不允许该操作

### 2.8 PUT /api/v1/orders/{id}/ready — 制作完成/待取餐（S端）

**权限**：staff

**请求体**：无

**前置状态**：preparing

**成功响应** 200：

```json
{
  "id": "uuid",
  "status": "ready",
  "status_label": "待取餐",
  "pickup_code": "5827"
}
```

**错误响应**：
- 409：订单状态不允许该操作

### 2.9 PUT /api/v1/orders/{id}/complete — 确认取餐/完成（S端）

**权限**：staff

**请求体**：无

**前置状态**：ready

**成功响应** 200：

```json
{
  "id": "uuid",
  "status": "completed",
  "status_label": "已完成"
}
```

**错误响应**：
- 409：订单状态不允许该操作

### 2.10 GET /api/v1/orders/{id}/status — 轮询订单状态（C端）

**权限**：customer（仅自己的订单）

轻量接口，C端订单详情页和订单列表页定时调用此接口检测状态变化。完整订单详情请使用 `GET /orders/{id}`。

**成功响应** 200：

```json
{
  "status": "accepted",
  "status_label": "已接单",
  "estimated_wait_minutes": 10,
  "updated_at": "2026-04-10T14:31:00+08:00"
}
```

**轮询建议**：
- 订单详情页：每 5 秒轮询一次
- 订单列表页：每 10 秒轮询当前订单（`GET /orders?status=pending,accepted,preparing,ready`）
- 状态变为终态（completed/cancelled/rejected）后停止轮询
- 离开页面时停止轮询

> **说明**：此端点的轮询策略详见 [通知模块](06-api-notification.md)。

### 2.11 GET /api/v1/orders/pending — 最新待处理订单（S端）

**权限**：staff

用于 S端轮询检测新订单，返回最近待处理订单列表。

**查询参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| limit | integer | 否 | 返回条数，默认 5，最大 10 |

**成功响应** 200：

```json
{
  "pending_count": 3,
  "latest_pending_order_id": "uuid",
  "latest_pending_created_at": "2026-04-10T14:30:25+08:00",
  "latest_orders": [
    {
      "id": "uuid",
      "order_no_short": "258827",
      "product_summary": "燕麦拿铁 x2",
      "total_amount": 5600,
      "total_amount_display": "56.00",
      "created_at": "2026-04-10T14:30:25+08:00"
    }
  ]
}
```

**说明**：
- `pending_count`：待处理订单总数，用于角标展示
- `latest_pending_order_id`：当前最新 pending 订单 ID，前端主游标
- `latest_pending_created_at`：当前最新 pending 订单创建时间，可作为辅助游标
- `latest_orders`：最新的 pending 订单，用于展示新订单提示条
- 前端应优先对比 `latest_pending_order_id`，游标变化则触发声音提示 + 展示提示条；不能只比较 `pending_count`

### 2.11 POST /api/v1/orders/{id}/reorder — 再来一单（C端）

**权限**：customer（仅自己的订单）

根据历史订单的商品信息，返回当前仍可购买的商品列表。

**处理流程**：
1. 查询订单的 order_items，获取 product_id 列表
2. 根据 product_id 查询商品当前状态（是否仍在架）
3. 返回在架商品列表 + 已下架提示

**成功响应** 200：

```json
{
  "available_items": [
    {
      "product_id": "uuid",
      "name": "燕麦拿铁",
      "price": 2800,
      "price_display": "28.00",
      "image_url": "https://...",
      "quantity": 2
    }
  ],
  "unavailable_items": [
    {
      "product_name": "季节限定拿铁",
      "reason": "已下架"
    }
  ],
  "is_shop_open": true
}
```

**说明**：
- `available_items`：仍在架的商品，`quantity` 为原订单数量，前端可据此加入购物车
- `unavailable_items`：已下架或已删除的商品，仅保留快照名称
- `is_shop_open`：门店营业状态，前端据此提示"门店已打烊"
- product_id 为 NULL 的 order_items（商品已被物理删除）归入 unavailable_items

**错误响应**：
- 404：订单不存在
- 403：不是自己的订单

## 3. 状态流转校验矩阵

| 当前状态 | → accepted | → preparing | → ready | → completed | → cancelled | → rejected |
|---------|:----------:|:-----------:|:-------:|:-----------:|:-----------:|:----------:|
| pending | ✅ (staff) | - | - | - | ✅ (customer) | ✅ (staff) |
| accepted | - | ✅ (staff) | - | - | - | - |
| preparing | - | - | ✅ (staff) | - | - | - |
| ready | - | - | - | ✅ (staff) | - | - |
| completed | - | - | - | - | - | - |
| cancelled | - | - | - | - | - | - |
| rejected | - | - | - | - | - | - |

## 4. 权限边界汇总

| 接口 | customer | staff | admin |
|------|----------|-------|-------|
| POST /orders | ✅ | - | - |
| GET /orders | 仅自己的 | 全量 | 全量 |
| GET /orders/{id} | 仅自己的 | 全量 | 全量 |
| GET /orders/{id}/status | 仅自己的 | - | - |
| PUT /orders/{id}/cancel | ✅ | - | - |
| PUT /orders/{id}/accept | - | ✅ | - |
| PUT /orders/{id}/reject | - | ✅ | - |
| PUT /orders/{id}/prepare | - | ✅ | - |
| PUT /orders/{id}/ready | - | ✅ | - |
| PUT /orders/{id}/complete | - | ✅ | - |
| GET /orders/pending | - | ✅ | - |
| POST /orders/{id}/reorder | ✅ | - | - |
