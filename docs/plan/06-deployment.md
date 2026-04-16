# M7: 部署上线 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-driven-development to implement this plan task-by-task.

**Goal:** 生产环境部署配置——多阶段构建 Dockerfile、Nginx 反向代理、生产 Docker Compose、HTTPS 配置指南、环境变量文档，实现 `docker compose up` 一键启动。

**Architecture:** Docker Compose 三服务：nginx(反向代理+静态文件) + backend(FastAPI+Uvicorn) + db(PostgreSQL)。Nginx 处理 /api/ → backend, / → admin H5。

**Tech Stack:** Docker, Docker Compose, Nginx, Let's Encrypt

**前置依赖：** M1-M6 全部完成

---

## 文件结构

```
ideal_cafe/
├── docker-compose.yml             # 生产配置（覆盖 M1 的开发配置）
├── docker-compose.dev.yml         # 开发配置（M1 已创建）
├── nginx/
│   └── nginx.conf                 # Nginx 配置
├── backend/
│   └── Dockerfile                 # 多阶段构建（覆盖 M1 的开发 Dockerfile）
├── admin/
│   └── Dockerfile                 # S/A端 H5 构建
├── scripts/
│   └── init_admin.py              # 已有
└── docs/
    └── deployment.md              # 部署文档
```

---

## 部署任务

### M7-D01: 生产 Dockerfile (多阶段构建)

**Files:**
- Modify: `backend/Dockerfile`

**Steps:**

- [ ] 重写 `backend/Dockerfile` 为多阶段构建：
  ```dockerfile
  # Stage 1: Install dependencies
  FROM python:3.12-slim AS builder
  COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
  WORKDIR /app
  COPY pyproject.toml uv.lock ./
  RUN uv sync --frozen --no-dev --no-install-project

  # Stage 2: Runtime
  FROM python:3.12-slim
  WORKDIR /app
  COPY --from=builder /app/.venv /app/.venv
  COPY src/ src/
  COPY alembic/ alembic/
  COPY alembic.ini ./
  ENV PATH="/app/.venv/bin:$PATH"
  EXPOSE 8000
  CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
  ```

- [ ] 验证 `docker compose build backend` 成功
- [ ] Commit: `feat: use multi-stage Docker build for production`

**验收标准：** 镜像体积 < 200MB，安全无冗余依赖。

---

### M7-D02: Nginx 配置

**Files:**
- Create: `nginx/nginx.conf`

**Steps:**

- [ ] 创建 `nginx/nginx.conf`：
  ```nginx
  upstream backend {
      server backend:8000;
  }

  server {
      listen 80;
      server_name _;

      client_max_body_size 5M;

      # API 反向代理
      location /api/ {
          proxy_pass http://backend;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }

      # S/A端 H5
      location / {
          root /usr/share/nginx/html;
          index index.html;
          try_files $uri $uri/ /index.html;
      }

      # 上传文件静态服务
      location /uploads/ {
          alias /app/uploads/;
      }
  }
  ```

- [ ] Commit: `feat: add Nginx configuration for reverse proxy`

**验收标准：** /api/ → backend，/ → admin H5，SPA fallback 正常。

---

### M7-D03: 生产 Docker Compose

**Files:**
- Modify: `docker-compose.yml` (生产配置)
- Create: `docker-compose.dev.yml` (开发配置，从原 docker-compose.yml 改名)

**Steps:**

- [ ] 重命名原 `docker-compose.yml` → `docker-compose.dev.yml`
- [ ] 创建生产 `docker-compose.yml`：
  ```yaml
  services:
    db:
      image: postgres:16-alpine
      environment:
        POSTGRES_DB: ${POSTGRES_DB:-ideal_cafe}
        POSTGRES_USER: ${POSTGRES_USER:-postgres}
        POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?POSTGRES_PASSWORD required}
      volumes:
        - pgdata:/var/lib/postgresql/data
      restart: unless-stopped

    backend:
      build:
        context: ./backend
        dockerfile: Dockerfile
      environment:
        DATABASE_URL: postgresql+asyncpg://${POSTGRES_USER:-postgres}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB:-ideal_cafe}
        SECRET_KEY: ${SECRET_KEY:?SECRET_KEY required}
        WECHAT_APP_ID: ${WECHAT_APP_ID:-}
        WECHAT_APP_SECRET: ${WECHAT_APP_SECRET:-}
        UPLOAD_DIR: /app/uploads
      depends_on:
        - db
      volumes:
        - uploads:/app/uploads
      restart: unless-stopped

    web:
      image: nginx:alpine
      ports:
        - "${PORT:-80}:80"
      volumes:
        - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
        - admin_dist:/usr/share/nginx/html
        - uploads:/app/uploads:ro
      depends_on:
        - backend
      restart: unless-stopped

  volumes:
    pgdata:
    uploads:
    admin_dist:
  ```

