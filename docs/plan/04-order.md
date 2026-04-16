# M5: 订单模块 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 实现订单核心业务——下单（含幂等+限流）、状态流转、取消/拒绝、再来一单。C端可浏览→加购物车→下单→查看订单→再来一单，S端可接单→制作→完成。

**Architecture:** 订单创建需校验门店营业状态+商品上架状态，金额以后台价格为准，保存商品快照。状态机严格校验流转合法性。幂等键防重复提交，3秒限流防刷单。

**Tech Stack:** FastAPI, SQLAlchemy, asyncio (限流)

**前置依赖：** M1 + M2 + M3 + M4 完成

---

## 文件结构

```
backend/src/
├── utils/
│   └── order_no.py            # 订单号生成
│   └── rate_limit.py          # 限流工具
├── schemas/
│   └── order.py               # 订单请求/响应 schema
├── services/
│   └── order.py               # 订单业务逻辑（核心）
├── routers/
│   └── orders.py              # /api/v1/orders/...

frontend/src/
├── api/
│   └── order.ts               # 订单 API
├── stores/
│   └── cart.ts                # 购物车 store（已有，M3-C03）
├── pages/cart/
│   └── index.vue              # 购物车页面
├── pages/checkout/
│   └── index.vue              # 确认订单页
├── pages/order/
│   └── index.vue              # 订单列表页
├── pages/order-detail/
│   └── index.vue              # 订单详情页

admin/src/
├── api/
│   └── order.ts               # 订单 API
├── pages/orders/
│   └── index.vue              # 订单处理/列表页
```

---

## 后端任务

### M5-B01: 订单号生成工具

**Files:**
- Create: `backend/src/utils/order_no.py`

**Steps:**

- [ ] 创建 `backend/src/utils/order_no.py`（参照 design/01-database.md 4 节）：
  ```python
  import random
  from datetime import datetime, timezone

  async def generate_order_no(db: AsyncSession) -> str:
      """生成订单号: yyyyMMddHHmmss + 4位随机数字，冲突重试"""
      from src.models.order import Order
      from sqlalchemy import select

      for _ in range(10):
          timestamp = datetime.now(timezone.utc).strftime("%Y%m%d%H%M%S")
          random_suffix = str(random.randint(1000, 9999))
          order_no = f"{timestamp}{random_suffix}"

          result = await db.execute(select(Order).where(Order.order_no == order_no))
          if not result.scalar_one_or_none():
              return order_no

      raise RuntimeError("订单号生成失败，请重试")
  ```

- [ ] 测试：生成 10 个订单号，格式正确且无重复
- [ ] Commit: `feat: add order number generator`

**验收标准：** 格式 yyyyMMddHHmmss + 4位随机，冲突自动重试。

---

### M5-B02: 取餐码生成工具

**Files:**
- Create: `backend/src/utils/pickup_code.py`

**Steps:**

- [ ] 创建 `backend/src/utils/pickup_code.py`（参照 design/01-database.md 5 节）：
  ```python
  import random
  from datetime import date

  async def generate_pickup_code(db: AsyncSession, business_date: date) -> str:
      """生成4位随机取餐码，同营业日唯一，冲突重试"""
      from src.models.order import Order
      from sqlalchemy import select

      for _ in range(20):
          code = str(random.randint(1, 9999)).zfill(4)
          result = await db.execute(
              select(Order).where(
                  Order.business_date == business_date,
                  Order.pickup_code == code,
              )
          )
          if not result.scalar_one_or_none():
              return code

      raise RuntimeError("取餐码生成失败，请重试")
  ```

- [ ] Commit: `feat: add pickup code generator`

**验收标准：** 4 位数字，同营业日唯一。

---

### M5-B03: 创建订单 (POST /orders)

**Files:**
- Create: `backend/src/schemas/order.py`
- Create: `backend/src/services/order.py`
- Create: `backend/src/routers/orders.py`

**Steps:**

- [ ] 创建 `backend/src/schemas/order.py`（参照 design/04-api-order.md 2.1 节）：
  ```python
  class OrderItemCreate(BaseModel):
      product_id: str
      quantity: int = Field(..., gt=0)

  class OrderCreateRequest(BaseModel):
      items: list[OrderItemCreate] = Field(..., min_length=1)
      remark: str = Field("", max_length=200)

  class OrderItemResponse(BaseModel):
      product_name: str
      product_price: int
      product_price_display: str
      quantity: int
      subtotal: int
      subtotal_display: str

  class OrderCreateResponse(BaseModel):
      id: str
      order_no: str
      status: str
      total_amount: int
      total_amount_display: str
      remark: str
      pickup_code: str
      items: list[OrderItemResponse]
      created_at: datetime

  STATUS_LABELS = {
      "pending": "待处理",
      "accepted": "已接单",
      "preparing": "制作中",
      "ready": "待取餐",
      "completed": "已完成",
      "cancelled": "已取消",
      "rejected": "已拒绝",
  }
  ```

