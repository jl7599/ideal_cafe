# M2: 认证模块 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 实现三端认证（C端微信登录、S/A端账号密码登录），双 Token 机制（access_token + refresh_token），用户/店员 CRUD，角色权限校验。

**Architecture:** JWT 双 Token 机制，access_token 2h + refresh_token 30d，登出/改密码时 refresh_token 加入数据库黑名单。认证中间件根据角色和 is_active 校验权限。

**Tech Stack:** python-jose (JWT), passlib/bcrypt (密码哈希), httpx (微信 API 调用)

**前置依赖：** M1 完成

---

## 文件结构

```
backend/src/
├── utils/
│   └── jwt.py                 # JWT 生成/验证/解码
├── services/
│   ├── auth.py                # 登录、登出、刷新 Token 业务逻辑
│   └── user.py                # 用户信息业务逻辑
├── schemas/
│   ├── auth.py                # 认证请求/响应 schema
│   ├── user.py                # 用户 schema
│   └── staff.py               # 店员 schema
├── routers/
│   ├── auth.py                # /api/v1/auth/... 路由
│   ├── users.py               # /api/v1/users/... 路由
│   └── staff.py               # /api/v1/staff/... 路由
└── dependencies.py            # 增加 get_current_user, require_role

frontend/src/
├── api/
│   └── auth.ts                # 认证 API 调用
├── stores/
│   └── auth.ts                # 认证 store (token + user)
├── pages/mine/
│   └── index.vue              # 个人中心页
└── utils/
    └── auth.ts                # 登录态守卫

admin/src/
├── api/
│   └── auth.ts
├── stores/
│   └── auth.ts
├── pages/login/
│   └── index.vue
└── pages/change-password/
    └── index.vue
```

---

## 后端任务

### M2-B01: JWT 工具

**Files:**
- Create: `backend/src/utils/jwt.py`

**Steps:**

- [ ] 创建 `backend/src/utils/jwt.py`：
  ```python
  import uuid
  from datetime import datetime, timedelta, timezone

  from jose import JWTError, jwt

  from src.config import settings

  def create_access_token(user_id: str, role: str) -> tuple[str, datetime]:
      jti = str(uuid.uuid4())
      expire = datetime.now(timezone.utc) + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
      payload = {
          "sub": user_id,
          "role": role,
          "type": "access",
          "jti": jti,
          "exp": expire,
      }
      token = jwt.encode(payload, settings.SECRET_KEY, algorithm="HS256")
      return token, expire

  def create_refresh_token(user_id: str, role: str) -> tuple[str, datetime]:
      jti = str(uuid.uuid4())
      expire = datetime.now(timezone.utc) + timedelta(days=settings.REFRESH_TOKEN_EXPIRE_DAYS)
      payload = {
          "sub": user_id,
          "role": role,
          "type": "refresh",
          "jti": jti,
          "exp": expire,
      }
      token = jwt.encode(payload, settings.SECRET_KEY, algorithm="HS256")
      return token, expire

  def decode_token(token: str) -> dict:
      try:
          payload = jwt.decode(token, settings.SECRET_KEY, algorithms=["HS256"])
          return payload
      except JWTError:
          raise ValueError("Invalid token")

  def create_token_pair(user_id: str, role: str) -> dict:
      access_token, _ = create_access_token(user_id, role)
      refresh_token, refresh_expire = create_refresh_token(user_id, role)
      return {
          "access_token": access_token,
          "refresh_token": refresh_token,
          "token_type": "bearer",
          "expires_in": settings.ACCESS_TOKEN_EXPIRE_MINUTES * 60,
          "refresh_token_jti": jwt.decode(refresh_token, settings.SECRET_KEY, algorithms=["HS256"])["jti"],
          "refresh_token_expires_at": refresh_expire,
      }
  ```

- [ ] 验证：`uv run python -c "from src.utils.jwt import create_token_pair; print(create_token_pair('test-id', 'customer'))"`
- [ ] Commit: `feat: add JWT utility for access/refresh token`

**验收标准：** 可生成/解码 access_token 和 refresh_token，payload 包含 {sub, role, type, jti, exp}。

---

### M2-B02: Token 黑名单 service

**Files:**
- Create: `backend/src/services/auth.py` (部分)

**Steps:**

