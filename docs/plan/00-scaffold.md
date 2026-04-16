# M1: 项目脚手架 & 数据库 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 搭建后端 FastAPI 项目骨架、数据库全部模型和迁移、两个前端项目骨架、开发环境 Docker Compose，让所有后续里程碑可以在其上开发。

**Architecture:** 后端 FastAPI + SQLAlchemy 2.0 async + PostgreSQL + Alembic；C端 uni-app Vue3 → 微信小程序；S/A端 uni-app Vue3 → H5；开发环境 Docker Compose 一键启动。

**Tech Stack:** Python 3.12+, FastAPI, SQLAlchemy 2.0, Alembic, Pydantic v2, uv, PostgreSQL, uni-app, Vue3, Pinia, pnpm, Docker

---

## 文件结构

```
backend/
├── src/
│   ├── __init__.py
│   ├── main.py                # FastAPI 应用入口
│   ├── config.py              # 配置管理（pydantic-settings）
│   ├── database.py            # 异步引擎 + 会话工厂
│   ├── dependencies.py        # 通用依赖（get_db）
│   ├── models/
│   │   ├── __init__.py        # 导出所有模型
│   │   ├── user.py            # users 表
│   │   ├── staff.py           # staff 表
│   │   ├── category.py        # categories 表
│   │   ├── product.py         # products 表
│   │   ├── shop.py            # shop 表
│   │   ├── order.py           # orders + order_items 表
│   │   └── token_blacklist.py # token_blacklist 表
│   ├── schemas/
│   │   └── __init__.py
│   ├── routers/
│   │   └── __init__.py
│   ├── services/
│   │   └── __init__.py
│   └── utils/
│       └── __init__.py
├── alembic/
│   ├── env.py
│   └── versions/
├── alembic.ini
├── tests/
│   ├── __init__.py
│   └── conftest.py            # 测试 DB fixture
├── pyproject.toml
└── .env.example

frontend/                       # C端
├── src/
│   ├── api/
│   │   └── request.ts         # HTTP 请求封装
│   ├── stores/
│   │   └── index.ts
│   ├── pages/
│   │   ├── index/
│   │   │   └── index.vue      # 菜单主页（占位）
│   │   ├── order/
│   │   │   └── index.vue      # 订单列表（占位）
│   │   └── mine/
│   │       └── index.vue      # 我的（占位）
│   ├── utils/
│   └── static/
├── pages.json
├── manifest.json
├── package.json
└── tsconfig.json

admin/                          # S/A端
├── src/
│   ├── api/
│   │   └── request.ts
│   ├── stores/
│   │   └── index.ts
│   ├── pages/
│   │   └── login/
│   │       └── index.vue      # 登录页（占位）
│   ├── utils/
│   └── static/
├── pages.json
├── manifest.json
├── package.json
└── tsconfig.json

scripts/
└── init_admin.py

docker-compose.yml
Dockerfile.backend
```

---

## 后端任务

### M1-B01: 初始化 FastAPI 项目

**Files:**
- Create: `backend/pyproject.toml`
- Create: `backend/src/__init__.py`
- Create: `backend/src/main.py`

**Steps:**

- [ ] 使用 `uv init backend` 创建项目，修改 pyproject.toml 添加依赖：
  ```toml
  [project]
  name = "ideal-cafe-backend"
  version = "0.1.0"
  requires-python = ">=3.12"
  dependencies = [
      "fastapi>=0.115.0",
      "uvicorn[standard]>=0.30.0",
      "sqlalchemy[asyncio]>=2.0.0",
      "asyncpg>=0.30.0",
      "alembic>=1.14.0",
      "pydantic>=2.0.0",
      "pydantic-settings>=2.0.0",
      "python-jose[cryptography]>=3.3.0",
      "passlib[bcrypt]>=1.7.4",
      "python-multipart>=0.0.9",
      "httpx>=0.27.0",
  ]

  [tool.ruff]
  line-length = 88
  ```

- [ ] 创建 `backend/src/main.py`：
  ```python
  from fastapi import FastAPI
  from fastapi.middleware.cors import CORSMiddleware

  app = FastAPI(title="Ideal Cafe", version="0.1.0")

  app.add_middleware(
      CORSMiddleware,
      allow_origins=["*"],
      allow_credentials=True,
      allow_methods=["*"],
      allow_headers=["*"],
  )

  @app.get("/health")
  async def health_check():
      return {"status": "ok"}
  ```

