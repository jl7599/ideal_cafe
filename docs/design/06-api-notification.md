# 通知模块 API 设计

## 1. 概述

通知模块不独立存在，而是基于其他模块的事件触发。MVP 阶段采用轮询方案：

- C端：轮询订单状态变更
- S端：轮询待处理订单检测新订单

**后续迭代方向**：WebSocket / SSE 实现实时推送。

## 2. 接口列表

### 2.1 GET /api/v1/orders/{id}/status — 轮询订单状态（C端）

**权限**：customer（仅自己的订单）

**说明**：C端订单详情页定时调用此接口检测状态变化。相比 GET /orders/{id}，此接口更轻量，仅返回状态相关字段。

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
- 离开页面时停止轮询
- 状态变为终态（completed/cancelled/rejected）后停止轮询

### 2.2 GET /api/v1/orders/pending — 最新待处理订单（S端）

**说明**：此接口定义见订单模块 04-api-order.md 2.10 节。S端定时调用检测新订单。

**轮询建议**：
- S端订单处理页：每 3 秒轮询一次
- 优先比较 `latest_pending_order_id` 是否变化，变化时触发声音提示和视觉提示
- `pending_count` 仅用于角标展示，不能单独作为新订单判断依据
- 页面不可见时可降低频率

### 2.3 GET /api/v1/shop/status — 营业状态（复用）

**说明**：见门店模块 05-api-shop.md。C端菜单页和购物车页定时轮询此接口检测营业状态变化。

**轮询建议**：
- 菜单页/购物车页：每 10 秒轮询一次
- 状态变化时更新页面 UI（显示/隐藏加入购物车、去结算按钮状态）

## 3. C端通知触发逻辑（前端实现）

```
订单详情页：
  定时轮询 GET /orders/{id}/status
  │
  ├─ status 变化
  │    ├─ 更新进度条
  │    └─ 顶部弹出提示条
  │         ├─ accepted → "您的订单已被接单，预计 XX 分钟后出品"
  │         ├─ preparing → "您的咖啡正在制作中"
  │         ├─ ready → "您的咖啡已制作完成，请到吧台取餐，取餐码：XXXX"
  │         ├─ cancelled → "订单已取消"
  │         └─ rejected → "您的订单已被拒绝，原因：XXX"
  │
  └─ status 为终态 → 停止轮询
```

## 4. S端新订单检测逻辑（前端实现）

```
S端订单处理页：
  定时轮询 GET /orders/pending
  │
  ├─ latest_pending_order_id 变化
  │    ├─ 播放提示音（1-2 秒短促铃声）
  │    └─ 页面顶部弹出提示条 "新订单 #XXXXX"
  │         └─ 3 秒后自动消失
  │
  └─ latest_pending_order_id 不变 → 无操作
```

## 5. MVP 轮询策略汇总

| 端 | 页面 | 接口 | 间隔 | 停止条件 |
|----|------|------|------|---------|
| C端 | 订单详情 | GET /orders/{id}/status | 5 秒 | 状态为终态 / 离开页面 |
| C端 | 订单列表 | GET /orders?status=pending,accepted,preparing,ready | 10 秒 | 离开页面 |
| C端 | 菜单页 | GET /shop/status | 10 秒 | 离开页面 |
| C端 | 购物车 | GET /shop/status | 10 秒 | 离开页面 |
| S端 | 订单处理 | GET /orders/pending | 3 秒 | 离开页面 |

## 6. 后续迭代

| 方案 | 适用场景 | 复杂度 |
|------|---------|--------|
| WebSocket | S端实时新订单推送 | 中 |
| SSE | C端订单状态实时更新 | 低 |
| 微信订阅消息 | C端离线通知 | 中 |

SSE 适合 C端（单向推送），WebSocket 适合 S端（双向通信，未来可支持更多实时功能）。
