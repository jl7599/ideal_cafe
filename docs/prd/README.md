## PRD 文档

产品需求文档位于 `docs/prd/`，按业务域纵向切分：

| 文件 | 内容 |
|------|------|
| 00-overview.md | 产品概览、角色权限、MVP 范围、全局交互规范、导航结构 |
| 01-auth.md | 微信登录(C端)、账号密码登录(S/A端)、角色路由守卫 |
| 02-menu.md | 菜单浏览(C端)、商品/分类管理(A端)、购物车 |
| 03-order.md | 下单流程、订单状态机(pending→accepted→preparing→ready→completed)、三端订单处理 |
| 04-shop.md | 门店信息展示(C端)、门店配置(A端)、营业状态控制 |
| 05-notification.md | 订单状态通知(C端)、新订单提醒(S端) |

需求 ID 编码规则：`{模块代号}-{角色}-{序号}`，模块 AUTH=01/MENU=02/ORDER=03/SHOP=04/NOTIF=05，角色 C/S/A/X。