- [ ] 创建 `backend/src/services/order.py` 的 `create_order` 方法：
  1. 校验门店营业状态（shop.is_open）
  2. 幂等键检查：按 Idempotency-Key 查询已有订单
  3. 校验所有商品存在且上架
  4. 同一 product_id 不可重复出现
  5. 按后台价格计算 total_amount
  6. 创建订单 + 订单商品快照
  7. 生成订单号 + 取餐码

- [ ] 创建 `backend/src/routers/orders.py`：
  ```python
  router = APIRouter(prefix="/api/v1/orders", tags=["orders"])

  @router.post("", response_model=OrderCreateResponse, status_code=201)
  async def create_order(
      body: OrderCreateRequest,
      request: Request,
      current_user: dict = Depends(require_role("customer")),
      db: AsyncSession = Depends(get_db),
  ):
      idempotency_key = request.headers.get("Idempotency-Key")
      if not idempotency_key:
          raise HTTPException(status_code=400, detail="缺少 Idempotency-Key")
      # 调用 service...
  ```

- [ ] Commit: `feat: add order creation endpoint with idempotency`

**验收标准：** 下单成功返回订单详情，幂等键重复返回已有订单，门店打烊→400，商品下架→400。

---

### M5-B04: 订单列表 (GET /orders)

**Files:**
- Modify: `backend/src/routers/orders.py`
- Modify: `backend/src/services/order.py`

**Steps:**

- [ ] 实现 `GET /orders`（参照 design/04-api-order.md 2.2 节）：
  - customer：仅自己的订单
  - staff/admin：全量订单
  - 支持 status 筛选（逗号分隔）、start_date/end_date（A端）、分页
  - 返回字段：order_no_short, status_label, product_summary, user_nickname
  - product_summary 生成逻辑：取前 3 个商品名 + 数量拼接
- [ ] Commit: `feat: add order list endpoint`

**验收标准：** customer 仅看自己的，staff/admin 看全量，分页+筛选正常。

---

### M5-B05: 订单详情 (GET /orders/{id})

**Files:**
- Modify: `backend/src/routers/orders.py`
- Modify: `backend/src/services/order.py`

**Steps:**

- [ ] 实现 `GET /orders/{id}`（参照 design/04-api-order.md 2.3 节）：
  - customer 仅自己的订单，否则 403
  - staff/admin 可看全量
  - 包含商品快照 + 用户信息（非 customer）
  - 返回 order_no_short, status_label, pickup_code, estimated_wait_minutes
- [ ] Commit: `feat: add order detail endpoint`

**验收标准：** customer 访问他人订单 → 403。

---

### M5-B06: 订单状态轮询 (GET /orders/{id}/status)

**Files:**
- Modify: `backend/src/routers/orders.py`

**Steps:**

- [ ] 实现轻量状态轮询（参照 design/04-api-order.md 2.10 节）：
  ```python
  @router.get("/{order_id}/status")
  async def get_order_status(
      order_id: str,
      current_user: dict = Depends(require_role("customer")),
      db: AsyncSession = Depends(get_db),
  ):
      order = await get_order_for_user(db, order_id, current_user["user_id"])
      return {
          "status": order.status,
          "status_label": STATUS_LABELS[order.status],
          "estimated_wait_minutes": order.estimated_wait_minutes,
          "updated_at": order.updated_at,
      }
  ```

- [ ] Commit: `feat: add lightweight order status polling endpoint`

**验收标准：** 仅返回状态相关字段，轻量。

---

### M5-B07: 待处理订单 (GET /orders/pending)

**Files:**
- Modify: `backend/src/routers/orders.py`
- Modify: `backend/src/services/order.py`

**Steps:**

- [ ] 实现 `GET /orders/pending`（参照 design/04-api-order.md 2.11 节）：
  - staff only
  - 返回 pending_count, latest_pending_order_id, latest_pending_created_at, latest_orders
  - latest_orders 默认 5 条，最大 10 条