- [ ] 运行 `cd backend && uv sync` 安装依赖
- [ ] 运行 `uv run uvicorn src.main:app --reload`，访问 http://localhost:8000/health 返回 `{"status": "ok"}`
- [ ] 访问 http://localhost:8000/docs 看到 Swagger UI
- [ ] Commit: `feat: init FastAPI project with uv`

**验收标准：** `uv run uvicorn src.main:app --reload` 可启动，/health 和 /docs 正常。

---

### M1-B02: 配置管理

**Files:**
- Create: `backend/src/config.py`
- Create: `backend/.env.example`

**Steps:**

- [ ] 创建 `backend/src/config.py`：
  ```python
  from pydantic_settings import BaseSettings

  class Settings(BaseSettings):
      DATABASE_URL: str = "postgresql+asyncpg://postgres:postgres@localhost:5432/ideal_cafe"
      SECRET_KEY: str = "dev-secret-key-change-in-production"
      WECHAT_APP_ID: str = ""
      WECHAT_APP_SECRET: str = ""
      ACCESS_TOKEN_EXPIRE_MINUTES: int = 120
      REFRESH_TOKEN_EXPIRE_DAYS: int = 30

      model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

  settings = Settings()
  ```

- [ ] 创建 `backend/.env.example`：
  ```env
  DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/ideal_cafe
  SECRET_KEY=dev-secret-key-change-in-production
  WECHAT_APP_ID=
  WECHAT_APP_SECRET=
  ACCESS_TOKEN_EXPIRE_MINUTES=120
  REFRESH_TOKEN_EXPIRE_DAYS=30
  ```

- [ ] 验证 `uv run python -c "from src.config import settings; print(settings.DATABASE_URL)"` 输出正确
- [ ] Commit: `feat: add config management with pydantic-settings`

**验收标准：** 环境变量从 .env 读取，缺失 WECHAT_APP_ID 不报错（默认空字符串）。

---

### M1-B03: SQLAlchemy async 引擎 + 会话管理

**Files:**
- Create: `backend/src/database.py`

**Steps:**

- [ ] 创建 `backend/src/database.py`：
  ```python
  from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine
  from sqlalchemy.orm import DeclarativeBase

  from src.config import settings

  engine = create_async_engine(settings.DATABASE_URL, echo=False)
  async_session = async_sessionmaker(engine, expire_on_commit=False)

  class Base(DeclarativeBase):
      pass
  ```

- [ ] 验证 import 无报错
- [ ] Commit: `feat: add SQLAlchemy async engine and session`

**验收标准：** `from src.database import engine, async_session, Base` 无报错。

---

### M1-B04: 创建全部 ORM 模型

**Files:**
- Create: `backend/src/models/__init__.py`
- Create: `backend/src/models/user.py`
- Create: `backend/src/models/staff.py`
- Create: `backend/src/models/category.py`
- Create: `backend/src/models/product.py`
- Create: `backend/src/models/shop.py`
- Create: `backend/src/models/order.py`
- Create: `backend/src/models/token_blacklist.py`

**Steps:**

- [ ] 创建 `backend/src/models/user.py`（参照 design/01-database.md 2.1 节）：
  ```python
  import uuid
  from datetime import datetime

  from sqlalchemy import String, Index
  from sqlalchemy.orm import Mapped, mapped_column

  from src.database import Base

  class User(Base):
      __tablename__ = "users"

      id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
      openid: Mapped[str] = mapped_column(String(64), unique=True, nullable=False)
      unionid: Mapped[str | None] = mapped_column(String(64), nullable=True)
      session_key: Mapped[str | None] = mapped_column(String(128), nullable=True)
      nickname: Mapped[str] = mapped_column(String(50), nullable=False)
      avatar_url: Mapped[str] = mapped_column(String(500), nullable=False, default="")
      created_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now)
      updated_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now, onupdate=datetime.now)

      __table_args__ = (
          Index("idx_users_openid", "openid"),
          Index("idx_users_unionid", "unionid", postgresql_where="unionid IS NOT NULL"),
      )
  ```

