# 数据库设计

数据库：PostgreSQL，使用 SQLAlchemy 2.0 async 声明式模型。

## 1. ER 关系概览

```
users ──1:N── orders ──1:N── order_items
staff
categories ──1:N── products
shop (单行记录)
token_blacklist (独立表)
```

## 2. 表结构

### 2.1 users — C端用户

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, default uuid4 | 主键 |
| openid | VARCHAR(64) | UNIQUE, NOT NULL | 微信 openid |
| unionid | VARCHAR(64) | NULL | 微信 unionid，跨端识别用户，后续扩展用 |
| session_key | VARCHAR(128) | NULL | 微信 session_key，用于解密敏感数据 |
| nickname | VARCHAR(50) | NOT NULL | 昵称，默认「理想咖啡」+ 4位随机数 |
| avatar_url | VARCHAR(500) | NOT NULL, DEFAULT '' | 头像 URL，空字符串表示默认头像 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

**索引**：
- `idx_users_openid` ON (openid) — 微信登录查询
- `idx_users_unionid` ON (unionid) WHERE unionid IS NOT NULL — 跨端查询

### 2.2 staff — 店员/管理员

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, default uuid4 | 主键 |
| phone | VARCHAR(11) | UNIQUE, NOT NULL | 手机号 |
| password_hash | VARCHAR(128) | NOT NULL | bcrypt 哈希 |
| role | VARCHAR(10) | NOT NULL, CHECK IN ('staff', 'admin') | 角色 |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE | 是否可登录（删除=禁用） |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

**索引**：
- `idx_staff_phone` ON (phone) — 登录查询

**说明**：
- 删除店员 = 设置 `is_active = FALSE`，而非物理删除
- 初始管理员通过 `scripts/init_admin.py` 脚本创建

### 2.3 categories — 商品分类

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, default uuid4 | 主键 |
| name | VARCHAR(50) | NOT NULL | 分类名称 |
| icon | VARCHAR(500) | DEFAULT '' | 图标 URL |
| sort_order | INTEGER | NOT NULL, DEFAULT 0 | 排序，越小越靠前 |
| is_enabled | BOOLEAN | NOT NULL, DEFAULT TRUE | 是否启用 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

**约束**：
- 有商品关联的分类不可删除（应用层校验）

### 2.4 products — 商品

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, default uuid4 | 主键 |
| category_id | UUID | FK → categories.id, NOT NULL | 所属分类 |
| name | VARCHAR(100) | NOT NULL | 商品名称 |
| subtitle | VARCHAR(200) | DEFAULT '' | 副标题 |
| description | TEXT | DEFAULT '' | 详细描述 |
| price | INTEGER | NOT NULL | 价格，单位：分（28.00 元 = 2800） |
| image_url | VARCHAR(500) | NOT NULL | 商品图片 URL |
| sort_order | INTEGER | NOT NULL, DEFAULT 0 | 排序 |
| is_on_shelf | BOOLEAN | NOT NULL, DEFAULT TRUE | 是否上架 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

**索引**：
- `idx_products_category` ON (category_id) — 按分类查询
- `idx_products_shelf_sort` ON (is_on_shelf, sort_order) — C端列表查询

**说明**：
- 价格用整数存分，避免浮点精度问题
- 删除商品为物理删除，但已有订单不受影响（订单保存商品快照）

### 2.5 shop — 门店信息

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, default uuid4 | 主键 |
| name | VARCHAR(100) | NOT NULL | 店名 |
| address | VARCHAR(200) | NOT NULL | 地址 |
| phone | VARCHAR(20) | DEFAULT '' | 联系电话 |
| logo_url | VARCHAR(500) | DEFAULT '' | 门店 Logo URL |
| business_hours | VARCHAR(200) | NOT NULL, DEFAULT '' | 营业时间展示文本，如「周一至周五 08:00-20:00」 |
| announcement | TEXT | DEFAULT '' | 公告内容 |
| announcement_enabled | BOOLEAN | NOT NULL, DEFAULT FALSE | 公告是否启用 |
| is_open | BOOLEAN | NOT NULL, DEFAULT TRUE | 营业状态开关，TRUE=营业中，FALSE=已打烊 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

**说明**：
- 单店模型：全局仅一条记录
- `business_hours` 为纯展示文本，不参与营业状态判定
- 营业状态完全由 `is_open` 字段控制，管理员手动切换
- 初始记录通过数据库迁移脚本创建