- [ ] 在 `backend/src/services/auth.py` 中添加黑名单逻辑：
  ```python
  from datetime import datetime
  from sqlalchemy import select
  from sqlalchemy.ext.asyncio import AsyncSession

  from src.models.token_blacklist import TokenBlacklist

  async def add_to_blacklist(db: AsyncSession, jti: str, expires_at: datetime) -> None:
      entry = TokenBlacklist(token_jti=jti, expires_at=expires_at)
      db.add(entry)

  async def is_in_blacklist(db: AsyncSession, jti: str) -> bool:
      result = await db.execute(
          select(TokenBlacklist).where(TokenBlacklist.token_jti == jti)
      )
      return result.scalar_one_or_none() is not None
  ```

- [ ] Commit: `feat: add token blacklist service`

**验收标准：** 可加入黑名单，可查询是否在黑名单中。

---

### M2-B03: 认证依赖 (get_current_user, require_role)

**Files:**
- Modify: `backend/src/dependencies.py`

**Steps:**

- [ ] 在 `dependencies.py` 添加认证依赖：
  ```python
  from collections.abc import AsyncGenerator
  from functools import wraps

  from fastapi import Depends, HTTPException, status
  from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
  from sqlalchemy import select
  from sqlalchemy.ext.asyncio import AsyncSession

  from src.database import async_session
  from src.models.staff import Staff
  from src.models.user import User
  from src.services.auth import is_in_blacklist
  from src.utils.jwt import decode_token

  security = HTTPBearer()

  async def get_db() -> AsyncGenerator[AsyncSession, None]:
      async with async_session() as session:
          try:
              yield session
              await session.commit()
          except Exception:
              await session.rollback()
              raise

  async def get_current_user(
      credentials: HTTPAuthorizationCredentials = Depends(security),
      db: AsyncSession = Depends(get_db),
  ) -> dict:
      token = credentials.credentials
      try:
          payload = decode_token(token)
      except ValueError:
          raise HTTPException(status_code=401, detail="Token 无效")

      if payload.get("type") != "access":
          raise HTTPException(status_code=401, detail="Token 类型错误")

      # staff 检查 is_active
      if payload["role"] in ("staff", "admin"):
          result = await db.execute(select(Staff).where(Staff.id == payload["sub"]))
          staff = result.scalar_one_or_none()
          if not staff or not staff.is_active:
              raise HTTPException(status_code=401, detail="账号已被禁用")

      return {"user_id": payload["sub"], "role": payload["role"]}

  def require_role(*roles: str):
      async def check_role(current_user: dict = Depends(get_current_user)) -> dict:
          if current_user["role"] not in roles:
              raise HTTPException(status_code=403, detail="权限不足")
          return current_user
      return check_role
  ```

- [ ] Commit: `feat: add auth dependencies (get_current_user, require_role)`

**验收标准：** 无效 token → 401，角色不符 → 403，staff is_active=FALSE → 401。

---

### M2-B04: 微信登录 endpoint

**Files:**
- Create: `backend/src/schemas/auth.py`
- Create: `backend/src/routers/auth.py`
- Modify: `backend/src/services/auth.py`
- Modify: `backend/src/main.py` (注册路由)

**Steps:**

- [ ] 创建 `backend/src/schemas/auth.py`：
  ```python
  from pydantic import BaseModel, Field

  class WechatLoginRequest(BaseModel):
      code: str = Field(..., min_length=1)

  class StaffLoginRequest(BaseModel):
      phone: str = Field(..., pattern=r"^1\d{10}$")
      password: str = Field(..., min_length=6, max_length=20)

  class RefreshTokenRequest(BaseModel):
      refresh_token: str

  class LogoutRequest(BaseModel):
      refresh_token: str

  class TokenResponse(BaseModel):
      access_token: str
      refresh_token: str
      token_type: str = "bearer"
      expires_in: int

  class WechatLoginResponse(TokenResponse):
      user: dict

  class StaffLoginResponse(TokenResponse):
      staff: dict
  ```