- [ ] 确保 `docker compose up` 一键启动三个服务
- [ ] Commit: `feat: add production docker-compose with nginx`

**验收标准：** `docker compose up` 启动 nginx + backend + db，完整业务流程可跑通。

---

### M7-D04: 环境变量文档 + 生产配置检查清单

**Files:**
- Create: `docs/deployment.md`
- Modify: `.env.example`

**Steps:**

- [ ] 创建 `docs/deployment.md`：
  ```markdown
  # 部署指南

  ## 环境要求
  - Docker 24+
  - Docker Compose v2+

  ## 快速启动
  1. 复制 `.env.example` 为 `.env`，填写生产配置
  2. 运行 `docker compose up -d`
  3. 运行数据库迁移：`docker compose exec backend uv run alembic upgrade head`
  4. 创建管理员：`docker compose exec backend uv run python scripts/init_admin.py <phone> <password>`
  5. 访问 http://your-domain/ 管理后台

  ## 环境变量

  | 变量 | 必填 | 说明 | 示例 |
  |------|------|------|------|
  | SECRET_KEY | 是 | JWT 密钥，随机字符串 | openssl rand -hex 32 |
  | POSTGRES_PASSWORD | 是 | 数据库密码 | 强密码 |
  | WECHAT_APP_ID | 是 | 微信小程序 AppID | wx... |
  | WECHAT_APP_SECRET | 是 | 微信小程序 AppSecret | ... |
  | PORT | 否 | Nginx 端口，默认 80 | 80 |

  ## 生产检查清单
  - [ ] SECRET_KEY 已更换为随机字符串
  - [ ] POSTGRES_PASSWORD 已设置为强密码
  - [ ] 微信 AppID/AppSecret 已配置
  - [ ] HTTPS 已配置（见下方）
  - [ ] 数据库备份策略已设置
  - [ ] 初始管理员已创建
  ```

- [ ] 更新 `.env.example`
- [ ] Commit: `feat: add deployment guide and env documentation`

**验收标准：** 文档完整，按文档可成功部署。

---

### M7-D05: HTTPS 配置指南

**Files:**
- Modify: `docs/deployment.md`

**Steps:**

- [ ] 在 `docs/deployment.md` 添加 HTTPS 配置：
  ```markdown
  ## HTTPS 配置 (Let's Encrypt)

  1. 安装 certbot: `apt install certbot`
  2. 获取证书: `certbot certonly --standalone -d your-domain.com`
  3. 修改 nginx.conf:
     - listen 443 ssl
     - ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem
     - ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem
  4. HTTP → HTTPS 重定向
  5. 设置自动续期: `certbot renew --deploy-hook "docker compose restart web"`
  ```
- [ ] Commit: `docs: add HTTPS configuration guide`

**验收标准：** 文档可用，按步骤可配置 HTTPS。

---

## 测试任务

### M7-T01: 端到端冒烟测试

**Steps:**

- [ ] 在 Docker 环境中执行完整业务流程：
  1. 创建管理员账号
  2. S/A端登录（admin）
  3. 创建分类 + 商品（含图片上传）
  4. 切换营业状态为"营业中"
  5. C端微信登录（模拟）
  6. 浏览菜单 → 加购物车 → 下单
  7. S端接单 → 制作 → 完成
  8. 验证取餐码正确
  9. 测试再来一单
  10. 切换营业状态为"打烊"，验证无法下单
  11. 订单列表/详情/状态轮询正常
  12. 店员管理（新增/禁用/重置密码）
- [ ] 记录测试结果
- [ ] Commit: `test: add e2e smoke test checklist`

**验收标准：** 完整业务流程在 Docker 环境中可跑通。

---

## 里程碑验收条件

- [ ] `docker compose up` 一键启动 nginx + backend + db
- [ ] `/api/` 路径正确代理到 backend
- [ ] `/` 路径正确显示 admin H5 页面
- [ ] API 文档可通过 `/api/docs` 访问
- [ ] 图片上传后可通过 URL 访问
- [ ] 完整业务流程在 Docker 环境中可跑通
- [ ] 环境变量文档完整
- [ ] HTTPS 配置指南可用
