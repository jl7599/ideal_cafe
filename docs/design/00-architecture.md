# 架构设计

## 1. 技术栈选型

| 层级 | 技术 | 说明 |
|------|------|------|
| 后端框架 | FastAPI | 异步、自动 OpenAPI 文档 |
| ORM | SQLAlchemy 2.0 (async) | 声明式模型，异步引擎 |
| 数据库 | PostgreSQL | 主事务库，支持订单/商品/门店等核心数据存储 |
| 数据库迁移 | Alembic | 版本化管理 schema 变更 |
| 数据校验 | Pydantic v2 | 请求/响应 schema，FastAPI 原生集成 |
| C端前端 | uni-app (Vue3) | 编译到微信小程序 |
| S/A端前端 | uni-app (Vue3) | 编译到 H5 网页版 |
| 状态管理 | Pinia | Vue3 标准状态管理 |
| HTTP 客户端 | uni.request 封装 | 统一拦截、错误处理、token 注入 |
| 部署 | Docker + Docker Compose | 开发/生产环境统一 |
| 文件存储 | 本地磁盘 (MVP) | 后续迁移到 OSS |

## 2. 项目结构

```
ideal_cafe/
├── backend/
│   ├── src/
│   │   ├── main.py              # FastAPI 应用入口
│   │   ├── config.py            # 配置管理（环境变量）
│   │   ├── database.py          # 数据库引擎、会话
│   │   ├── dependencies.py      # 通用依赖（认证、权限）
│   │   ├── models/              # SQLAlchemy ORM 模型
│   │   │   ├── __init__.py
│   │   │   ├── user.py
│   │   │   ├── staff.py
│   │   │   ├── category.py
│   │   │   ├── product.py
│   │   │   ├── order.py
│   │   │   └── shop.py
│   │   ├── schemas/             # Pydantic 请求/响应模型
│   │   │   ├── __init__.py
│   │   │   ├── auth.py
│   │   │   ├── user.py
│   │   │   ├── staff.py
│   │   │   ├── category.py
│   │   │   ├── product.py
│   │   │   ├── order.py
│   │   │   └── shop.py
│   │   ├── routers/             # API 路由
│   │   │   ├── __init__.py
│   │   │   ├── auth.py
│   │   │   ├── users.py
│   │   │   ├── staff.py
│   │   │   ├── categories.py
│   │   │   ├── products.py
│   │   │   ├── orders.py
│   │   │   └── shop.py
│   │   ├── services/            # 业务逻辑层
│   │   │   ├── __init__.py
│   │   │   ├── auth.py
│   │   │   ├── order.py
│   │   │   └── shop.py
│   │   └── utils/
│   │       ├── __init__.py
│   │       └── order_no.py      # 订单号生成
│   ├── alembic/                 # 数据库迁移
│   ├── alembic.ini
│   ├── tests/
│   ├── pyproject.toml
│   └── uv.lock
├── frontend/                    # C端 — 微信小程序
│   ├── src/
│   │   ├── pages/
│   │   │   ├── index/           # 菜单主页
│   │   │   ├── order/           # 订单列表
│   │   │   ├── mine/            # 我的
│   │   │   ├── product-detail/  # 商品详情
│   │   │   ├── cart/            # 购物车
│   │   │   ├── checkout/        # 确认订单
│   │   │   └── order-detail/    # 订单详情
│   │   ├── components/
│   │   ├── api/                 # 请求封装
│   │   ├── stores/              # Pinia 状态管理
│   │   ├── utils/
│   │   └── static/
│   ├── package.json
│   └── ...
├── admin/                       # S/A端 — 网页版
│   ├── src/
│   │   ├── pages/
│   │   │   ├── login/           # 登录页
│   │   │   ├── orders/          # 订单处理/列表
│   │   │   ├── products/        # 商品管理
│   │   │   ├── shop/            # 门店配置
│   │   │   └── staff/           # 店员管理
│   │   ├── components/
│   │   ├── api/
│   │   ├── stores/
│   │   ├── utils/
│   │   └── static/
│   ├── package.json
│   └── ...
├── docs/
│   ├── prd/
│   └── design/
├── scripts/
│   └── init_admin.py            # 初始化管理员脚本
├── docker-compose.yml
└── README.md
```

## 3. 前端项目说明

C端和 S/A端拆为两个独立的 uni-app 项目：