- [ ] 在 `backend/src/services/auth.py` 添加微信登录逻辑：
  ```python
  import httpx
  import uuid

  from sqlalchemy import select
  from sqlalchemy.ext.asyncio import AsyncSession

  from src.config import settings
  from src.models.user import User
  from src.utils.jwt import create_token_pair

  async def wechat_login(db: AsyncSession, code: str) -> dict:
      async with httpx.AsyncClient() as client:
          resp = await client.get(
              "https://api.weixin.qq.com/sns/jscode2session",
              params={
                  "appid": settings.WECHAT_APP_ID,
                  "secret": settings.WECHAT_APP_SECRET,
                  "js_code": code,
                  "grant_type": "authorization_code",
              },
          )
      data = resp.json()
      if "errcode" in data and data["errcode"] != 0:
          raise ValueError(f"微信登录失败: {data.get('errmsg', '')}")

      openid = data["openid"]
      session_key = data.get("session_key")

      result = await db.execute(select(User).where(User.openid == openid))
      user = result.scalar_one_or_none()

      if not user:
          import random
          nickname = f"理想咖啡{random.randint(1000, 9999)}"
          user = User(openid=openid, session_key=session_key, nickname=nickname)
          db.add(user)
          await db.flush()

      tokens = create_token_pair(str(user.id), "customer")
      return {
          **tokens,
          "user": {
              "id": str(user.id),
              "nickname": user.nickname,
              "avatar_url": user.avatar_url,
          },
      }
  ```

- [ ] 创建 `backend/src/routers/auth.py`：
  ```python
  from fastapi import APIRouter, Depends, HTTPException, status
  from sqlalchemy.ext.asyncio import AsyncSession

  from src.dependencies import get_db
  from src.schemas.auth import WechatLoginRequest, WechatLoginResponse
  from src.services.auth import wechat_login

  router = APIRouter(prefix="/api/v1/auth", tags=["auth"])

  @router.post("/wechat/login", response_model=WechatLoginResponse)
  async def wechat_login_endpoint(body: WechatLoginRequest, db: AsyncSession = Depends(get_db)):
      try:
          return await wechat_login(db, body.code)
      except ValueError as e:
          raise HTTPException(status_code=400, detail=str(e))
  ```

- [ ] 在 `main.py` 注册路由：`app.include_router(router)`
- [ ] 测试：`POST /api/v1/auth/wechat/login` 需要有效微信 code（MVP 可先用 mock 测试）
- [ ] Commit: `feat: add wechat login endpoint`

**验收标准：** 有效 code → 颁发 token + 创建/查找用户，无效 code → 400。

---

### M2-B05: 店员登录 endpoint

**Files:**
- Modify: `backend/src/routers/auth.py`
- Modify: `backend/src/services/auth.py`

**Steps:**

- [ ] 在 `services/auth.py` 添加店员登录：
  ```python
  from passlib.context import CryptContext

  pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

  async def staff_login(db: AsyncSession, phone: str, password: str) -> dict:
      result = await db.execute(select(Staff).where(Staff.phone == phone))
      staff = result.scalar_one_or_none()

      if not staff or not pwd_context.verify(password, staff.password_hash):
          raise ValueError("手机号或密码错误")
      if not staff.is_active:
          raise ValueError("账号已被禁用")

      tokens = create_token_pair(str(staff.id), staff.role)
      return {
          **tokens,
          "staff": {
              "id": str(staff.id),
              "phone": staff.phone[:3] + "****" + staff.phone[-4:],
              "role": staff.role,
          },
      }
  ```

- [ ] 在 `routers/auth.py` 添加路由：
  ```python
  @router.post("/staff/login")
  async def staff_login_endpoint(body: StaffLoginRequest, db: AsyncSession = Depends(get_db)):
      try:
          return await staff_login(db, body.phone, body.password)
      except ValueError as e:
          msg = str(e)
          if "禁用" in msg:
              raise HTTPException(status_code=403, detail=msg)
          raise HTTPException(status_code=401, detail=msg)
  ```

- [ ] Commit: `feat: add staff login endpoint`

**验收标准：** 正确密码 → 颁发 token，错误密码 → 401，禁用账号 → 403。

---

### M2-B06: 刷新 Token endpoint

**Files:**
- Modify: `backend/src/routers/auth.py`
- Modify: `backend/src/services/auth.py`

**Steps:**

- [ ] 在 `services/auth.py` 添加刷新逻辑：
  ```python
  async def refresh_access_token(db: AsyncSession, refresh_token: str) -> dict:
      try:
          payload = decode_token(refresh_token)
      except ValueError:
          raise ValueError("refresh_token 无效或过期")

      if payload.get("type") != "refresh":
          raise ValueError("Token 类型错误")

      jti = payload.get("jti")
      if await is_in_blacklist(db, jti):
          raise ValueError("refresh_token 已失效")

      # staff 检查 is_active
      if payload["role"] in ("staff", "admin"):
          result = await db.execute(select(Staff).where(Staff.id == payload["sub"]))
          staff = result.scalar_one_or_none()
          if not staff or not staff.is_active:
              raise ValueError("账号已被禁用")

      return create_token_pair(payload["sub"], payload["role"])
  ```