- [ ] 创建 `backend/src/models/staff.py`（参照 design/01-database.md 2.2 节）：
  ```python
  import uuid
  from datetime import datetime

  from sqlalchemy import Boolean, String, CheckConstraint, Index
  from sqlalchemy.orm import Mapped, mapped_column

  from src.database import Base

  class Staff(Base):
      __tablename__ = "staff"

      id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
      phone: Mapped[str] = mapped_column(String(11), unique=True, nullable=False)
      password_hash: Mapped[str] = mapped_column(String(128), nullable=False)
      role: Mapped[str] = mapped_column(String(10), nullable=False)
      is_active: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True)
      created_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now)
      updated_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now, onupdate=datetime.now)

      __table_args__ = (
          CheckConstraint("role IN ('staff', 'admin')", name="ck_staff_role"),
          Index("idx_staff_phone", "phone"),
      )
  ```

- [ ] 创建 `backend/src/models/category.py`（参照 design/01-database.md 2.3 节）：
  ```python
  import uuid
  from datetime import datetime

  from sqlalchemy import Boolean, Integer, String
  from sqlalchemy.orm import Mapped, mapped_column

  from src.database import Base

  class Category(Base):
      __tablename__ = "categories"

      id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
      name: Mapped[str] = mapped_column(String(50), nullable=False)
      icon: Mapped[str] = mapped_column(String(500), nullable=False, default="")
      sort_order: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
      is_enabled: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True)
      created_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now)
      updated_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now, onupdate=datetime.now)
  ```

- [ ] 创建 `backend/src/models/product.py`（参照 design/01-database.md 2.4 节）：
  ```python
  import uuid
  from datetime import datetime

  from sqlalchemy import Boolean, ForeignKey, Integer, String, Index
  from sqlalchemy.orm import Mapped, mapped_column

  from src.database import Base

  class Product(Base):
      __tablename__ = "products"

      id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
      category_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("categories.id"), nullable=False)
      name: Mapped[str] = mapped_column(String(100), nullable=False)
      subtitle: Mapped[str] = mapped_column(String(200), nullable=False, default="")
      description: Mapped[str] = mapped_column(nullable=False, default="")
      price: Mapped[int] = mapped_column(Integer, nullable=False)
      image_url: Mapped[str] = mapped_column(String(500), nullable=False)
      sort_order: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
      is_on_shelf: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True)
      created_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now)
      updated_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now, onupdate=datetime.now)

      __table_args__ = (
          Index("idx_products_category", "category_id"),
          Index("idx_products_shelf_sort", "is_on_shelf", "sort_order"),
      )
  ```

- [ ] 创建 `backend/src/models/shop.py`（参照 design/01-database.md 2.5 节）：
  ```python
  import uuid
  from datetime import datetime

  from sqlalchemy import Boolean, String, Text
  from sqlalchemy.orm import Mapped, mapped_column

  from src.database import Base

  class Shop(Base):
      __tablename__ = "shop"

      id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
      name: Mapped[str] = mapped_column(String(100), nullable=False)
      address: Mapped[str] = mapped_column(String(200), nullable=False)
      phone: Mapped[str] = mapped_column(String(20), nullable=False, default="")
      logo_url: Mapped[str] = mapped_column(String(500), nullable=False, default="")
      business_hours: Mapped[str] = mapped_column(String(200), nullable=False, default="")
      announcement: Mapped[str] = mapped_column(Text, nullable=False, default="")
      announcement_enabled: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False)
      is_open: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True)
      created_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now)
      updated_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now, onupdate=datetime.now)
  ```