| 维度 | C端 (frontend/) | S/A端 (admin/) |
|------|-----------------|----------------|
| 编译目标 | 微信小程序 | H5 网页版 |
| 用户角色 | 消费者 | 店员 / 管理员 |
| 页面结构 | TabBar + 子页面 | 侧边栏 + 内容区 |
| 认证方式 | 微信授权 | 账号密码 |
| 购物车 | 本地存储 | 无 |

**不合并为一个项目的原因**：页面结构、导航模式、认证流程完全不同，合并后条件编译复杂度远大于维护两个项目。

## 4. 后端分层架构

```
Router（路由层）
  │  职责：URL 映射、请求校验、响应格式
  │  不含业务逻辑
  │
  ▼
Service（业务层）
  │  职责：业务逻辑、状态流转、跨模型操作
  │  可被多个 Router 复用
  │
  ▼
Model（数据层）
  │  职责：ORM 映射、数据库操作
  │
  ▼
Database
```

**依赖注入**：通过 FastAPI 的 `Depends` 机制注入数据库会话和当前用户。

## 5. 认证架构

### 5.1 双 Token 机制

C端和 S/A端共用 Token 机制，但颁发逻辑不同：

```
C端登录流程：
  微信 code → 后端调微信 API 换 openid → 查找/创建用户 → 颁发 access_token + refresh_token

S/A端登录流程：
  手机号 + 密码 → 校验 → 颁发 access_token + refresh_token
```

### 5.2 Token 规则

| 参数 | 值 | 说明 |
|------|------|------|
| access_token 有效期 | 2 小时 | 短期令牌，用于接口认证 |
| refresh_token 有效期 | 30 天 | 长期令牌，用于刷新 access_token |
| Token 载荷 | `{user_id, role, type}` | role: customer/staff/admin，type: access/refresh |
| 传输方式 | Authorization: Bearer {token} | S/A端；C端存本地每次请求携带 |

### 5.3 角色与权限边界

```
请求 → 认证中间件（验证 token 有效性）
         │
         ├─ 无效/过期 → 401
         │
         └─ 有效 → 提取 role
                    │
                    ├─ staff 账号 is_active = FALSE → 401（软删除即时失效）
                    │
                    ├─ customer → 仅可访问 C端接口
                    ├─ staff    → 可访问 S端接口（订单操作）
                    └─ admin   → 可访问 S端查询 + A端接口（不可操作订单）
```

**与 PRD 权限矩阵一致**：admin 仅查看订单，不操作订单状态。

**权限校验原则**：后端接口根据角色校验权限，不依赖前端控制。每个 Router 声明允许的角色，`dependencies.py` 提供通用校验依赖。

## 6. 部署架构

```
Docker Compose
├── web (nginx)
│     ├── /        → admin (S/A端 H5)
│     ├── /api/    → backend
│     └── /miniprogram/ → 静态资源（小程序不需要，但可放管理后台）
├── backend (FastAPI + Uvicorn)
│     └── port 8000 (internal)
└── db (PostgreSQL)
      └── port 5432 (internal)
```

**开发环境**：`docker-compose.yml` + `.env` 本地配置

**生产环境**：同架构，额外配置 HTTPS、域名、数据库备份

## 7. 全局错误码

| HTTP 状态码 | 含义 | 场景 |
|-------------|------|------|
| 200 | 成功 | 正常响应 |
| 201 | 创建成功 | 下单、新增资源 |
| 400 | 请求参数错误 | 校验不通过 |
| 401 | 未认证 | Token 无效/过期 |
| 403 | 权限不足 | 角色无权访问 |
| 404 | 资源不存在 | 商品/订单不存在 |
| 409 | 状态冲突 | 订单状态不允许该操作 |
| 422 | 数据校验失败 | Pydantic 校验错误 |
| 429 | 请求过频 | 防重复提交 |
| 500 | 服务器错误 | 未预期异常 |

错误响应格式：

```json
{
  "detail": "错误描述"
}
```

## 8. API URL 规范

```
基础路径: /api/v1

认证:   /api/v1/auth/...
用户:   /api/v1/users/...
店员:   /api/v1/staff/...
分类:   /api/v1/categories/...
商品:   /api/v1/products/...
订单:   /api/v1/orders/...
门店:   /api/v1/shop/...
```

**命名规范**：
- URL 使用 kebab-case（如 `/order-detail/`）
- 资源名使用复数（如 `/categories/`）
- 操作使用动词（如 `/orders/{id}/cancel`）
