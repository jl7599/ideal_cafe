# M3: 菜单模块 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 实现商品分类和商品的 CRUD，C端可浏览菜单，A端可管理商品（含图片上传和上下架）。

**Architecture:** 后端 Router → Service → Model 三层，分类和商品独立路由。C端仅返回启用/上架数据，A端返回全部。文件上传 MVP 存本地磁盘。

**Tech Stack:** FastAPI, SQLAlchemy, Pydantic, python-multipart (文件上传)

**前置依赖：** M1 + M2 完成

---

## 文件结构

```
backend/src/
├── services/
│   ├── category.py            # 分类业务逻辑
│   └── product.py             # 商品业务逻辑
├── schemas/
│   ├── category.py            # 分类请求/响应 schema
│   └── product.py             # 商品请求/响应 schema
├── routers/
│   ├── categories.py          # /api/v1/categories/...
│   ├── products.py            # /api/v1/products/...
│   └── upload.py              # /api/v1/upload, /api/v1/static/...

frontend/src/
├── api/
│   ├── category.ts            # 分类 API
│   └── product.ts             # 商品 API
├── pages/index/
│   └── index.vue              # 菜单主页
├── pages/product-detail/
│   └── index.vue              # 商品详情页
└── stores/
    └── cart.ts                # 购物车 store

admin/src/
├── api/
│   ├── category.ts
│   └── product.ts
├── pages/products/
│   └── index.vue              # 商品管理页
├── components/
│   └── FileUpload.vue         # 文件上传组件
```

---

## 后端任务

### M3-B01: 分类 CRUD

**Files:**
- Create: `backend/src/schemas/category.py`
- Create: `backend/src/services/category.py`
- Create: `backend/src/routers/categories.py`

**Steps:**

- [ ] 创建 `backend/src/schemas/category.py`（参照 design/03-api-menu.md 2.1-2.4 节）：
  ```python
  class CategoryResponse(BaseModel):
      id: str
      name: str
      icon: str
      sort_order: int
      is_enabled: bool

  class CategoryListResponse(BaseModel):
      items: list[CategoryResponse]
      total: int

  class CategoryCreateRequest(BaseModel):
      name: str = Field(..., min_length=1, max_length=50)
      icon: str = ""
      sort_order: int | None = None
      is_enabled: bool = True

  class CategoryUpdateRequest(BaseModel):
      name: str | None = Field(None, min_length=1, max_length=50)
      icon: str | None = None
      sort_order: int | None = None
      is_enabled: bool | None = None
  ```

- [ ] 创建 `backend/src/services/category.py`：
  - `get_categories(db, all=False)` — all=False 仅返回 is_enabled=True
  - `create_category(db, data)` — sort_order 未传时自动 max+1
  - `update_category(db, category_id, data)` — 部分更新
  - `delete_category(db, category_id)` — 有商品时拒绝删除（409）

- [ ] 创建 `backend/src/routers/categories.py`：
  - `GET /api/v1/categories` — customer 看启用，admin+all=true 看全部
  - `POST /api/v1/categories` — admin only
  - `PUT /api/v1/categories/{id}` — admin only
  - `DELETE /api/v1/categories/{id}` — admin only，有商品 → 409

- [ ] 在 `main.py` 注册路由
- [ ] Commit: `feat: add category CRUD endpoints`

**验收标准：** C端仅返回启用分类，A端返回全部，有商品时禁止删除分类。

---

### M3-B02: 商品 CRUD

**Files:**
- Create: `backend/src/schemas/product.py`
- Create: `backend/src/services/product.py`
- Create: `backend/src/routers/products.py`

**Steps:**

- [ ] 创建 `backend/src/schemas/product.py`（参照 design/03-api-menu.md 2.5-2.8 节）：
  ```python
  class ProductResponse(BaseModel):
      id: str
      category_id: str
      category_name: str
      name: str
      subtitle: str
      description: str
      price: int
      price_display: str  # "28.00"
      image_url: str
      sort_order: int
      is_on_shelf: bool

  class ProductListResponse(BaseModel):
      items: list[ProductResponse]
      total: int
      page: int
      page_size: int
  ```
  - price_display 通过 `@computed_field` 或 validator 实现：`f"{price / 100:.2f}"`