- [ ] 创建 `backend/src/models/order.py`（参照 design/01-database.md 2.6 + 2.7 节）：
  ```python
  import uuid
  from datetime import date, datetime

  from sqlalchemy import Date, ForeignKey, Integer, String, UniqueConstraint, Index
  from sqlalchemy.orm import Mapped, mapped_column

  from src.database import Base

  class Order(Base):
      __tablename__ = "orders"

      id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
      order_no: Mapped[str] = mapped_column(String(18), unique=True, nullable=False)
      user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), nullable=False)
      status: Mapped[str] = mapped_column(String(15), nullable=False, default="pending")
      idempotency_key: Mapped[uuid.UUID] = mapped_column(unique=True, nullable=False)
      total_amount: Mapped[int] = mapped_column(Integer, nullable=False)
      remark: Mapped[str] = mapped_column(String(200), nullable=False, default="")
      business_date: Mapped[date] = mapped_column(Date, nullable=False)
      pickup_code: Mapped[str] = mapped_column(String(4), nullable=False)
      cancel_reason: Mapped[str] = mapped_column(String(200), nullable=False, default="")
      reject_reason: Mapped[str] = mapped_column(String(200), nullable=False, default="")
      estimated_wait_minutes: Mapped[int | None] = mapped_column(Integer, nullable=True)
      created_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now)
      updated_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now, onupdate=datetime.now)

      __table_args__ = (
          Index("idx_orders_order_no", "order_no"),
          UniqueConstraint("business_date", "pickup_code", name="uq_orders_business_date_pickup_code"),
          Index("idx_orders_user_status", "user_id", "status"),
          Index("idx_orders_status_created", "status", "created_at"),
      )

  class OrderItem(Base):
      __tablename__ = "order_items"

      id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
      order_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("orders.id"), nullable=False)
      product_id: Mapped[uuid.UUID | None] = mapped_column(
          ForeignKey("products.id", ondelete="SET NULL"), nullable=True
      )
      product_name: Mapped[str] = mapped_column(String(100), nullable=False)
      product_price: Mapped[int] = mapped_column(Integer, nullable=False)
      quantity: Mapped[int] = mapped_column(Integer, nullable=False)
      subtotal: Mapped[int] = mapped_column(Integer, nullable=False)

      __table_args__ = (
          Index("idx_order_items_order", "order_id"),
          Index("idx_order_items_product", "product_id", postgresql_where="product_id IS NOT NULL"),
      )
  ```

- [ ] 创建 `backend/src/models/token_blacklist.py`（参照 design/01-database.md 2.8 节）：
  ```python
  import uuid
  from datetime import datetime

  from sqlalchemy import String, Index
  from sqlalchemy.orm import Mapped, mapped_column

  from src.database import Base

  class TokenBlacklist(Base):
      __tablename__ = "token_blacklist"

      id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
      token_jti: Mapped[str] = mapped_column(String(64), unique=True, nullable=False)
      expires_at: Mapped[datetime] = mapped_column(nullable=False)
      created_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now)

      __table_args__ = (
          Index("idx_token_blacklist_jti", "token_jti"),
      )
  ```

- [ ] 创建 `backend/src/models/__init__.py` 导出所有模型：
  ```python
  from src.models.user import User
  from src.models.staff import Staff
  from src.models.category import Category
  from src.models.product import Product
  from src.models.shop import Shop
  from src.models.order import Order, OrderItem
  from src.models.token_blacklist import TokenBlacklist

  __all__ = [
      "User", "Staff", "Category", "Product",
      "Shop", "Order", "OrderItem", "TokenBlacklist",
  ]
  ```

- [ ] 验证 `uv run python -c "from src.models import *"` 无报错
- [ ] Commit: `feat: add all SQLAlchemy ORM models`

**验收标准：** 8 个模型类全部可 import，字段/索引/约束与 design/01-database.md 一致。

---

### M1-B05: Alembic 初始化 + 初始迁移

**Files:**
- Create: `backend/alembic.ini`
- Create: `backend/alembic/env.py`
- Create: `backend/alembic/versions/` (auto-generated)

**Steps:**

