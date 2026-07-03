---
name: kpos-log-order-lookup
description: >-
  Locate and analyze KPOS/Menusifu appserver order lifecycle from log files by
  orderNumber (daily sequence). Use when the user asks to find an order in logs,
  trace order status changes, investigate online delivery orders, or analyze
  M000014266 / wisdomount appserver logs.
---

# KPOS 日志查订单

在 KPOS（wisdomount）`appserver` 日志中，按**当日流水号 orderNumber** 定位订单并还原完整状态变更链路。

## 触发场景

- 用户给出订单号（如「订单 80」「orderNumber 80」）并要求查日志
- 需要按时间顺序列出订单状态变更，并附日志位置
- 分析在线外卖（ONLINE_DELIVERY）、支付、送厨、取消等流程

## 核心概念

| 字段 | 含义 |
|------|------|
| `orderNumber` | 当日流水号（用户口中的「订单号」） |
| `orderId` | 系统内部 ID（如 127675），全链路追踪的主键 |
| `refNumber` / `cloudId` | 云端引用号，在线订单第二追踪键 |

**禁止**单独搜索裸数字（如 `\b80\b`）——会命中请求耗时、金额、行号、staffId 等噪声。

## 查询流程

```
Task Progress:
- [ ] Step 1: 定位日期日志文件
- [ ] Step 2: 用 orderNumber 模式搜索
- [ ] Step 3: 建立 orderNumber ↔ orderId 映射
- [ ] Step 4: 用 orderId/refNumber 追踪全链路
- [ ] Step 5: 识别操作人与操作类型
- [ ] Step 6: 按表格格式输出
```

### Step 1: 定位日期日志文件

日志目录结构示例：

```
YYYY-MM/appserver-MM-DD-YYYY-N.log
```

- 按用户指定日期进入对应月份目录（如 `2026-06/`）
- 同一天可能有多个滚动文件（`-1.log` ~ `-5.log`），需**全部搜索**

### Step 2: 用 orderNumber 模式搜索

在目标日期的 `appserver-*` 文件中依次尝试（`N` = 用户给的订单号）：

```
orderNumber:N,
orderNumber>N<
orderNumber":"N"
Order Number【N】
Order Number[N]
orderNum>N<
```

优先命中 `DevicePrintingService.print()`、`SyncOnlineOrder`、`FetchOrderResponse` 等行。

### Step 3: 建立 orderNumber ↔ orderId 映射

从命中行提取：

```
orderId:127675,orderNumber:80,...
Order ID【127675】,Order Number【80】
<id>127675</id>...<orderNumber>80</orderNumber>
```

确认后，后续搜索以 **orderId** 为主。

### Step 4: 用 orderId / refNumber 追踪全链路

对确认的 `orderId` 搜索：

```
{orderId}
refNumber:{cloudId}   # 若已知
```

重点日志类与状态含义：

| 日志来源 | 典型含义 |
|----------|----------|
| `AwsServiceManager` + `operationType:CREATE` | 在线订单创建入站 |
| `ThirdPartyPlatformOrderService.createOrder` + `支付结束清桌` | 创建完成，已含在线支付 → **PAID** |
| `KitchenTicketGenerator.print` / `send to kitchen success` | 送厨，菜品 → **SENT_TO_KITCHEN** |
| `SyncOnlineOrder` + `customerDisplayRecord` | 顾客屏：`NOT_READY`/`IN_KITCHEN`/`DONE` |
| `FetchOrderResponse` / `SaveOrderResponse` Payload | 订单主状态快照 |
| `SaveOrderType` + `CANCELED_AFTER_SENT_TO_KITCHEN` | 送厨后取消 |
| `logOrderChangeRecord` | 人工/系统修改记录 |
| `DatahubClient.publishOneOrder` | 数据上报时间点 |

常见订单主状态：`PAID`、`PRINTED`、`SUBMITTED`、`CANCELED_AFTER_SENT_TO_KITCHEN`

常见菜品状态：`ORDERED` → `SENT_TO_KITCHEN`

在线同步 `orderStatus` 数值示例：`0`（新建）、`-2`（取消类）

### Step 5: 识别操作人与操作类型

**操作人**从以下日志字段提取（按优先级）：

| 来源 | 字段示例 |
|------|----------|
| `logOrderChangeRecord` | `modified by User [Online]` / `User [Cindy]` |
| `KPosServlet.logInfo` | `user ID:55,user name：Cindy` |
| `LockManager` | `locked by ... user :Cindy` |
| `SaveOrderType` Payload | `<app:currentUserId>55</app:currentUserId>` + `serverName` |
| `FetchOrderResponse` | `<userId>48</userId><serverName>Online</serverName>` |
| SQS / 后台线程 | 无用户 → 填 **系统**（注明来源，如 `SQS CREATE`、`pool-40-thread-1`） |

格式：`姓名 (userId=N)`；无法确定姓名时写 `userId=N`；完全无用户写 `系统`。

**操作类型**使用统一枚举（勿每行自创措辞）：