- [ ] 创建 `backend/src/services/product.py`：
  - `get_products(db, category_id=None, all=False, page=1, page_size=20)` — 含分页 + category_name 冗余
  - `get_product(db, product_id, all=False)` — customer 看下架 → 404
  - `create_product(db, data)` — sort_order 自动计算
  - `update_product(db, product_id, data)`
  - `delete_product(db, product_id)` — 物理删除

- [ ] 创建 `backend/src/routers/products.py`：
  - `GET /api/v1/products` — customer 看上架，admin+all=true 看全部
  - `GET /api/v1/products/{id}` — customer 访问下架 → 404
  - `POST /api/v1/products` — admin only
  - `PUT /api/v1/products/{id}` — admin only
  - `DELETE /api/v1/products/{id}` — admin only

- [ ] Commit: `feat: add product CRUD endpoints`

**验收标准：** C端仅返回上架商品，A端返回全部，分页正常，price_display 格式正确。

---

### M3-B03: 商品上下架

**Files:**
- Modify: `backend/src/routers/products.py`

**Steps:**

- [ ] 添加路由：
  ```python
  @router.put("/{product_id}/shelf")
  async def toggle_shelf(
      product_id: str,
      body: ShelfToggleRequest,
      current_user: dict = Depends(require_role("admin")),
      db: AsyncSession = Depends(get_db),
  ):
      product = await get_product(db, product_id, all=True)
      if not product:
          raise HTTPException(status_code=404, detail="商品不存在")
      product.is_on_shelf = body.is_on_shelf
      return {"id": str(product.id), "is_on_shelf": product.is_on_shelf}
  ```

- [ ] Commit: `feat: add product shelf toggle endpoint`

**验收标准：** 切换 is_on_shelf 后 C 端立即可见/不可见。

---

### M3-B04: 文件上传

**Files:**
- Create: `backend/src/routers/upload.py`

**Steps:**

- [ ] 创建 `backend/src/routers/upload.py`（参照 design/03-api-menu.md 3.1 节）：
  ```python
  import uuid
  import os
  from fastapi import APIRouter, Depends, HTTPException, UploadFile, File
  from src.dependencies import require_role

  router = APIRouter(prefix="/api/v1", tags=["upload"])

  ALLOWED_TYPES = {"image/jpeg", "image/png", "image/webp"}
  MAX_SIZE = 5 * 1024 * 1024  # 5MB

  @router.post("/upload")
  async def upload_file(
      file: UploadFile = File(...),
      current_user: dict = Depends(require_role("admin")),
  ):
      if file.content_type not in ALLOWED_TYPES:
          raise HTTPException(status_code=400, detail="不支持的文件类型")
      content = await file.read()
      if len(content) > MAX_SIZE:
          raise HTTPException(status_code=400, detail="文件过大，最大 5MB")

      ext = file.filename.rsplit(".", 1)[-1] if file.filename and "." in file.filename else "jpg"
      filename = f"{uuid.uuid4().hex}.{ext}"
      upload_dir = os.getenv("UPLOAD_DIR", "uploads")
      os.makedirs(upload_dir, exist_ok=True)

      filepath = os.path.join(upload_dir, filename)
      with open(filepath, "wb") as f:
          f.write(content)

      return {"url": f"/api/v1/static/{filename}"}
  ```

- [ ] Commit: `feat: add file upload endpoint`

**验收标准：** 仅允许 jpeg/png/webp，最大 5MB，返回 URL 路径。

---

### M3-B05: 静态文件服务

**Files:**
- Modify: `backend/src/main.py`

**Steps:**

- [ ] 在 `main.py` 挂载静态文件目录：
  ```python
  from fastapi.staticfiles import StaticFiles
  import os

  upload_dir = os.getenv("UPLOAD_DIR", "uploads")
  os.makedirs(upload_dir, exist_ok=True)
  app.mount("/api/v1/static", StaticFiles(directory=upload_dir), name="static")
  ```