- [ ] 在 backend 目录下运行 `uv run alembic init alembic`
- [ ] 修改 `backend/alembic.ini` 中 `sqlalchemy.url` 为空（从 config 读取）
- [ ] 修改 `backend/alembic/env.py`：
  ```python
  import asyncio
  from logging.config import fileConfig

  from sqlalchemy import pool
  from sqlalchemy.ext.asyncio import async_engine_from_config

  from alembic import context
  from src.config import settings
  from src.database import Base
  from src.models import *  # noqa: F401, F403 — ensure all models registered

  config = context.config
  config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)

  if config.config_file_name is not None:
      fileConfig(config.config_file_name)

  target_metadata = Base.metadata

  def run_migrations_offline() -> None:
      url = config.get_main_option("sqlalchemy.url")
      context.configure(url=url, target_metadata=target_metadata, literal_binds=True)
      with context.begin_transaction():
          context.run_migrations()

  def do_run_migrations(connection):
      context.configure(connection=connection, target_metadata=target_metadata)
      with context.begin_transaction():
          context.run_migrations()

  async def run_async_migrations() -> None:
      connectable = async_engine_from_config(
          config.get_section(config.config_ini_section, {}),
          prefix="sqlalchemy.",
          poolclass=pool.NullPool,
      )
      async with connectable.connect() as connection:
          await connection.run_sync(do_run_migrations)
      await connectable.dispose()

  def run_migrations_online() -> None:
      asyncio.run(run_async_migrations())

  if context.is_offline_mode():
      run_migrations_offline()
  else:
      run_migrations_online()
  ```

- [ ] 运行 `uv run alembic revision --autogenerate -m "init: create all tables"`
- [ ] 确认生成的迁移文件包含 8 张表
- [ ] 运行 `uv run alembic upgrade head`
- [ ] 验证数据库有 8 张表：`\dt` 或 SQL 查询
- [ ] Commit: `feat: add Alembic migration for all tables`

**验收标准：** `alembic upgrade head` 成功创建 8 张表，字段/索引与设计文档一致。

---

### M1-B06: 通用依赖

**Files:**
- Create: `backend/src/dependencies.py`

**Steps:**

- [ ] 创建 `backend/src/dependencies.py`：
  ```python
  from collections.abc import AsyncGenerator

  from sqlalchemy.ext.asyncio import AsyncSession

  from src.database import async_session

  async def get_db() -> AsyncGenerator[AsyncSession, None]:
      async with async_session() as session:
          try:
              yield session
              await session.commit()
          except Exception:
              await session.rollback()
              raise
  ```

- [ ] Commit: `feat: add get_db dependency`

**验收标准：** `from src.dependencies import get_db` 无报错。

---

### M1-B07: 全局异常处理 + 统一错误响应

**Files:**
- Modify: `backend/src/main.py`

**Steps:**

- [ ] 在 `backend/src/main.py` 添加异常处理器：
  ```python
  from fastapi import FastAPI, Request
  from fastapi.middleware.cors import CORSMiddleware
  from fastapi.responses import JSONResponse
  from sqlalchemy.exc import IntegrityError

  app = FastAPI(title="Ideal Cafe", version="0.1.0")

  app.add_middleware(
      CORSMiddleware,
      allow_origins=["*"],
      allow_credentials=True,
      allow_methods=["*"],
      allow_headers=["*"],
  )

  @app.exception_handler(IntegrityError)
  async def integrity_error_handler(request: Request, exc: IntegrityError):
      return JSONResponse(status_code=409, content={"detail": "数据冲突"})

  @app.exception_handler(Exception)
  async def generic_error_handler(request: Request, exc: Exception):
      return JSONResponse(status_code=500, content={"detail": "服务器内部错误"})

  @app.get("/health")
  async def health_check():
      return {"status": "ok"}
  ```

- [ ] 验证访问不存在的路由返回 `{"detail": "Not Found"}`（FastAPI 默认行为已满足）
- [ ] Commit: `feat: add global exception handlers`

**验收标准：** 错误响应统一为 `{"detail": "..."}` 格式，覆盖 409/500。

---

### M1-B08: 初始化管理员脚本

**Files:**
- Create: `scripts/init_admin.py`

**Steps:**

- [ ] 创建 `scripts/init_admin.py`：
  ```python
  """初始化管理员账号脚本。用法: uv run python scripts/init_admin.py <phone> <password>"""
  import asyncio
  import sys
  import uuid

  from passlib.context import CryptContext

  sys.path.insert(0, "backend")

  from src.database import async_session, engine, Base
  from src.models.staff import Staff

  pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

  async def create_admin(phone: str, password: str):
      async with async_session() as session:
          from sqlalchemy import select
          result = await session.execute(select(Staff).where(Staff.phone == phone))
          existing = result.scalar_one_or_none()
          if existing:
              print(f"管理员账号 {phone} 已存在，跳过")
              return
          staff = Staff(
              id=uuid.uuid4(),
              phone=phone,
              password_hash=pwd_context.hash(password),
              role="admin",
              is_active=True,
          )
          session.add(staff)
          await session.commit()
          print(f"管理员账号 {phone} 创建成功")

  if __name__ == "__main__":
      if len(sys.argv) != 3:
          print("用法: uv run python scripts/init_admin.py <phone> <password>")
          sys.exit(1)
      asyncio.run(create_admin(sys.argv[1], sys.argv[2]))
  ```