- [ ] Commit: `feat: add pending orders endpoint for S-end`

**验收标准：** 返回待处理订单摘要，用于 S 端轮询检测新订单。

---

### M5-B08: 取消订单 (PUT /orders/{id}/cancel)

**Files:**
- Modify: `backend/src/routers/orders.py`

**Steps:**

- [ ] 实现取消订单（参照 design/04-api-order.md 2.4 节）：
  - customer only，仅自己的订单
  - 仅 pending 状态可取消，否则 409
  - cancel_reason 选填
- [ ] Commit: `feat: add order cancel endpoint`

**验收标准：** pending → cancelled，非 pending → 409。

---

### M5-B09: 接单 (PUT /orders/{id}/accept)

**Files:**
- Modify: `backend/src/routers/orders.py`

**Steps:**

- [ ] 实现接单（参照 design/04-api-order.md 2.5 节）：
  - staff only
  - 仅 pending → accepted，否则 409
  - estimated_wait_minutes 选填
- [ ] Commit: `feat: add order accept endpoint`

**验收标准：** pending → accepted，非 pending → 409。

---

### M5-B10: 拒绝订单 (PUT /orders/{id}/reject)

**Files:**
- Modify: `backend/src/routers/orders.py`

**Steps:**

- [ ] 实现拒绝订单（参照 design/04-api-order.md 2.6 节）：
  - staff only
  - 仅 pending → rejected，否则 409
  - reject_reason 必填，为空 → 400
- [ ] Commit: `feat: add order reject endpoint`

**验收标准：** pending → rejected，原因为空 → 400，非 pending → 409。

---

### M5-B11: 状态推进 (prepare/ready/complete)

**Files:**
- Modify: `backend/src/routers/orders.py`
- Modify: `backend/src/services/order.py`

**Steps:**

- [ ] 实现三个状态推进端点（参照 design/04-api-order.md 2.7-2.9 节）：
  - `PUT /orders/{id}/prepare` — accepted → preparing (staff)
  - `PUT /orders/{id}/ready` — preparing → ready (staff)
  - `PUT /orders/{id}/complete` — ready → completed (staff)
  - 非法状态流转 → 409
- [ ] 在 service 中抽取通用的 `transition_order_status(db, order_id, from_status, to_status)` 方法
- [ ] Commit: `feat: add order status transition endpoints`

**验收标准：** 状态推进严格遵循状态机，非法流转 → 409。

---

### M5-B12: 再来一单 (POST /orders/{id}/reorder)

**Files:**
- Modify: `backend/src/routers/orders.py`
- Modify: `backend/src/services/order.py`

**Steps:**

- [ ] 实现再来一单（参照 design/04-api-order.md 2.11 节）：
  - customer only，仅自己的订单
  - 查询订单的 order_items → product_id 列表
  - 查询商品当前状态（是否在架）
  - product_id 为 NULL（商品已删除）归入 unavailable_items
  - 返回 available_items + unavailable_items + is_shop_open
- [ ] Commit: `feat: add reorder endpoint`

**验收标准：** 返回在架/已下架分类，门店状态。

---

### M5-B13: 订单状态机校验

**Files:**
- Modify: `backend/src/services/order.py`

**Steps:**

- [ ] 实现状态机校验方法：
  ```python
  VALID_TRANSITIONS = {
      "pending": {"accepted", "cancelled", "rejected"},
      "accepted": {"preparing"},
      "preparing": {"ready"},
      "ready": {"completed"},
  }

  def validate_status_transition(current: str, target: str) -> None:
      allowed = VALID_TRANSITIONS.get(current, set())
      if target not in allowed:
          raise ValueError(f"订单状态不允许从 {current} 变为 {target}")
  ```

- [ ] 所有状态变更端点调用此方法
- [ ] Commit: `feat: add order status machine validation`

**验收标准：** 非法状态变更统一返回 409。

---

### M5-B14: 限流工具

**Files:**
- Create: `backend/src/utils/rate_limit.py`

**Steps:**

- [ ] 实现简单内存限流：
  ```python
  from collections import defaultdict
  import time

  _user_last_order_time: dict[str, float] = defaultdict(float)
  RATE_LIMIT_SECONDS = 3

  def check_rate_limit(user_id: str) -> bool:
      """检查是否在限流期内。返回 True 表示被限流。"""
      now = time.time()
      last = _user_last_order_time.get(user_id, 0)
      if now - last < RATE_LIMIT_SECONDS:
          return True
      _user_last_order_time[user_id] = now
      return False
  ```