- [ ] 验证上传图片后可通过 URL 访问
- [ ] Commit: `feat: add static file serving for uploads`

**验收标准：** `/api/v1/static/xxx.jpg` 可访问上传的图片。

---

### M3-B06: 菜单 schemas 补全

**Files:**
- Modify: `backend/src/schemas/category.py`
- Modify: `backend/src/schemas/product.py`

**Steps:**

- [ ] 对照 design/03-api-menu.md 检查所有字段
- [ ] 确保 ProductResponse 的 category_name 通过 join 查询获取
- [ ] 确保 price_display 计算逻辑正确（分 → 元）
- [ ] Commit: `fix: ensure menu schemas match design doc`

**验收标准：** 所有字段与设计文档一致。

---

### M3-B07: 分类有商品时禁止删除

**Files:**
- Modify: `backend/src/services/category.py`

**Steps:**

- [ ] 在 delete_category 中检查：
  ```python
  result = await db.execute(
      select(Product).where(Product.category_id == category_id).limit(1)
  )
  if result.scalar_one_or_none():
      raise ValueError("分类下有商品，不可删除")
  ```

- [ ] 测试：有商品的分类 → 409，无商品的分类 → 200
- [ ] Commit: `feat: prevent category deletion when products exist`

**验收标准：** 有商品的分类返回 409。

---

## C端前端任务

### M3-C01: 菜单页

**Files:**
- Create: `frontend/src/api/category.ts`
- Create: `frontend/src/api/product.ts`
- Modify: `frontend/src/pages/index/index.vue`

**Steps:**

- [ ] 创建 `frontend/src/api/category.ts`：`getCategories()`
- [ ] 创建 `frontend/src/api/product.ts`：`getProducts(categoryId?)`
- [ ] 实现菜单页：左侧分类列表 + 右侧商品网格
  - 点击分类切换商品列表
  - 商品卡片显示：图片、名称、副标题、价格
  - 点击商品跳转详情页
  - "加入购物车"按钮（需登录）
- [ ] Commit: `feat: add menu page with category sidebar`

**验收标准：** 分类切换正常，商品展示完整，点击可跳转详情。

---

### M3-C02: 商品详情页

**Files:**
- Modify: `frontend/src/pages/product-detail/index.vue`

**Steps:**

- [ ] 实现商品详情页：图片 + 名称 + 副标题 + 描述 + 价格
- [ ] "加入购物车"按钮，选择数量
- [ ] Commit: `feat: add product detail page`

**验收标准：** 显示完整商品信息，可加入购物车。

---

### M3-C03: 购物车 store

**Files:**
- Create: `frontend/src/stores/cart.ts`

**Steps:**

- [ ] 创建 `frontend/src/stores/cart.ts`：
  ```typescript
  import { defineStore } from 'pinia'

  interface CartItem {
    productId: string
    name: string
    price: number
    imageUrl: string
    quantity: number
  }

  export const useCartStore = defineStore('cart', {
    state: () => ({
      items: JSON.parse(uni.getStorageSync('cart_items') || '[]') as CartItem[],
    }),
    getters: {
      totalQuantity: (state) => state.items.reduce((sum, item) => sum + item.quantity, 0),
      totalAmount: (state) => state.items.reduce((sum, item) => sum + item.price * item.quantity, 0),
    },
    actions: {
      addItem(item: Omit<CartItem, 'quantity'>) {
        const existing = this.items.find((i) => i.productId === item.productId)
        if (existing) {
          existing.quantity++
        } else {
          this.items.push({ ...item, quantity: 1 })
        }
        this.save()
      },
      updateQuantity(productId: string, quantity: number) {
        const item = this.items.find((i) => i.productId === productId)
        if (item) {
          item.quantity = quantity
          if (quantity <= 0) this.removeItem(productId)
        }
        this.save()
      },
      removeItem(productId: string) {
        this.items = this.items.filter((i) => i.productId !== productId)
        this.save()
      },
      clear() {
        this.items = []
        this.save()
      },
      save() {
        uni.setStorageSync('cart_items', JSON.stringify(this.items))
      },
    },
  })
  ```