- [ ] 运行 `uv run python scripts/init_admin.py 13800000001 123456`
- [ ] 验证数据库 staff 表有记录，role=admin
- [ ] 重复运行，验证输出"已存在，跳过"
- [ ] Commit: `feat: add init admin script`

**验收标准：** 可创建管理员，重复执行不报错。

---

## C端前端任务

### M1-C01: 初始化 uni-app Vue3 项目

**Files:**
- Create: `frontend/` 整个目录

**Steps:**

- [ ] 在项目根目录运行 `npx degit dcloudio/uni-preset-vue#vite-ts frontend`
- [ ] `cd frontend && pnpm install`
- [ ] 配置 `manifest.json` 的 appid（开发阶段可用测试 appid）
- [ ] `pnpm dev:mp-weixin` 可编译，微信开发者工具可打开 `dist/dev/mp-weixin`
- [ ] Commit: `feat: init C-end uni-app Vue3 project`

**验收标准：** `pnpm dev:mp-weixin` 可编译，微信开发者工具可打开。

---

### M1-C02: 配置 pages.json (TabBar)

**Files:**
- Modify: `frontend/src/pages.json`

**Steps:**

- [ ] 修改 `pages.json` 配置 TabBar：
  ```json
  {
    "pages": [
      { "path": "pages/index/index", "style": { "navigationBarTitleText": "菜单" } },
      { "path": "pages/order/index", "style": { "navigationBarTitleText": "订单" } },
      { "path": "pages/mine/index", "style": { "navigationBarTitleText": "我的" } }
    ],
    "subPackages": [
      { "root": "pages/product-detail", "pages": [{ "path": "index", "style": { "navigationBarTitleText": "商品详情" } }] },
      { "root": "pages/cart", "pages": [{ "path": "index", "style": { "navigationBarTitleText": "购物车" } }] },
      { "root": "pages/checkout", "pages": [{ "path": "index", "style": { "navigationBarTitleText": "确认订单" } }] },
      { "root": "pages/order-detail", "pages": [{ "path": "index", "style": { "navigationBarTitleText": "订单详情" } }] }
    ],
    "tabBar": {
      "color": "#999999",
      "selectedColor": "#333333",
      "list": [
        { "pagePath": "pages/index/index", "text": "菜单" },
        { "pagePath": "pages/order/index", "text": "订单" },
        { "pagePath": "pages/mine/index", "text": "我的" }
      ]
    }
  }
  ```

- [ ] 创建占位页面文件（3 个 tab 页 + 4 个子页面），每个只含 `<template><view>占位</view></template>`
- [ ] 编译验证 TabBar 显示正常
- [ ] Commit: `feat: configure pages.json with TabBar and sub-packages`

**验收标准：** 底部 TabBar 显示 3 个 tab，点击可切换。

---

### M1-C03: HTTP 请求封装

**Files:**
- Create: `frontend/src/api/request.ts`

**Steps:**

- [ ] 创建 `frontend/src/api/request.ts`：
  ```typescript
  const BASE_URL = import.meta.env.VITE_API_BASE_URL || ''

  interface RequestOptions {
    url: string
    method?: 'GET' | 'POST' | 'PUT' | 'DELETE'
    data?: any
    header?: Record<string, string>
  }

  function getToken(): string {
    return uni.getStorageSync('access_token') || ''
  }

  export async function request<T = any>(options: RequestOptions): Promise<T> {
    const { url, method = 'GET', data, header = {} } = options
    const token = getToken()

    const res = await uni.request({
      url: `${BASE_URL}${url}`,
      method,
      data,
      header: {
        'Content-Type': 'application/json',
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
        ...header,
      },
    })

    const statusCode = res.statusCode
    const responseData = res.data as any

    if (statusCode === 401) {
      // Token 过期，尝试刷新（后续 M2 实现）
      uni.removeStorageSync('access_token')
      uni.removeStorageSync('refresh_token')
      uni.reLaunch({ url: '/pages/index/index' })
      throw new Error(responseData.detail || '未登录')
    }

    if (statusCode >= 400) {
      throw new Error(responseData.detail || '请求失败')
    }

    return responseData as T
  }
  ```

