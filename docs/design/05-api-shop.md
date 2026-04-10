# 门店模块 API 设计

## 1. 概述

门店模块管理单店信息，包括：
- 门店基本信息（店名、地址、电话、Logo）
- 营业时间（展示文本，不参与状态判定）
- 营业状态（管理员手动开关）
- 门店公告

**单店模型**：全局仅一条门店记录。

## 2. 接口列表

### 2.1 GET /api/v1/shop — 获取门店信息

**权限**：公开（C端）、staff、admin

**成功响应** 200：

```json
{
  "id": "uuid",
  "name": "Ideal Cafe",
  "address": "XX 市 XX 区 XX 路 XX 号",
  "phone": "138-XXXX-XXXX",
  "logo_url": "https://...",
  "business_hours": "周一至周五 08:00-20:00，周六日 09:00-21:00",
  "announcement": "今日新品：埃塞俄比亚水洗耶加雪菲",
  "announcement_enabled": true,
  "is_open": true,
  "created_at": "2026-01-01T00:00:00+08:00",
  "updated_at": "2026-04-10T09:00:00+08:00"
}
```

### 2.2 GET /api/v1/shop/status — 获取营业状态

**权限**：公开

轻量接口，C端菜单页轮询营业状态变化。

**成功响应** 200：

```json
{
  "is_open": true,
  "status_label": "营业中",
  "announcement": "今日新品：埃塞俄比亚水洗耶加雪菲",
  "announcement_enabled": true
}
```

**status_label 映射**：

| is_open | status_label |
|---------|-------------|
| true | 营业中 |
| false | 已打烊 |

### 2.3 PUT /api/v1/shop — 更新门店信息（A端）

**权限**：admin

**请求体**：

```json
{
  "name": "Ideal Cafe",
  "address": "XX 市 XX 区 XX 路 XX 号",
  "phone": "138-XXXX-XXXX",
  "logo_url": "https://...",
  "business_hours": "周一至周五 08:00-20:00，周六日 09:00-21:00",
  "announcement": "今日新品：埃塞俄比亚水洗耶加雪菲",
  "announcement_enabled": true
}
```

**校验规则**：
- name：1-100 字符，必填
- address：1-200 字符，必填
- phone：选填
- logo_url：选填，URL 格式
- business_hours：选填，最大 200 字符
- announcement：选填，最大 500 字符
- announcement_enabled：布尔

**成功响应** 200：同 GET /api/v1/shop 响应

### 2.4 PUT /api/v1/shop/toggle-open — 切换营业状态（A端）

**权限**：admin

**请求体**：

```json
{
  "is_open": false
}
```

**处理逻辑**：
- `is_open = false`：打烊，顾客无法下单
- `is_open = true`：营业，顾客可正常下单
- 手动打烊时提示「打烊后顾客将无法下单，确认打烊？」（前端二次确认）

**成功响应** 200：

```json
{
  "is_open": false,
  "status_label": "已打烊"
}
```

### 2.5 PUT /api/v1/shop/announcement — 更新公告（A端）

**权限**：admin

独立接口，方便单独更新公告而不需要提交完整门店信息。

**请求体**：

```json
{
  "announcement": "今日新品：埃塞俄比亚水洗耶加雪菲",
  "announcement_enabled": true
}
```

**成功响应** 200：

```json
{
  "announcement": "今日新品：埃塞俄比亚水洗耶加雪菲",
  "announcement_enabled": true
}
```

## 3. 权限边界汇总

| 接口 | 公开 | customer | staff | admin |
|------|------|----------|-------|-------|
| GET /shop | ✅ | - | - | - |
| GET /shop/status | ✅ | - | - | - |
| PUT /shop | - | - | - | ✅ |
| PUT /shop/toggle-open | - | - | - | ✅ |
| PUT /shop/announcement | - | - | - | ✅ |

> **说明**：「公开」= 无需认证即可访问，包含未登录用户和 customer。