- [ ] Commit: `feat: add cart store with local persistence`

**验收标准：** 加入购物车、修改数量、清空、刷新后保持。

---

## S/A端前端任务

### M3-S01: 分类管理页

**Files:**
- Create: `admin/src/api/category.ts`
- Create: `admin/src/pages/categories/index.vue`（或复用 products 页面的分类 tab）

**Steps:**

- [ ] 实现分类列表：名称、图标、排序、启用状态
- [ ] 新增分类弹窗/表单
- [ ] 编辑分类
- [ ] 删除分类（有商品时禁用按钮 + 提示）
- [ ] 拖拽/输入排序
- [ ] Commit: `feat: add category management page`

**验收标准：** CRUD 正常，有商品时禁用删除。

---

### M3-S02: 商品管理页

**Files:**
- Create: `admin/src/api/product.ts`
- Create: `admin/src/pages/products/index.vue`

**Steps:**

- [ ] 实现商品列表：按分类筛选，分页，显示上下架状态
- [ ] 新增商品表单：分类选择、名称、副标题、描述、价格、图片、排序
- [ ] 编辑商品
- [ ] 删除商品
- [ ] 上下架切换
- [ ] Commit: `feat: add product management page`

**验收标准：** CRUD + 上下架正常，图片上传可用。

---

### M3-S03: 文件上传组件

**Files:**
- Create: `admin/src/components/FileUpload.vue`

**Steps:**

- [ ] 实现文件上传组件：
  - 选择图片 → 预览 → 上传 → 返回 URL
  - 限制文件类型：jpeg/png/webp
  - 限制大小：5MB
  - 上传中显示 loading
- [ ] Commit: `feat: add file upload component`

**验收标准：** 选择图片后上传成功，返回 URL 可在表单中使用。

---

## 测试任务

### M3-T01: 分类 API 测试

**Files:**
- Create: `backend/tests/test_categories.py`

**Steps:**

- [ ] 测试用例：
  - `test_get_categories_customer` — 仅返回启用分类
  - `test_get_categories_admin_all` — 返回全部分类
  - `test_create_category` — 创建成功
  - `test_update_category` — 部分更新
  - `test_delete_category_no_products` — 无商品时删除成功
  - `test_delete_category_with_products` — 有商品 → 409
  - `test_create_category_auto_sort` — sort_order 自动计算
- [ ] Commit: `test: add category API tests`

**验收标准：** 全部通过。

---

### M3-T02: 商品 API 测试

**Files:**
- Create: `backend/tests/test_products.py`

**Steps:**

- [ ] 测试用例：
  - `test_get_products_customer` — 仅返回上架商品
  - `test_get_products_admin_all` — 返回全部
  - `test_get_products_pagination` — 分页正常
  - `test_get_products_filter_category` — 按分类筛选
  - `test_get_product_customer_off_shelf` — 下架 → 404
  - `test_create_product` — 创建成功
  - `test_update_product` — 更新成功
  - `test_delete_product` — 删除成功
  - `test_toggle_shelf` — 上下架切换
  - `test_price_display` — price_display 格式 "28.00"
- [ ] Commit: `test: add product API tests`

**验收标准：** 全部通过。

---

### M3-T03: 上传 API 测试

**Files:**
- Create: `backend/tests/test_upload.py`

**Steps:**

- [ ] 测试用例：
  - `test_upload_image_success` — jpeg 上传成功
  - `test_upload_invalid_type` — 非图片 → 400
  - `test_upload_too_large` — 超 5MB → 400
- [ ] Commit: `test: add upload API tests`

**验收标准：** 全部通过。

---

## 里程碑验收条件

- [ ] C端可浏览菜单（分类切换 + 商品列表 + 商品详情）
- [ ] A端可管理分类（CRUD + 有商品不可删）
- [ ] A端可管理商品（CRUD + 上下架 + 图片上传）
- [ ] C端仅看到启用分类和上架商品
- [ ] 图片上传后可通过 URL 访问