- [ ] 在 `routers/auth.py` 添加路由：
  ```python
  @router.post("/token/refresh")
  async def refresh_token_endpoint(body: RefreshTokenRequest, db: AsyncSession = Depends(get_db)):
      try:
          return await refresh_access_token(db, body.refresh_token)
      except ValueError as e:
          raise HTTPException(status_code=401, detail=str(e))
  ```

- [ ] Commit: `feat: add token refresh endpoint`

**验收标准：** 有效 refresh_token → 颁发新 token 对，黑名单中的 → 401。

---

### M2-B07: 登出 endpoint

**Files:**
- Modify: `backend/src/routers/auth.py`

**Steps:**

- [ ] 在 `routers/auth.py` 添加：
  ```python
  from src.dependencies import get_current_user

  @router.post("/logout")
  async def logout(
      body: LogoutRequest,
      current_user: dict = Depends(get_current_user),
      db: AsyncSession = Depends(get_db),
  ):
      payload = decode_token(body.refresh_token)
      jti = payload.get("jti")
      expires_at = datetime.fromtimestamp(payload["exp"], tz=timezone.utc)
      await add_to_blacklist(db, jti, expires_at)
      return {"detail": "已登出"}
  ```

- [ ] Commit: `feat: add logout endpoint`

**验收标准：** 登出后 refresh_token 加入黑名单，无法再刷新。

---

### M2-B08: 用户信息 endpoints

**Files:**
- Create: `backend/src/schemas/user.py`
- Create: `backend/src/routers/users.py`

**Steps:**

- [ ] 创建 `backend/src/schemas/user.py`：
  ```python
  from datetime import datetime
  from pydantic import BaseModel, Field

  class UserResponse(BaseModel):
      id: str
      nickname: str
      avatar_url: str
      created_at: datetime

  class UserUpdateRequest(BaseModel):
      nickname: str = Field(..., min_length=1, max_length=50)
      avatar_url: str = Field("", max_length=500)
  ```

- [ ] 创建 `backend/src/routers/users.py`：
  ```python
  router = APIRouter(prefix="/api/v1/users", tags=["users"])

  @router.get("/me", response_model=UserResponse)
  async def get_me(current_user: dict = Depends(require_role("customer")), db: AsyncSession = Depends(get_db)):
      result = await db.execute(select(User).where(User.id == current_user["user_id"]))
      user = result.scalar_one_or_none()
      if not user:
          raise HTTPException(status_code=404, detail="用户不存在")
      return UserResponse(id=str(user.id), nickname=user.nickname, avatar_url=user.avatar_url, created_at=user.created_at)

  @router.put("/me", response_model=UserResponse)
  async def update_me(body: UserUpdateRequest, current_user: dict = Depends(require_role("customer")), db: AsyncSession = Depends(get_db)):
      result = await db.execute(select(User).where(User.id == current_user["user_id"]))
      user = result.scalar_one_or_none()
      if not user:
          raise HTTPException(status_code=404, detail="用户不存在")
      user.nickname = body.nickname
      user.avatar_url = body.avatar_url
      return UserResponse(id=str(user.id), nickname=user.nickname, avatar_url=user.avatar_url, created_at=user.created_at)
  ```

- [ ] 在 `main.py` 注册路由
- [ ] Commit: `feat: add user profile endpoints`

**验收标准：** customer 可查看/修改昵称和头像。

---

### M2-B09: 店员 CRUD endpoints

**Files:**
- Create: `backend/src/schemas/staff.py`
- Create: `backend/src/routers/staff.py`

**Steps:**

- [ ] 创建 `backend/src/schemas/staff.py`：
  ```python
  class StaffResponse(BaseModel):
      id: str
      phone: str
      role: str
      is_active: bool
      created_at: datetime

  class StaffListResponse(BaseModel):
      items: list[StaffResponse]
      total: int

  class StaffCreateRequest(BaseModel):
      phone: str = Field(..., pattern=r"^1\d{10}$")
      password: str = Field(..., min_length=6, max_length=20)
      role: str = Field(..., pattern=r"^(staff|admin)$")
  ```

- [ ] 创建 `backend/src/routers/staff.py`，实现：
  - `GET /api/v1/staff` — admin 查询店员列表，支持 is_active 筛选
  - `POST /api/v1/staff` — admin 新增店员，phone 唯一校验（409）
  - `DELETE /api/v1/staff/{id}` — admin 软删除（is_active=False），不能删自己（400）
  - 参照 design/02-api-auth.md 2.7-2.10 节
