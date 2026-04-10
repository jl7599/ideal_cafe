# 菜单模块 API 设计

## 1. 概述

菜单模块负责商品分类和商品的 CRUD，C端浏览菜单，A端管理商品。

**C端规则**：
- 只返回 `is_enabled = TRUE` 的分类
- 只返回 `is_on_shelf = TRUE` 的商品
- 无需登录即可浏览（但加入购物车需要登录）

**A端规则**：
- 返回全部分类和商品（含禁用/下架）
- 可进行增删改查和上下架操作

## 2. 接口列表

### 2.1 GET /api/v1/categories — 分类列表

**权限**：customer（公开，无需登录）、admin

**查询参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| all | boolean | 否 | 是否返回全部（含禁用），默认 false。A端传 true |

**行为差异**：
- customer / 未登录：仅返回 `is_enabled = TRUE` 的分类
- admin + all=true：返回全部分类

**成功响应** 200：

```json
{
  "items": [
    {
      "id": "uuid",
      "name": "意式咖啡",
      "icon": "https://...",
      "sort_order": 1,
      "is_enabled": true
    }
  ],
  "total": 5
}
```

### 2.2 POST /api/v1/categories — 新增分类（A端）

**权限**：admin

**请求体**：

```json
{
  "name": "手冲单品",
  "icon": "",
  "sort_order": 3,
  "is_enabled": true
}
```

**校验规则**：
- name：1-50 字符，必填
- icon：URL 格式，选填
- sort_order：整数，选填。未传时由后端自动计算为当前最大值 + 1

**成功响应** 201：

```json
{
  "id": "uuid",
  "name": "手冲单品",
  "icon": "",
  "sort_order": 3,
  "is_enabled": true
}
```

### 2.3 PUT /api/v1/categories/{id} — 编辑分类（A端）

**权限**：admin

**请求体**（部分更新，仅传需要修改的字段）：

```json
{
  "name": "手冲咖啡",
  "icon": "https://...",
  "sort_order": 2,
  "is_enabled": true
}
```

**校验规则**：
- 所有字段可选，至少传一个字段
- name：1-50 字符
- icon：URL 格式
- sort_order：整数，选填。未传时由后端自动计算为当前最大值 + 1

**成功响应** 200：同新增响应

**错误响应**：
- 404：分类不存在

### 2.4 DELETE /api/v1/categories/{id} — 删除分类（A端）

**权限**：admin

**处理逻辑**：
1. 检查分类下是否有商品
2. 有商品 → 拒绝删除
3. 无商品 → 物理删除

**成功响应** 200：

```json
{
  "detail": "分类已删除"
}
```

**错误响应**：
- 404：分类不存在
- 409：分类下有商品，不可删除

### 2.5 GET /api/v1/products — 商品列表

**权限**：customer（公开）、admin

**查询参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| category_id | UUID | 否 | 按分类筛选 |
| all | boolean | 否 | 是否返回全部（含下架），默认 false。A端传 true |
| page | integer | 否 | 页码，默认 1 |
| page_size | integer | 否 | 每页条数，默认 20，最大 100 |

**行为差异**：
- customer / 未登录：仅返回 `is_on_shelf = TRUE` 的商品
- admin + all=true：返回全部商品

**成功响应** 200：

```json
{
  "items": [
    {
      "id": "uuid",
      "category_id": "uuid",
      "category_name": "意式咖啡",
      "name": "燕麦拿铁",
      "subtitle": "燕麦奶与浓缩的经典搭配",
      "description": "...",
      "price": 2800,
      "price_display": "28.00",
      "image_url": "https://...",
      "sort_order": 1,
      "is_on_shelf": true
    }
  ],
  "total": 15,
  "page": 1,
  "page_size": 20
}
```

**说明**：
- `price`：整数，单位分
- `price_display`：格式化的展示价格，如 "28.00"
- `category_name`：冗余返回，减少前端额外请求

### 2.6 GET /api/v1/products/{id} — 商品详情

