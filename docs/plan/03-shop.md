# M4: 门店模块 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 实现门店信息管理，营业状态手动切换，公告管理。C端可查看营业状态和公告，A端可编辑门店信息和开关营业状态。

**Architecture:** 单店模型，shop 表仅一条记录。营业状态由 is_open 字段控制，管理员手动切换。C端菜单页轮询营业状态。

**Tech Stack:** FastAPI, SQLAlchemy, Alembic (data migration)

**前置依赖：** M1 + M2 完成（可与 M3 并行）

---

## 文件结构

```
backend/src/
├── schemas/
│   └── shop.py                # 门店请求/响应 schema
├── services/
│   └── shop.py                # 门店业务逻辑
├── routers/
│   └── shop.py                # /api/v1/shop/...
└── alembic/versions/
    └── xxx_seed_shop.py       # 门店初始数据迁移

frontend/src/
├── api/
│   └── shop.ts                # 门店 API
├── components/
│   └── ShopStatus.vue         # 营业状态指示器
│   └── Announcement.vue       # 公告展示组件

admin/src/
├── api/
│   └── shop.ts
├── pages/shop/
│   └── index.vue              # 门店管理页
```

---

## 后端任务

### M4-B01: 门店初始数据 seed

**Files:**
- Create: `backend/alembic/versions/xxx_seed_shop.py`

**Steps:**

- [ ] 创建 Alembic 数据迁移：
  ```python
  """seed shop initial data

  Revision ID: seed_shop
  """
  from alembic import op
  import uuid

  def upgrade() -> None:
      op.execute(
          f"""
          INSERT INTO shop (id, name, address, phone, logo_url, business_hours,
                            announcement, announcement_enabled, is_open,
                            created_at, updated_at)
          VALUES ('{uuid.uuid4()}', 'Ideal Cafe', '请在管理后台设置门店地址',
                  '', '', '周一至周五 08:00-20:00', '', FALSE, TRUE,
                  NOW(), NOW())
          """
      )

  def downgrade() -> None:
      op.execute("DELETE FROM shop")
  ```

- [ ] 运行 `alembic upgrade head`
- [ ] 验证 shop 表有一条记录
- [ ] Commit: `feat: add shop initial data seed migration`

**验收标准：** `alembic upgrade head` 后 shop 表有一条初始记录。

---

### M4-B02: 获取门店信息 (GET /shop)

**Files:**
- Create: `backend/src/schemas/shop.py`
- Create: `backend/src/services/shop.py`
- Create: `backend/src/routers/shop.py`

**Steps:**

- [ ] 创建 `backend/src/schemas/shop.py`（参照 design/05-api-shop.md 2.1 节）：
  ```python
  class ShopResponse(BaseModel):
      id: str
      name: str
      address: str
      phone: str
      logo_url: str
      business_hours: str
      announcement: str
      announcement_enabled: bool
      is_open: bool
      created_at: datetime
      updated_at: datetime
  ```

- [ ] 创建 `backend/src/services/shop.py`：
  ```python
  async def get_shop(db: AsyncSession) -> Shop:
      result = await db.execute(select(Shop))
      shop = result.scalar_one_or_none()
      if not shop:
          raise ValueError("门店信息不存在")
      return shop
  ```

- [ ] 创建 `backend/src/routers/shop.py`：
  ```python
  router = APIRouter(prefix="/api/v1/shop", tags=["shop"])

  @router.get("", response_model=ShopResponse)
  async def get_shop_info(db: AsyncSession = Depends(get_db)):
      return await get_shop(db)
  ```

- [ ] 在 `main.py` 注册路由
- [ ] Commit: `feat: add GET /shop endpoint`

**验收标准：** 返回门店完整信息。

---

### M4-B03: 获取营业状态 (GET /shop/status)

**Files:**
- Modify: `backend/src/routers/shop.py`

**Steps:**

- [ ] 添加路由（参照 design/05-api-shop.md 2.2 节）：
  ```python
  class ShopStatusResponse(BaseModel):
      is_open: bool
      status_label: str
      announcement: str
      announcement_enabled: bool

  @router.get("/status", response_model=ShopStatusResponse)
  async def get_shop_status(db: AsyncSession = Depends(get_db)):
      shop = await get_shop(db)
      return ShopStatusResponse(
          is_open=shop.is_open,
          status_label="营业中" if shop.is_open else "已打烊",
          announcement=shop.announcement,
          announcement_enabled=shop.announcement_enabled,
      )
  ```

- [ ] Commit: `feat: add GET /shop/status lightweight endpoint`

**验收标准：** 返回营业状态 + 公告，轻量接口。

---

### M4-B04: 更新门店信息 (PUT /shop)

**Files:**
- Modify: `backend/src/routers/shop.py`
- Modify: `backend/src/schemas/shop.py`

**Steps:**

- [ ] 添加 schema 和路由（参照 design/05-api-shop.md 2.3 节）：
  - ShopUpdateRequest：name(必填), address(必填), phone, logo_url, business_hours, announcement, announcement_enabled
  - 校验规则：name 1-100, address 1-200, announcement 0-500
  - admin only
- [ ] Commit: `feat: add PUT /shop endpoint`

**验收标准：** admin 可编辑门店信息，字段校验正确。

---

### M4-B05: 切换营业状态 (PUT /shop/toggle-open)

**Files:**
- Modify: `backend/src/routers/shop.py`

**Steps:**