- [ ] Commit: `feat: add HTTP request wrapper with token injection`

**验收标准：** 封装 uni.request，自动注入 token，401 时清除 token 并跳转。

---

### M1-C04: Pinia 初始化

**Files:**
- Create: `frontend/src/stores/index.ts`
- Modify: `frontend/src/main.ts`

**Steps:**

- [ ] 安装 `pnpm add pinia`
- [ ] 创建 `frontend/src/stores/index.ts`：
  ```typescript
  import { createPinia } from 'pinia'

  const pinia = createPinia()
  export default pinia
  ```
- [ ] 在 `main.ts` 中 `app.use(pinia)`
- [ ] Commit: `feat: add Pinia store`

**验收标准：** Pinia 可在组件中使用。

---

## S/A端前端任务

### M1-S01: 初始化 uni-app Vue3 项目 (H5)

**Files:**
- Create: `admin/` 整个目录

**Steps:**

- [ ] 在项目根目录运行 `npx degit dcloudio/uni-preset-vue#vite-ts admin`
- [ ] `cd admin && pnpm install`
- [ ] 配置 `manifest.json` 的 h5 路由基础路径
- [ ] `pnpm dev:h5` 可运行，浏览器访问正常
- [ ] Commit: `feat: init S/A-end uni-app Vue3 project`

**验收标准：** `pnpm dev:h5` 可运行，浏览器可访问。

---

### M1-S02: 配置 pages.json (侧边栏导航布局)

**Files:**
- Modify: `admin/src/pages.json`

**Steps:**

- [ ] 修改 `pages.json`：
  ```json
  {
    "pages": [
      { "path": "pages/login/index", "style": { "navigationBarTitleText": "登录" } },
      { "path": "pages/orders/index", "style": { "navigationBarTitleText": "订单处理" } },
      { "path": "pages/products/index", "style": { "navigationBarTitleText": "商品管理" } },
      { "path": "pages/shop/index", "style": { "navigationBarTitleText": "门店配置" } },
      { "path": "pages/staff/index", "style": { "navigationBarTitleText": "店员管理" } }
    ]
  }
  ```
- [ ] 创建占位页面文件
- [ ] Commit: `feat: configure admin pages.json`

**验收标准：** H5 模式下页面可正常导航。

---

### M1-S03: HTTP 请求封装

**Files:**
- Create: `admin/src/api/request.ts`

**Steps:**

- [ ] 同 M1-C03 逻辑，适配 H5 模式（使用 `window.location.origin` 作为 BASE_URL）
- [ ] Commit: `feat: add admin HTTP request wrapper`

**验收标准：** 同 M1-C03。

---

### M1-S04: Pinia 初始化

**Files:**
- Create: `admin/src/stores/index.ts`
- Modify: `admin/src/main.ts`

**Steps:**

- [ ] 同 M1-C04
- [ ] Commit: `feat: add admin Pinia store`

**验收标准：** Pinia 可在组件中使用。

---

## 测试任务

### M1-T01: 测试基础设施

**Files:**
- Create: `backend/tests/__init__.py`
- Create: `backend/tests/conftest.py`

**Steps:**

- [ ] 添加测试依赖到 pyproject.toml：
  ```toml
  [project.optional-dependencies]
  dev = [
      "pytest>=8.0.0",
      "pytest-asyncio>=0.24.0",
      "httpx>=0.27.0",
      "aiosqlite>=0.20.0",
  ]
  ```