- [ ] 在 `main.py` 注册路由
- [ ] Commit: `feat: add staff CRUD endpoints`

**验收标准：** admin 可增删查店员，phone 重复 → 409，删除自己 → 400，软删除生效。

---

### M2-B10: 密码管理 endpoints

**Files:**
- Modify: `backend/src/routers/staff.py`

**Steps:**

- [ ] 在 `routers/staff.py` 添加：
  - `PUT /api/v1/staff/me/password` — 修改自身密码，旧密码校验 + 新 token 黑名单
  - `PUT /api/v1/staff/{id}/reset-password` — admin 重置店员密码
  - 参照 design/02-api-auth.md 2.9 和 2.11 节
- [ ] Commit: `feat: add password management endpoints`

**验收标准：** 改密码后旧 refresh_token 失效，admin 可重置店员密码。

---

### M2-B11: Auth + User + Staff schemas 补全

**Files:**
- Modify: `backend/src/schemas/auth.py`
- Modify: `backend/src/schemas/user.py`
- Modify: `backend/src/schemas/staff.py`

**Steps:**

- [ ] 对照 design/02-api-auth.md 检查所有请求/响应字段，确保完整
- [ ] 特别注意：StaffLoginResponse 的 phone 需脱敏（138****0001）
- [ ] 确保 TokenResponse 的 expires_in 单位为秒
- [ ] Commit: `fix: ensure all auth schemas match design doc`

**验收标准：** 所有 schema 字段与设计文档一致。

---

## C端前端任务

### M2-C01: 微信登录流程

**Files:**
- Create: `frontend/src/api/auth.ts`
- Modify: `frontend/src/api/request.ts` (401 刷新逻辑)

**Steps:**

- [ ] 创建 `frontend/src/api/auth.ts`：
  ```typescript
  import { request } from './request'

  export function wechatLogin() {
    return new Promise((resolve, reject) => {
      uni.login({
        provider: 'weixin',
        success: async (loginRes) => {
          try {
            const data = await request({
              url: '/api/v1/auth/wechat/login',
              method: 'POST',
              data: { code: loginRes.code },
            })
            resolve(data)
          } catch (e) {
            reject(e)
          }
        },
        fail: reject,
      })
    })
  }
  ```

- [ ] 在 `request.ts` 的 401 处理中添加 refresh_token 刷新逻辑：
  ```typescript
  // 401 时尝试用 refresh_token 刷新
  const refreshToken = uni.getStorageSync('refresh_token')
  if (refreshToken && !isRetrying) {
    isRetrying = true
    try {
      const res = await request({
        url: '/api/v1/auth/token/refresh',
        method: 'POST',
        data: { refresh_token: refreshToken },
      })
      uni.setStorageSync('access_token', res.access_token)
      uni.setStorageSync('refresh_token', res.refresh_token)
      // 重试原请求
      return request(options)
    } catch {
      uni.removeStorageSync('access_token')
      uni.removeStorageSync('refresh_token')
    } finally {
      isRetrying = false
    }
  }
  ```

- [ ] Commit: `feat: add wechat login flow with token refresh`

**验收标准：** 首次打开自动静默登录，token 过期自动刷新。

---

### M2-C02: Auth store

**Files:**
- Create: `frontend/src/stores/auth.ts`

**Steps:**

- [ ] 创建 `frontend/src/stores/auth.ts`：
  ```typescript
  import { defineStore } from 'pinia'
  import { wechatLogin } from '../api/auth'

  export const useAuthStore = defineStore('auth', {
    state: () => ({
      accessToken: uni.getStorageSync('access_token') || '',
      refreshToken: uni.getStorageSync('refresh_token') || '',
      user: null as any,
    }),
    getters: {
      isLoggedIn: (state) => !!state.accessToken,
    },
    actions: {
      async login() {
        const data: any = await wechatLogin()
        this.accessToken = data.access_token
        this.refreshToken = data.refresh_token
        this.user = data.user
        uni.setStorageSync('access_token', data.access_token)
        uni.setStorageSync('refresh_token', data.refresh_token)
      },
      logout() {
        this.accessToken = ''
        this.refreshToken = ''
        this.user = null
        uni.removeStorageSync('access_token')
        uni.removeStorageSync('refresh_token')
      },
    },
  })
  ```

