# 认证模块 API 设计

## 1. 概述

认证模块负责三种场景的登录：
- C端：微信授权登录（静默登录）
- S端：店员账号密码登录
- A端：管理员账号密码登录

S端和 A端共用登录接口，登录后根据 role 跳转不同页面。

## 2. 接口列表

### 2.1 POST /api/v1/auth/wechat/login — 微信登录（C端）

微信小程序调用 `wx.login()` 获取 code，发给后端换取 token。

**权限**：无（公开接口）

**请求体**：

```json
{
  "code": "0a3xxx..."  // wx.login() 返回的 code
}
```

**处理流程**：
1. 用 code 调用微信 API 换取 `openid` + `session_key`
2. 根据 openid 查找用户，不存在则自动创建（昵称=「理想咖啡」+ 随机4位数字，头像=默认）
3. 颁发 access_token + refresh_token

**成功响应** 200：

```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer",
  "expires_in": 7200,
  "user": {
    "id": "uuid",
    "nickname": "理想咖啡5827",
    "avatar_url": ""
  }
}
```

**错误响应**：
- 400：微信 code 无效或过期

### 2.2 POST /api/v1/auth/staff/login — 店员/管理员登录（S/A端）

**权限**：无（公开接口）

**请求体**：

```json
{
  "phone": "13800000001",
  "password": "123456"
}
```

**校验规则**：
- phone：11 位数字，以 1 开头
- password：6-20 位

**处理流程**：
1. 根据 phone 查找 staff 记录
2. 校验 password 与 password_hash
3. 校验 `is_active = TRUE`
4. 颁发 access_token + refresh_token

**成功响应** 200：

```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer",
  "expires_in": 7200,
  "staff": {
    "id": "uuid",
    "phone": "138****0001",
    "role": "staff"
  }
}
```

**错误响应**：
- 401：手机号或密码错误
- 403：账号已被禁用

### 2.3 POST /api/v1/auth/token/refresh — 刷新 Token

**权限**：需要有效的 refresh_token

**请求体**：

```json
{
  "refresh_token": "eyJ..."
}
```

**成功响应** 200：

```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer",
  "expires_in": 7200
}
```

**错误响应**：
- 401：refresh_token 无效或过期，需重新登录

### 2.4 POST /api/v1/auth/logout — 登出

**权限**：需认证（C/S/A端通用）

**请求体**：

```json
{
  "refresh_token": "eyJ..."
}
```

**处理流程**：
1. 将 refresh_token 加入服务端黑名单（数据库表 `token_blacklist`）
2. 客户端清除本地存储的 access_token 和 refresh_token
3. access_token 在过期前仍可使用（短期可接受），但无法再刷新

**成功响应** 200：

```json
{
  "detail": "已登出"
}
```

**说明**：
- refresh_token 黑名单确保登出后无法续期，access_token 最多存活 2 小时后自动失效
- 修改密码时同样将当前 refresh_token 加入黑名单，强制重新登录
- Token 验证时需同时检查 staff 的 `is_active` 状态，软删除后即使 Token 未过期也无法通过认证

### 2.5 GET /api/v1/users/me — 获取当前用户信息（C端）

**权限**：customer

**成功响应** 200：

```json
{
  "id": "uuid",
  "nickname": "理想咖啡5827",
  "avatar_url": "https://...",
  "created_at": "2026-04-10T10:00:00+08:00"
}
```

### 2.6 PUT /api/v1/users/me — 更新用户信息（C端）

**权限**：customer

**请求体**：

```json
{
  "nickname": "新昵称",
  "avatar_url": "https://..."
}
```

**校验规则**：
- nickname：1-50 字符，必填
- avatar_url：URL 格式，选填

**成功响应** 200：

```json
{
  "id": "uuid",
  "nickname": "新昵称",
  "avatar_url": "https://...",
  "created_at": "2026-04-10T10:00:00+08:00"
}
```

### 2.7 GET /api/v1/staff — 店员列表（A端）

**权限**：admin

**查询参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| is_active | boolean | 否 | 筛选状态，默认全部 |

**成功响应** 200：

```json
{
  "items": [
    {
      "id": "uuid",
      "phone": "138****0001",
      "role": "staff",
      "is_active": true,
      "created_at": "2026-04-10T10:00:00+08:00"
    }
  ],
  "total": 5
}
```

### 2.8 POST /api/v1/staff — 新增店员（A端）

**权限**：admin

**请求体**：

```json
{
  "phone": "13800000002",
  "password": "123456",
  "role": "staff"
}
```

**校验规则**：
- phone：11 位数字，以 1 开头，唯一
- password：6-20 位
- role：staff 或 admin

**成功响应** 201：

```json
{
  "id": "uuid",
  "phone": "13800000002",
  "role": "staff",
  "is_active": true,
  "created_at": "2026-04-10T10:00:00+08:00"
}
```

**错误响应**：
- 409：手机号已存在

### 2.9 PUT /api/v1/staff/{id}/reset-password — 重置店员密码（A端）

**权限**：admin

**请求体**：

```json
{
  "new_password": "654321"
}
```

**成功响应** 200：

```json
{
  "detail": "密码已重置"
}
```

### 2.10 DELETE /api/v1/staff/{id} — 删除店员（A端）

**权限**：admin

**处理逻辑**：设置 `is_active = FALSE`，非物理删除。

**成功响应** 200：

```json
{
  "detail": "店员已删除"
}
```

**错误响应**：
- 404：店员不存在
- 400：不能删除自己

### 2.11 PUT /api/v1/staff/me/password — 修改自身密码（S/A端）

**权限**：staff 或 admin

**请求体**：

```json
{
  "old_password": "123456",
  "new_password": "654321"
}
```

**校验规则**：
- old_password：必须正确
- new_password：6-20 位，不能与旧密码相同

**成功响应** 200：

```json
{
  "detail": "密码已修改，请重新登录"
}
```

**错误响应**：
- 400：旧密码错误

## 3. 权限边界汇总

| 接口 | customer | staff | admin | 公开 |
|------|----------|-------|-------|------|
| POST /auth/wechat/login | - | - | - | ✅ |
| POST /auth/staff/login | - | - | - | ✅ |
| POST /auth/token/refresh | - | - | - | ✅ |
| POST /auth/logout | ✅ | ✅ | ✅ | - |
| GET /users/me | ✅ | - | - | - |
| PUT /users/me | ✅ | - | - | - |
| GET /staff | - | - | ✅ | - |
| POST /staff | - | - | ✅ | - |
| PUT /staff/{id}/reset-password | - | - | ✅ | - |
| DELETE /staff/{id} | - | - | ✅ | - |
| PUT /staff/me/password | - | ✅ | ✅ | - |
