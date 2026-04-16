# M6: 通知/轮询 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 实现前端轮询策略——C端订单状态自动更新、S端新订单检测（声音+视觉提示）、营业状态变化感知。全部基于已有 API 的前端轮询实现，无需新增后端接口。

**Architecture:** MVP 采用轮询方案，C端订单详情 5s、订单列表 10s、S端新订单 3s、营业状态 10s。后续迭代切 SSE/WebSocket。

**Tech Stack:** Vue3 composables, uni-app API

**前置依赖：** M5 完成

---

## 文件结构

```
frontend/src/
├── composables/
│   ├── useOrderStatusPolling.ts    # C端订单状态轮询
│   ├── useOrderListPolling.ts      # C端订单列表轮询
│   └── useShopStatusPolling.ts     # C端营业状态轮询
├── components/
│   ├── OrderStatusToast.vue        # 状态变化提示条
│   └── ShopClosedOverlay.vue       # 打烊遮罩

admin/src/
├── composables/
│   └── usePendingOrderPolling.ts   # S端新订单轮询
├── components/
│   ├── NewOrderAlert.vue           # 新订单视觉提示
│   └── NewOrderSound.vue           # 新订单声音提示
├── static/
│   └── order-bell.mp3              # 提示音文件
```

---

## C端前端任务

### M6-C01: 订单状态轮询 composable

**Files:**
- Create: `frontend/src/composables/useOrderStatusPolling.ts`

**Steps:**

- [ ] 创建轮询 composable：
  ```typescript
  import { ref, onMounted, onUnmounted } from 'vue'
  import { request } from '../api/request'

  export function useOrderStatusPolling(orderId: Ref<string>) {
    const status = ref('')
    const statusLabel = ref('')
    const estimatedWaitMinutes = ref<number | null>(null)
    const isTerminal = ref(false)
    let timer: ReturnType<typeof setInterval> | null = null

    const poll = async () => {
      try {
        const data = await request({
          url: `/api/v1/orders/${orderId.value}/status`,
        })
        status.value = data.status
        statusLabel.value = data.status_label
        estimatedWaitMinutes.value = data.estimated_wait_minutes

        if (['completed', 'cancelled', 'rejected'].includes(data.status)) {
          isTerminal.value = true
          stop()
        }
      } catch {
        // 静默失败，下次重试
      }
    }

    const start = () => {
      stop()
      poll()
      timer = setInterval(poll, 5000)
    }

    const stop = () => {
      if (timer) {
        clearInterval(timer)
        timer = null
      }
    }

    onMounted(start)
    onUnmounted(stop)

    return { status, statusLabel, estimatedWaitMinutes, isTerminal, start, stop }
  }
  ```

- [ ] 在订单详情页使用：`const { status, statusLabel, isTerminal } = useOrderStatusPolling(orderId)`
- [ ] Commit: `feat: add order status polling composable (5s interval)`

**验收标准：** 订单详情页自动轮询，终态停止，离页停止。

---

### M6-C02: 订单列表轮询

**Files:**
- Create: `frontend/src/composables/useOrderListPolling.ts`
- Modify: `frontend/src/pages/order/index.vue`

**Steps:**

- [ ] 创建订单列表轮询 composable（10s 间隔）：
  - 轮询 `GET /orders?status=pending,accepted,preparing,ready`
  - 检测新订单或状态变化时刷新列表
  - 离页停止
- [ ] 在订单列表页使用
- [ ] Commit: `feat: add order list polling composable (10s interval)`

**验收标准：** 订单列表自动刷新。

---

### M6-C03: 状态变化提示

**Files:**
- Create: `frontend/src/components/OrderStatusToast.vue`
- Modify: `frontend/src/pages/order-detail/index.vue`

**Steps:**

- [ ] 创建 `OrderStatusToast.vue`：
  - 顶部弹出提示条
  - accepted → "您的订单已被接单，预计 XX 分钟后出品"
  - preparing → "您的咖啡正在制作中"
  - ready → "您的咖啡已制作完成，请到吧台取餐，取餐码：XXXX"
  - cancelled → "订单已取消"
  - rejected → "您的订单已被拒绝，原因：XXX"
  - 3 秒后自动消失
- [ ] 在订单详情页监听 status 变化，触发提示
- [ ] Commit: `feat: add order status change toast notifications`

**验收标准：** 状态变化时显示对应提示文案。

---

### M6-C04: 营业状态轮询

**Files:**
- Create: `frontend/src/composables/useShopStatusPolling.ts`
- Modify: `frontend/src/pages/index/index.vue`
- Modify: `frontend/src/pages/cart/index.vue`
- Create: `frontend/src/components/ShopClosedOverlay.vue`

**Steps:**

- [ ] 创建 `useShopStatusPolling.ts`（10s 间隔）：
  - 轮询 `GET /shop/status`
  - 返回 is_open, status_label, announcement
  - 检测变化时触发回调
