# KPOS 日志字段参考

## 日志文件命名

```
appserver-MM-DD-YYYY-N.log
```

- `N` 为当日滚动序号，数字越大通常越晚
- 另有 `appserver.log` 可能为当前活跃文件

## orderNumber 出现位置

| 模式 | 示例场景 |
|------|----------|
| `orderNumber:80` | `DevicePrintingService.print()` |
| `orderNumber>80<` | SOAP `FetchOrderResponse` XML |
| `Order Number【80】` | `SyncOnlineOrder.run()` |
| `Order->127675:80` | `OrderServiceHelper.logOrderChangeRecord()` |

## 关键 SOAP / JSON 字段

**FetchOrderResponse（订单快照）**

```xml
<order>
  <id>127675</id>
  <orderNumber>80</orderNumber>
  <status>PAID</status>
  <type>ONLINE_DELIVERY</type>
  <refNumber>6a4467369375ee4bcd6ef966</refNumber>
  ...
</order>
```

**SQS CREATE 消息（在线订单入站）**

```json
{
  "operationType": "CREATE",
  "order": {
    "orderStatus": 0,
    "type": "ONLINE_DELIVERY",
    "payments": [{ "amount": 81.76, "transaction_status": 1 }],
    "orderItems": [{ "status": "ORDERED" }],
    "sendToKitchen": { "send": true }
  }
}
```

**SaveOrder 取消**

```xml
<app:status>CANCELED_AFTER_SENT_TO_KITCHEN</app:status>
<app:printTicketWhenVoid>true</app:printTicketWhenVoid>
```

## 状态层级

```
订单主状态 (order.status)
├── PAID
├── PRINTED
├── SUBMITTED
└── CANCELED_AFTER_SENT_TO_KITCHEN

菜品状态 (orderItems[].status)
├── ORDERED
└── SENT_TO_KITCHEN

顾客显示屏 (customerDisplayRecord)
├── status: NOT_READY | DONE
└── kitchenStatus: IN_KITCHEN | DONE

在线同步 orderStatus（数值）
├── 0  — 新建
└── -2 — 取消类
```

## 操作人字段速查

| 日志关键字 | 提取方式 |
|------------|----------|
| `logOrderChangeRecord` | `User [Online]`、`User [Cindy]` |
| `KPosServlet.logInfo` | `user ID:55,user name：Cindy` |
| `LockManager.lock` | `user :Cindy`（常无 userId，需结合 SOAP `userId`） |
| `SaveOrderType` | `currentUserId` + 同响应 `serverName` |
| `FetchOrderResponse` | `userId` + `serverName`（多为下单服务员，非当前操作者） |
| `userAuth` in SOAP | `<app:userId>55</app:userId>` |

在线订单创建时 `serverName=Online`、`userId=48` 通常为系统在线账户，非店内店员。

## 常用 grep 命令（只读）

```bash
# 按订单号定位
rg "orderNumber:80|Order Number【80】" 2026-06/appserver-06-30*

# 按 orderId 全链路
rg "127675" 2026-06/appserver-06-30*

# 状态变更
rg "CANCELED_AFTER_SENT_TO_KITCHEN|logOrderChangeRecord.*127675" 2026-06/
```

## 商户标识

本批日志商户：`M000014266`（日志路径或 SQS `merchantId` 可见）