- [ ] 创建 `backend/tests/conftest.py`：
  ```python
  import asyncio
  from collections.abc import AsyncGenerator

  import pytest
  from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

  from src.database import Base

  TEST_DATABASE_URL = "sqlite+aiosqlite:///./test.db"

  test_engine = create_async_engine(TEST_DATABASE_URL, echo=False)
  TestSession = async_sessionmaker(test_engine, expire_on_commit=False)

  @pytest.fixture(scope="session")
  def event_loop():
      loop = asyncio.new_event_loop()
      yield loop
      loop.close()

  @pytest.fixture(autouse=True)
  async def setup_db():
      async with test_engine.begin() as conn:
          await conn.run_sync(Base.metadata.create_all)
      yield
      async with test_engine.begin() as conn:
          await conn.run_sync(Base.metadata.drop_all)

  @pytest.fixture
  async def db_session() -> AsyncGenerator[AsyncSession, None]:
      async with TestSession() as session:
          yield session
  ```

- [ ] 创建 `backend/pytest.ini` 或在 pyproject.toml 添加：
  ```toml
  [tool.pytest.ini_options]
  asyncio_mode = "auto"
  ```
- [ ] 验证 `uv run pytest` 可运行（0 个测试也通过）
- [ ] Commit: `feat: add test infrastructure with pytest-asyncio`

**验收标准：** `uv run pytest` 可运行，测试用独立 SQLite，自动创建/清理表。

---

## 部署任务

### M1-D01: 开发环境 Docker Compose

**Files:**
- Create: `docker-compose.yml`

**Steps:**

- [ ] 创建 `docker-compose.yml`：
  ```yaml
  services:
    db:
      image: postgres:16-alpine
      environment:
        POSTGRES_DB: ideal_cafe
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: postgres
      ports:
        - "5432:5432"
      volumes:
        - pgdata:/var/lib/postgresql/data

    backend:
      build:
        context: ./backend
        dockerfile: Dockerfile
      ports:
        - "8000:8000"
      environment:
        DATABASE_URL: postgresql+asyncpg://postgres:postgres@db:5432/ideal_cafe
      depends_on:
        - db
      volumes:
        - ./backend/src:/app/src

  volumes:
    pgdata:
  ```

- [ ] Commit: `feat: add docker-compose for development`

**验收标准：** `docker compose up` 启动 PostgreSQL + FastAPI。

---

### M1-D02: Backend Dockerfile

**Files:**
- Create: `backend/Dockerfile`

**Steps:**

- [ ] 创建 `backend/Dockerfile`：
  ```dockerfile
  FROM python:3.12-slim

  COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

  WORKDIR /app

  COPY pyproject.toml uv.lock ./
  RUN uv sync --frozen --no-dev

  COPY src/ src/
  COPY alembic/ alembic/
  COPY alembic.ini ./

  EXPOSE 8000

  CMD ["uv", "run", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
  ```

- [ ] 运行 `docker compose build backend`
- [ ] Commit: `feat: add backend Dockerfile`

**验收标准：** `docker compose build backend` 成功。

---

### M1-D03: .env.example 模板

**Files:**
- Create: `.env.example`（项目根目录）

**Steps:**

- [ ] 创建 `.env.example`：
  ```env
  # 数据库
  DATABASE_URL=postgresql+asyncpg://postgres:postgres@db:5432/ideal_cafe

  # JWT
  SECRET_KEY=change-this-to-a-random-string-in-production
  ACCESS_TOKEN_EXPIRE_MINUTES=120
  REFRESH_TOKEN_EXPIRE_DAYS=30

  # 微信小程序
  WECHAT_APP_ID=your-app-id
  WECHAT_APP_SECRET=your-app-secret

  # 文件上传
  UPLOAD_DIR=uploads
  MAX_UPLOAD_SIZE_MB=5
  ```
- [ ] 确保 `.gitignore` 包含 `.env`
- [ ] Commit: `feat: add .env.example template`

**验收标准：** 所有环境变量有注释说明，.env 已在 .gitignore 中。

---

## 里程碑验收条件

- [ ] 后端 `/health` 返回 200
- [ ] 后端 `/docs` 可访问 Swagger UI
- [ ] `alembic upgrade head` 创建 8 张表（users, staff, categories, products, shop, orders, order_items, token_blacklist）
- [ ] C端 `pnpm dev:mp-weixin` 可编译，TabBar 显示 3 个 tab
- [ ] S/A端 `pnpm dev:h5` 可运行
- [ ] `docker compose up` 启动 PostgreSQL + FastAPI
- [ ] `uv run pytest` 可运行
- [ ] `uv run python scripts/init_admin.py 13800000001 123456` 创建管理员成功