- [ ] 创建 `ShopClosedOverlay.vue`：打烊遮罩，禁用加入购物车和结算
- [ ] 在菜单页和购物车页使用：
  - 菜单页：打烊时隐藏"加入购物车"按钮
  - 购物车页：打烊时禁用"去结算"按钮，显示提示
- [ ] Commit: `feat: add shop status polling (10s) with closed overlay`

**验收标准：** 打烊时菜单页隐藏加购物车按钮，购物车页禁用结算。

---

## S/A端前端任务

### M6-S01: 新订单轮询 composable

**Files:**
- Create: `admin/src/composables/usePendingOrderPolling.ts`

**Steps:**

- [ ] 创建 `usePendingOrderPolling.ts`（3s 间隔）：
  ```typescript
  export function usePendingOrderPolling() {
    const pendingCount = ref(0)
    const latestOrderId = ref('')
    const latestOrders = ref([])
    const hasNewOrder = ref(false)
    let lastKnownOrderId = ''
    let timer: ReturnType<typeof setInterval> | null = null

    const poll = async () => {
      try {
        const data = await request({ url: '/api/v1/orders/pending' })
        pendingCount.value = data.pending_count
        latestOrders.value = data.latest_orders

        if (data.latest_pending_order_id !== lastKnownOrderId && lastKnownOrderId) {
          hasNewOrder.value = true
        }
        lastKnownOrderId = data.latest_pending_order_id
      } catch {}
    }

    // 页面不可见时降频（30s）
    const handleVisibility = () => {
      if (document.hidden) {
        stop()
        timer = setInterval(poll, 30000)
      } else {
        stop()
        start()
      }
    }

    // ...
  }
  ```

- [ ] 在订单处理页使用
- [ ] Commit: `feat: add pending order polling composable (3s interval)`

**验收标准：** 页面可见时 3s 轮询，不可见时降频 30s。

---

### M6-S02: 新订单声音提示

**Files:**
- Create: `admin/src/static/order-bell.mp3`
- Modify: `admin/src/composables/usePendingOrderPolling.ts`

**Steps:**

- [ ] 准备短促铃声文件（1-2 秒）放入 `static/`
- [ ] 在轮询检测到新订单时播放：
  ```typescript
  const playBell = () => {
    const audio = new Audio('/static/order-bell.mp3')
    audio.play().catch(() => {
      // 浏览器可能阻止自动播放，用户交互后恢复
    })
  }
  ```
- [ ] Commit: `feat: add new order sound alert`

**验收标准：** 新订单到达时播放短促铃声。

---

### M6-S03: 新订单视觉提示

**Files:**
- Create: `admin/src/components/NewOrderAlert.vue`
- Modify: `admin/src/pages/orders/index.vue`

**Steps:**

- [ ] 创建 `NewOrderAlert.vue`：
  - 页面顶部弹出提示条："新订单 #XXXXX"
  - 3 秒后自动消失
  - 点击可跳转订单详情
- [ ] 在订单处理页监听 hasNewOrder 变化，触发提示
- [ ] Commit: `feat: add new order visual alert`

**验收标准：** 新订单到达时顶部弹出提示，3 秒后消失。

---

## 测试任务

### M6-T01: 轮询行为集成测试

**Files:**
- Modify: `backend/tests/test_order_status.py`

**Steps:**

- [ ] 确保轮询依赖的后端接口正确：
  - `GET /orders/{id}/status` 返回正确状态
  - `GET /orders/pending` 返回正确的 latest_pending_order_id
  - `GET /shop/status` 返回正确的营业状态
- [ ] 前端轮询行为在真机/模拟器上手动验证
- [ ] Commit: `test: verify polling endpoints for notification`

**验收标准：** 后端接口正确，前端手动验证轮询行为。

---

## 里程碑验收条件

- [ ] C端订单详情页：状态变化后自动更新 UI（5s 内）
- [ ] C端订单详情页：终态（completed/cancelled/rejected）后停止轮询
- [ ] C端订单详情页：状态变化时显示对应提示文案
- [ ] C端订单列表页：自动刷新（10s 内）
- [ ] C端菜单页/购物车页：打烊时禁用加购物车和结算（10s 内感知）
- [ ] S端订单处理页：新订单到达时播放铃声 + 顶部弹出提示（3s 内）
- [ ] S端订单处理页：页面不可见时降低轮询频率

---

## 轮询策略汇总（参照 design/06-api-notification.md 第5节）

| 端 | 页面 | 接口 | 间隔 | 停止条件 |
|----|------|------|------|---------|
| C端 | 订单详情 | GET /orders/{id}/status | 5 秒 | 终态 / 离页 |
| C端 | 订单列表 | GET /orders?status=pending,accepted,preparing,ready | 10 秒 | 离页 |
| C端 | 菜单页 | GET /shop/status | 10 秒 | 离页 |
| C端 | 购物车 | GET /shop/status | 10 秒 | 离页 |
| S端 | 订单处理 | GET /orders/pending | 3 秒 | 离页 |