- [ ] Commit: `feat: add auth store with persistent tokens`

**验收标准：** 刷新页面后仍保持登录态。

---

### M2-C03: 登录态守卫

**Files:**
- Create: `frontend/src/utils/auth.ts`

**Steps:**

- [ ] 创建 `frontend/src/utils/auth.ts`：
  ```typescript
  import { useAuthStore } from '../stores/auth'

  export async function ensureLogin() {
    const authStore = useAuthStore()
    if (!authStore.isLoggedIn) {
      await authStore.login()
    }
  }
  ```

- [ ] 在需要登录的页面（购物车、订单等）调用 `ensureLogin()`
- [ ] Commit: `feat: add auth guard utility`

**验收标准：** 未登录时访问需登录页面，自动触发静默登录。

---

### M2-C04: 个人中心页

**Files:**
- Modify: `frontend/src/pages/mine/index.vue`

**Steps:**

- [ ] 实现个人中心页：显示昵称、头像，可修改
- [ ] 调用 `GET /api/v1/users/me` 获取信息
- [ ] 调用 `PUT /api/v1/users/me` 修改昵称/头像
- [ ] Commit: `feat: add mine page with profile editing`

**验收标准：** 可查看和编辑个人信息。

---

## S/A端前端任务

### M2-S01: 登录页 UI

**Files:**
- Modify: `admin/src/pages/login/index.vue`

**Steps:**

- [ ] 实现登录页：手机号 + 密码表单，前端校验
- [ ] 校验规则：手机号 11 位以 1 开头，密码 6-20 位
- [ ] 错误提示：手机号或密码错误（401）、账号已被禁用（403）
- [ ] Commit: `feat: add login page UI`

**验收标准：** 表单校验正常，提交后显示错误信息。

---

### M2-S02: 登录 API 对接 + 角色路由

**Files:**
- Create: `admin/src/api/auth.ts`
- Create: `admin/src/stores/auth.ts`

**Steps:**

- [ ] 创建 `admin/src/api/auth.ts`：staffLogin, refreshToken, logout
- [ ] 创建 `admin/src/stores/auth.ts`：存储 token + staff 信息，登录后根据 role 跳转
  - staff → `/pages/orders/index`
  - admin → `/pages/products/index`
- [ ] 在 `request.ts` 中添加 401 自动刷新逻辑
- [ ] Commit: `feat: add admin auth store with role-based routing`

**验收标准：** 登录成功后跳转对应角色页面。

---

### M2-S03: 修改密码页

**Files:**
- Create: `admin/src/pages/change-password/index.vue`

**Steps:**

- [ ] 实现修改密码页：旧密码 + 新密码 + 确认新密码
- [ ] 调用 `PUT /api/v1/staff/me/password`
- [ ] 成功后清除 token，跳转登录页
- [ ] Commit: `feat: add change password page`

**验收标准：** 改密码成功后跳转登录页重新登录。

---

## 测试任务

### M2-T01: 认证 API 测试

**Files:**
- Create: `backend/tests/test_auth.py`

**Steps:**

- [ ] 测试用例：
  - `test_staff_login_success` — 正确密码登录成功
  - `test_staff_login_wrong_password` — 错误密码 → 401
  - `test_staff_login_inactive` — 禁用账号 → 403
  - `test_refresh_token` — 刷新成功获取新 token 对
  - `test_refresh_token_blacklisted` — 黑名单中的 refresh_token → 401
  - `test_logout` — 登出后 refresh_token 不可刷新
  - `test_require_role_customer` — customer 访问 staff 接口 → 403
  - `test_require_role_admin` — staff 访问 admin 接口 → 403
- [ ] 运行 `uv run pytest tests/test_auth.py -v`
- [ ] Commit: `test: add auth API tests`

**验收标准：** 覆盖正常流程 + 异常场景，全部通过。

---

## 里程碑验收条件

- [ ] 微信登录：有效 code → 颁发 token + 创建/查找用户
- [ ] 店员登录：正确密码 → 颁发 token，错误 → 401，禁用 → 403
- [ ] Token 刷新：有效 refresh_token → 新 token 对，黑名单中 → 401
- [ ] 登出：refresh_token 加入黑名单，无法再刷新
- [ ] 角色权限：customer 不可访问 staff/admin 接口，staff 不可访问 admin 接口
- [ ] C端：静默登录成功，个人中心可编辑
- [ ] S/A端：登录页登录成功，角色路由正确