| 操作类型 | 适用场景 |
|----------|----------|
| 订单创建 | SQS CREATE、本地 createOrder |
| 支付 | 在线支付入账、SavePayment、支付结束清桌 |
| 送厨/打印 | KitchenTicket、PrintTask、send to kitchen |
| 状态同步 | SyncOnlineOrder、MessagePublisher |
| 订单查询 | FetchOrder、ListOrders、order/recall |
| 订单锁定 | AddLock / LockManager.lock |
| 订单解锁 | RemoveLock / releaseAllLocks |
| 订单修改 | SaveOrder（非取消类字段变更） |
| 订单取消 | status → CANCELED_* |
| 数据上报 | DatahubClient.publishOneOrder |
| 显示屏更新 | customerDisplayRecord 状态变更 |

### Step 6: 表格输出（默认格式）

**必须用 Markdown 表格**，按时间升序，**仅包含有业务含义的事件**（跳过重复 FetchOrder、纯轮询、重复 SyncOnlineOrder 等同态日志）。

| 时间 | 操作类型 | 具体操作内容 | 操作人 | 关联日志位置 |
|------|----------|--------------|--------|--------------|
| 2026-06-30 18:02:49 | 订单创建 | 接收 SQS CREATE，在线外卖订单入站；菜品状态 ORDERED；含在线支付 $81.76 | 系统 (SQS) | `2026-06/appserver-06-30-2026-4.log:9666` |
| 2026-06-30 18:02:49 | 支付 | 创建订单 127675，支付结束清桌；订单主状态 **PAID** | 系统 (Online, userId=48) | `2026-06/appserver-06-30-2026-4.log:9670` |
| 2026-06-30 18:02:50 | 送厨/打印 | 厨房单打印至 printer 26/25；菜品 → SENT_TO_KITCHEN | 系统 | `2026-06/appserver-06-30-2026-4.log:9677` |
| 2026-06-30 18:05:32 | 订单取消 | SaveOrder：PAID → **CANCELED_AFTER_SENT_TO_KITCHEN**；printTicketWhenVoid=true | Cindy (userId=55) | `2026-06/appserver-06-30-2026-4.log:11216` |

**列填写规则：**

1. **时间** — 日志行首时间戳，精确到秒
2. **操作类型** — 上表枚举之一
3. **具体操作内容** — 写清状态变化（前→后）、金额、关键参数；状态名用原文（如 `PAID`）
4. **操作人** — `姓名 (userId=N)` 或 `系统 (来源)`；不确定写 `未知`
5. **关联日志位置** — `` `路径:行号` ``；同一事件多行日志可在内容中注明，位置列写**主证据行**

表格后附：

- **订单摘要**（orderNumber、orderId、类型、客户、金额、最终状态，3～5 行）
- **状态流转简图**（可选 mermaid flowchart）
- **排查提示**（异常、缺失环节、建议下一步）

## 输出质量建议

向用户交付前自检：

- [ ] 是否去重（多次 SyncOnlineOrder 只保留首次/状态变化点）
- [ ] 取消/支付等关键节点是否都有操作人
- [ ] 「具体操作内容」是否含状态前后值，而非仅写类名
- [ ] 日志位置是否可一键定位（路径+行号）
- [ ] 是否区分订单主状态 / 菜品状态 / 顾客屏状态（必要时在内容列注明层级）

**可主动提供的增值信息**（用户未禁止时）：

1. **耗时分析** — 关键节点间隔（如创建→送厨、送厨→取消）
2. **支付明细** — paymentId、卡类型、金额、是否在线支付
3. **异常/错误** — 同订单 ERROR/WARN（如 Sync 时 NPE）
4. **重复操作统计** — 查询次数、同步次数（解释日志膨胀）
5. **跨文件索引** — 该 orderId 出现在哪些 log 文件及各自行数范围

## 搜索工具建议

- 用 `Grep` 的 `glob` 限定日期：`appserver-06-30*`
- 先 `orderNumber` 窄搜，再 `orderId` 宽搜
- 长行 Payload 用 `Read` 带 `offset` 读上下文，避免整文件加载
- 对 `orderId` 做 `output_mode: count` 确认覆盖哪些日志文件

## 反模式

| 不要做 | 原因 |
|--------|------|
| 只搜 `\b80\b` | 大量误匹配 |
| 只查一个 log 文件 | 订单可能跨滚动文件 |
| 只列 FetchOrder 快照 | 会漏创建、送厨、同步等中间步骤 |
| 混淆 orderNumber 与 orderId | 用户说的「订单号」通常是 orderNumber |

## 示例（订单号 80）

1. `orderNumber:80` 命中 → `orderId:127675`
2. `127675` 全链路 → 18:02:49 创建(PAID) → 18:02:50 送厨 → 18:05:32 取消
3. 主文件：`appserver-06-30-2026-4.log`；后续查询在 `-5.log`

更多字段与 Payload 片段说明见 [reference.md](reference.md)。