- [ ] 在 create_order 中集成：限流且幂等键不同 → 429
- [ ] Commit: `feat: add rate limiter for order creation`

**验收标准：** 同一用户 3 秒内重复下单（不同幂等键）→ 429。

---

### M5-B15: Order schemas 补全

**Files:**
- Modify: `backend/src/schemas/order.py`

**Steps:**

- [ ] 对照 design/04-api-order.md 检查所有字段：
  - OrderListResponse: order_no_short, status_label, product_summary, user_nickname
  - OrderDetailResponse: 完整信息 + user 对象（非 customer）
  - 所有金额字段同时返回 amount(int, 分) + amount_display(str, 元)
  - status_label 映射完整
- [ ] Commit: `fix: ensure order schemas match design doc`

**验收标准：** 所有字段与设计文档一致。

---

## C端前端任务

### M5-C01: 购物车页面

**Files:**
- Modify: `frontend/src/pages/cart/index.vue`

**Steps:**

- [ ] 实现购物车页面：
  - 商品列表（图片+名称+价格+数量+-按钮）
  - 左滑删除
  - 底部栏：合计金额 + "去结算"按钮
  - 空购物车状态
- [ ] 联动 cart store
- [ ] Commit: `feat: add cart page`

**验收标准：** 可修改数量、删除商品、显示合计、点击"去结算"跳转。

---

### M5-C02: 确认订单页

**Files:**
- Create: `frontend/src/api/order.ts`
- Modify: `frontend/src/pages/checkout/index.vue`

**Steps:**

- [ ] 创建 `frontend/src/api/order.ts`：`createOrder(items, remark, idempotencyKey)`
- [ ] 实现确认订单页：
  - 商品列表（只读）
  - 备注输入框（最大 200 字符）
  - 合计金额
  - "提交订单"按钮
  - 提交时生成 UUID 作为 Idempotency-Key
  - 防重复点击（提交中禁用按钮）
  - 门店打烊时显示提示，禁用提交
  - 成功后清空购物车，跳转订单详情
- [ ] Commit: `feat: add checkout page with idempotency`

**验收标准：** 下单成功跳转订单详情，重复提交不创建新订单。

---

### M5-C03: 订单列表页

**Files:**
- Modify: `frontend/src/pages/order/index.vue`

**Steps:**

- [ ] 实现订单列表页：
  - 顶部状态 tab：全部 / 进行中 / 已完成 / 已取消
  - 订单卡片：订单号后6位 + 状态标签 + 商品摘要 + 金额 + 时间
  - 下拉刷新 + 上拉加载更多
  - 点击订单跳转详情
  - "再来一单"按钮
- [ ] Commit: `feat: add order list page`

**验收标准：** 状态筛选正常，分页加载正常。

---

### M5-C04: 订单详情页

**Files:**
- Modify: `frontend/src/pages/order-detail/index.vue`

**Steps:**

- [ ] 实现订单详情页：
  - 状态进度条（pending → accepted → preparing → ready → completed）
  - 取餐码（大号显示）
  - 商品列表（名称+价格+数量+小计）
  - 备注
  - 合计金额
  - 取消按钮（仅 pending 状态）
  - "再来一单"按钮
  - 门店打烊/拒绝原因等提示
- [ ] Commit: `feat: add order detail page with status progress`

**验收标准：** 状态进度条正确，取餐码醒目，取消按钮仅 pending 可见。

---

### M5-C05: 再来一单流程

**Files:**
- Modify: `frontend/src/api/order.ts`
- Modify: `frontend/src/pages/order-detail/index.vue`

**Steps:**

- [ ] 添加 `reorder(orderId)` API
- [ ] 点击"再来一单"：
  - 调用 `POST /orders/{id}/reorder`
  - 在架商品自动加入购物车
  - 已下架商品弹窗提示
  - 门店打烊提示
  - 跳转购物车页
- [ ] Commit: `feat: add reorder flow`

**验收标准：** 在架商品加入购物车，下架商品有提示。

---

## S/A端前端任务

### M5-S01: 订单处理页 (待处理 + 接单/拒绝)

**Files:**
- Create: `admin/src/api/order.ts`
- Create: `admin/src/pages/orders/index.vue`

**Steps:**