### 2.6 orders — 订单

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, default uuid4 | 主键 |
| order_no | VARCHAR(18) | UNIQUE, NOT NULL | 订单号（yyyyMMddHHmmss + 4位随机） |
| user_id | UUID | FK → users.id, NOT NULL | 下单用户 |
| status | VARCHAR(15) | NOT NULL, DEFAULT 'pending' | 订单状态 |
| idempotency_key | UUID | UNIQUE, NOT NULL | 客户端提交的幂等键，同一次提交重试必须一致 |
| total_amount | INTEGER | NOT NULL | 合计金额，单位：分 |
| remark | VARCHAR(200) | DEFAULT '' | 整单备注 |
| business_date | DATE | NOT NULL | 订单营业日，用于约束当日取餐码唯一 |
| pickup_code | VARCHAR(4) | NOT NULL | 4位数字取餐码 |
| cancel_reason | VARCHAR(200) | DEFAULT '' | 取消原因 |
| reject_reason | VARCHAR(200) | DEFAULT '' | 拒绝原因 |
| estimated_wait_minutes | INTEGER | NULL | 预估等待时间（分钟） |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

**status 取值**：pending, accepted, preparing, ready, completed, cancelled, rejected

**索引**：
- `idx_orders_order_no` ON (order_no) — 订单号查询
- `uq_orders_idempotency_key` UNIQUE (idempotency_key) — 下单幂等保证
- `uq_orders_business_date_pickup_code` UNIQUE (business_date, pickup_code) — 同一营业日取餐码唯一
- `idx_orders_user_status` ON (user_id, status) — 用户订单列表
- `idx_orders_status_created` ON (status, created_at DESC) — S端按状态筛选

**状态流转约束**（应用层校验）：

```
pending → accepted → preparing → ready → completed
pending → cancelled（用户取消）
pending → rejected（店员拒绝，必须填 reject_reason）
```

### 2.7 order_items — 订单商品快照

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, default uuid4 | 主键 |
| order_id | UUID | FK → orders.id, NOT NULL | 所属订单 |
| product_id | UUID | FK → products.id ON DELETE SET NULL, NULL | 商品 ID，商品删除后自动置 NULL（快照仍保留名称和价格） |
| product_name | VARCHAR(100) | NOT NULL | 商品名称快照 |
| product_price | INTEGER | NOT NULL | 商品单价快照，单位：分 |
| quantity | INTEGER | NOT NULL | 数量 |
| subtotal | INTEGER | NOT NULL | 小计 = product_price × quantity，单位：分 |

**索引**：
- `idx_order_items_order` ON (order_id) — 查询订单商品
- `idx_order_items_product` ON (product_id) WHERE product_id IS NOT NULL — 按商品追溯订单

**说明**：
- `product_id` 允许 NULL：商品被物理删除后快照仍保留名称和价格，但 product_id 置 NULL
- 保存商品快照，商品后续被删除或改价不影响已有订单
- `product_id` 用于"再来一单"时精确匹配在架商品
- `subtotal` 冗余存储，避免计算

### 2.8 token_blacklist — Token 黑名单

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, default uuid4 | 主键 |
| token_jti | VARCHAR(64) | UNIQUE, NOT NULL | refresh_token 的唯一标识（jti 声明） |
| expires_at | TIMESTAMPTZ | NOT NULL | Token 原始过期时间，过期后可清理记录 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 加入黑名单时间 |

**索引**：
- `idx_token_blacklist_jti` ON (token_jti) — 刷新 Token 时查询

**说明**：
- 登出或修改密码时将 refresh_token 的 jti 加入此表
- 刷新 Token 时检查 jti 是否在黑名单中
- `expires_at` 用于定期清理过期记录（Token 过期后黑名单记录无意义）

## 3. 营业状态判定规则

营业状态完全由管理员手动控制，`shop.is_open = TRUE` 即营业中，`FALSE` 即已打烊。无自动时间判定。

## 4. 订单号生成规则

```
order_no = yyyyMMddHHmmss + 4位随机数字
示例：202604091430258827
```

生成逻辑：
1. 获取当前时间戳 `yyyyMMddHHmmss`
2. 生成 4 位随机数 `rand(1000, 9999)`
3. 拼接为 18 位字符串
4. 唯一性由数据库 UNIQUE 约束保证，冲突时重试

## 5. 取餐码生成规则

```
pickup_code = 4位随机数字 (0001-9999)
```

生成逻辑：
1. 生成 4 位随机数
2. 按当前订单的 `business_date` 写入 `(business_date, pickup_code)` 唯一约束
3. 若唯一约束冲突则重新生成并重试