- [ ] 添加路由（参照 design/05-api-shop.md 2.4 节）：
  ```python
  class ToggleOpenRequest(BaseModel):
      is_open: bool

  class ToggleOpenResponse(BaseModel):
      is_open: bool
      status_label: str

  @router.put("/toggle-open", response_model=ToggleOpenResponse)
  async def toggle_open(
      body: ToggleOpenRequest,
      current_user: dict = Depends(require_role("admin")),
      db: AsyncSession = Depends(get_db),
  ):
      shop = await get_shop(db)
      shop.is_open = body.is_open
      return ToggleOpenResponse(
          is_open=shop.is_open,
          status_label="营业中" if shop.is_open else "已打烊",
      )
  ```

- [ ] Commit: `feat: add PUT /shop/toggle-open endpoint`

**验收标准：** 打烊后 is_open=false，C 端不可下单。

---

### M4-B06: 更新公告 (PUT /shop/announcement)

**Files:**
- Modify: `backend/src/routers/shop.py`

**Steps:**

- [ ] 添加路由（参照 design/05-api-shop.md 2.5 节）：
  - AnnouncementUpdateRequest：announcement(0-500), announcement_enabled(bool)
  - admin only
  - 返回更新后的公告
- [ ] Commit: `feat: add PUT /shop/announcement endpoint`

**验收标准：** 可单独更新公告内容和启用状态。

---

### M4-B07: Shop schemas 补全

**Files:**
- Modify: `backend/src/schemas/shop.py`

**Steps:**

- [ ] 对照 design/05-api-shop.md 检查所有字段
- [ ] 确保 ShopResponse 的 created_at/updated_at 使用 ISO 格式
- [ ] Commit: `fix: ensure shop schemas match design doc`

**验收标准：** 所有字段与设计文档一致。

---

## C端前端任务

### M4-C01: 营业状态指示器

**Files:**
- Create: `frontend/src/api/shop.ts`
- Create: `frontend/src/components/ShopStatus.vue`
- Modify: `frontend/src/pages/index/index.vue`

**Steps:**

- [ ] 创建 `frontend/src/api/shop.ts`：`getShopStatus()`
- [ ] 创建 `ShopStatus.vue` 组件：显示"营业中"(绿) / "已打烊"(灰)
- [ ] 在菜单主页顶部引入组件
- [ ] Commit: `feat: add shop status indicator on menu page`

**验收标准：** 菜单页顶部显示营业状态。

---

### M4-C02: 公告展示

**Files:**
- Create: `frontend/src/components/Announcement.vue`
- Modify: `frontend/src/pages/index/index.vue`

**Steps:**

- [ ] 创建 `Announcement.vue` 组件：公告条，启用时显示，禁用时隐藏
- [ ] 在菜单主页顶部（营业状态下方）引入
- [ ] Commit: `feat: add announcement display on menu page`

**验收标准：** 有公告时显示公告条，无公告或禁用时隐藏。

---

## S/A端前端任务

### M4-S01: 门店信息管理页

**Files:**
- Create: `admin/src/api/shop.ts`
- Create: `admin/src/pages/shop/index.vue`

**Steps:**

- [ ] 创建 `admin/src/api/shop.ts`：getShop, updateShop, toggleOpen, updateAnnouncement
- [ ] 实现门店管理页：表单编辑店名/地址/电话/Logo/营业时间
- [ ] 保存时调用 `PUT /shop`
- [ ] Logo 使用 FileUpload 组件上传
- [ ] Commit: `feat: add shop management page`

**验收标准：** 可编辑门店信息，保存后即时生效。

---

### M4-S02: 营业状态切换

**Files:**
- Modify: `admin/src/pages/shop/index.vue`

**Steps:**

- [ ] 添加营业状态开关：
  - 大按钮或 toggle："营业中" / "已打烊"
  - 打烊时弹出确认框：「打烊后顾客将无法下单，确认打烊？」
  - 调用 `PUT /shop/toggle-open`
- [ ] Commit: `feat: add shop toggle open with confirmation`

**验收标准：** 切换打烊时有二次确认，切换后 C 端即时感知。

---

### M4-S03: 公告编辑器

**Files:**
- Modify: `admin/src/pages/shop/index.vue`

**Steps:**

- [ ] 添加公告编辑区域：
  - 文本输入框（最大 500 字符）
  - 启用/禁用开关
  - 调用 `PUT /shop/announcement`
  - 字符计数
- [ ] Commit: `feat: add announcement editor`

**验收标准：** 可编辑公告内容和启用状态。

---

## 测试任务

### M4-T01: 门店 API 测试

**Files:**
- Create: `backend/tests/test_shop.py`

**Steps:**

- [ ] 测试用例：
  - `test_get_shop` — 返回门店信息
  - `test_get_shop_status` — 返回营业状态
  - `test_update_shop_admin` — admin 可编辑
  - `test_update_shop_customer_forbidden` — customer → 403
  - `test_toggle_open` — 切换营业状态
  - `test_update_announcement` — 更新公告
  - `test_status_label` — is_open=true → "营业中"，false → "已打烊"
- [ ] Commit: `test: add shop API tests`

**验收标准：** 全部通过。

---

## 里程碑验收条件

- [ ] A端可编辑门店信息（店名/地址/电话/Logo/营业时间）
- [ ] A端可切换营业状态，打烊时二次确认
- [ ] A端可编辑公告内容和启用状态
- [ ] C端菜单页显示营业状态和公告
- [ ] 打烊后 C 端无法下单（下单接口返回 400）