- [ ] 创建 `admin/src/api/order.ts`：getOrders, acceptOrder, rejectOrder, prepareOrder, readyOrder, completeOrder
- [ ] 实现订单处理页：
  - 待处理订单列表（新订单置顶）
  - 每个订单卡片：订单号 + 用户 + 商品摘要 + 金额 + 时间
  - "接单"按钮（可填预估等待时间）
  - "拒绝"按钮（必须填原因）
  - 接单/拒绝后订单从列表消失
- [ ] Commit: `feat: add order processing page for S-end`

**验收标准：** 新订单高亮，接单/拒绝流程完整。

---

### M5-S02: 订单状态管理 (制作中→待取餐→已完成)

**Files:**
- Modify: `admin/src/pages/orders/index.vue`

**Steps:**

- [ ] 添加其他状态区域：
  - "制作中" tab：显示 accepted + preparing 订单，"开始制作"/"制作完成"按钮
  - "待取餐" tab：显示 ready 订单，"确认取餐"按钮
  - "已完成" tab：显示 completed/rejected/cancelled 订单
  - 按状态分区，操作按钮仅合法状态可见
- [ ] Commit: `feat: add order status management tabs`

**验收标准：** 按状态分区显示，操作按钮仅合法状态可见。

---

### M5-S03: 订单列表页 (A端查看)

**Files:**
- Modify: `admin/src/pages/orders/index.vue`

**Steps:**

- [ ] A端（admin 角色）：
  - 仅查看订单，不显示操作按钮
  - 支持状态筛选 + 日期筛选
  - 分页
- [ ] Commit: `feat: add admin order list view (read-only)`

**验收标准：** admin 仅查看，无操作按钮。

---

## 测试任务

### M5-T01: 创建订单测试

**Files:**
- Create: `backend/tests/test_order_create.py`

**Steps:**

- [ ] 测试用例：
  - `test_create_order_success` — 正常下单
  - `test_create_order_idempotency` — 同一幂等键返回已有订单
  - `test_create_order_rate_limit` — 3秒内不同幂等键 → 429
  - `test_create_order_shop_closed` — 打烊 → 400
  - `test_create_order_product_off_shelf` — 下架商品 → 400
  - `test_create_order_duplicate_product_id` — 同一 product_id 重复 → 400
  - `test_create_order_price_from_backend` — 金额以后台价格为准
  - `test_create_order_snapshot` — 商品快照保存正确
  - `test_create_order_pickup_code_unique` — 取餐码同日唯一
- [ ] Commit: `test: add order creation tests`

**验收标准：** 全部通过。

---

### M5-T02: 订单状态流转测试

**Files:**
- Create: `backend/tests/test_order_status.py`

**Steps:**

- [ ] 测试用例：
  - `test_accept_pending_order` — pending → accepted
  - `test_reject_pending_order` — pending → rejected
  - `test_cancel_pending_order` — pending → cancelled (customer)
  - `test_prepare_accepted_order` — accepted → preparing
  - `test_ready_preparing_order` — preparing → ready
  - `test_complete_ready_order` — ready → completed
  - `test_invalid_transition` — 所有非法流转 → 409
  - `test_cancel_accepted_order` — accepted 不能取消 → 409
  - `test_reorder_available` — 再来一单返回在架商品
  - `test_reorder_unavailable` — 已下架归入 unavailable
- [ ] Commit: `test: add order status transition tests`

**验收标准：** 全部通过。

---

### M5-T03: 订单权限测试

**Files:**
- Create: `backend/tests/test_order_permission.py`

**Steps:**

- [ ] 测试用例：
  - `test_customer_see_own_orders` — customer 仅自己的
  - `test_customer_access_other_order` — 访问他人 → 403
  - `test_staff_see_all_orders` — staff 看全量
  - `test_admin_see_all_orders` — admin 看全量
  - `test_customer_cannot_accept` — customer 接单 → 403
  - `test_admin_cannot_accept` — admin 接单 → 403 (A端仅查看)
  - `test_staff_can_operate` — staff 可接单/拒绝/推进
- [ ] Commit: `test: add order permission tests`

**验收标准：** 全部通过。

---

## 里程碑验收条件

- [ ] C端完整流程：浏览菜单 → 加购物车 → 下单 → 查看订单 → 再来一单
- [ ] S端完整流程：看到新订单 → 接单 → 开始制作 → 制作完成 → 确认取餐
- [ ] 门店打烊时无法下单
- [ ] 幂等键防重复提交
- [ ] 限流防刷单
- [ ] 订单金额以后台价格为准，商品快照正确
- [ ] A端仅查看订单，无操作按钮