**权限**：customer（公开）、admin

**成功响应** 200：

```json
{
  "id": "uuid",
  "category_id": "uuid",
  "category_name": "意式咖啡",
  "name": "燕麦拿铁",
  "subtitle": "燕麦奶与浓缩的经典搭配",
  "description": "选用瑞典 OATLY 燕麦奶...",
  "price": 2800,
  "price_display": "28.00",
  "image_url": "https://...",
  "sort_order": 1,
  "is_on_shelf": true
}
```

**错误响应**：
- 404：商品不存在
- customer 访问已下架商品也返回 404

### 2.7 POST /api/v1/products — 新增商品（A端）

**权限**：admin

**请求体**：

```json
{
  "category_id": "uuid",
  "name": "燕麦拿铁",
  "subtitle": "燕麦奶与浓缩的经典搭配",
  "description": "选用瑞典 OATLY 燕麦奶...",
  "price": 2800,
  "image_url": "https://...",
  "sort_order": 1,
  "is_on_shelf": true
}
```

**校验规则**：
- category_id：必填，分类必须存在
- name：1-100 字符，必填
- subtitle：选填，最大 200 字符
- description：选填
- price：正整数，单位分，必填
- image_url：URL 格式，必填
- sort_order：整数，选填。未传时由后端自动计算为当前分类下最大值 + 1
- is_on_shelf：布尔，默认 true

**成功响应** 201：同商品详情响应

### 2.8 PUT /api/v1/products/{id} — 编辑商品（A端）

**权限**：admin

**请求体**：同新增，所有字段可选

**成功响应** 200：同商品详情响应

**错误响应**：
- 404：商品不存在

### 2.9 DELETE /api/v1/products/{id} — 删除商品（A端）

**权限**：admin

**处理逻辑**：物理删除。已有订单保存了商品快照，不受影响。

**成功响应** 200：

```json
{
  "detail": "商品已删除"
}
```

**错误响应**：
- 404：商品不存在

### 2.10 PUT /api/v1/products/{id}/shelf — 切换上下架（A端）

**权限**：admin

**请求体**：

```json
{
  "is_on_shelf": false
}
```

**成功响应** 200：

```json
{
  "id": "uuid",
  "is_on_shelf": false
}
```

**错误响应**：
- 404：商品不存在

## 3. 文件上传接口

### 3.1 POST /api/v1/upload — 上传文件（A端）

**权限**：admin

**请求体**：`multipart/form-data`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| file | File | 是 | 上传的文件 |

**校验规则**：
- 允许的文件类型：`image/jpeg`, `image/png`, `image/webp`
- 文件大小：最大 5MB
- 文件名：自动生成 UUID 文件名，避免路径冲突

**存储方式**：
- MVP：保存到服务器本地 `uploads/` 目录，通过 `/api/v1/static/` 路径访问
- 后续可迁移到 OSS（如阿里云 OSS、腾讯云 COS）

**成功响应** 200：

```json
{
  "url": "/api/v1/static/abc123.jpg"
}
```

**错误响应**：
- 400：文件类型不允许 / 文件过大

## 4. 权限边界汇总

| 接口 | 公开 | customer | admin |
|------|------|----------|-------|
| GET /categories | ✅ (仅启用) | - | ✅ (全部) |
| POST /categories | - | - | ✅ |
| PUT /categories/{id} | - | - | ✅ |
| DELETE /categories/{id} | - | - | ✅ |
| GET /products | ✅ (仅上架) | - | ✅ (全部) |
| GET /products/{id} | ✅ (仅上架) | - | ✅ |
| POST /products | - | - | ✅ |
| PUT /products/{id} | - | - | ✅ |
| DELETE /products/{id} | - | - | ✅ |
| PUT /products/{id}/shelf | - | - | ✅ |
| POST /upload | - | - | ✅ |

> **说明**：「公开」= 无需认证即可访问，包含未登录用户和 customer。
