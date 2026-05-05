# Outline Webhook 外发机制分析报告

## 1. 事件订阅机制

### 1.1 订阅模型

Webhook 订阅信息存储在 `WebhookSubscription` 模型中，核心字段包括：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 订阅名称 |
| `url` | string | 回调 URL |
| `enabled` | boolean | 是否启用 |
| `events` | string[] | 订阅的事件列表 |
| `secret` | string | 签名密钥（加密存储） |
| `teamId` | string | 所属团队 |

**关键代码位置**：`server/models/WebhookSubscription.ts`

### 1.2 事件匹配逻辑

订阅支持灵活的事件匹配：

1. **通配符订阅**：使用 `"*"` 订阅所有事件
2. **精确匹配**：如 `"users.create"`
3. **前缀匹配**：如 `"documents."` 匹配所有以该前缀开头的事件

**匹配方法**（`WebhookSubscription.validForEvent`）：

```typescript
public validForEvent = (event: Event): boolean => {
  if (this.events.length === 1 && this.events[0] === "*") {
    return true;
  }

  for (const e of this.events) {
    if (e === event.name || event.name.startsWith(e + ".")) {
      return true;
    }
  }

  return false;
};
```

**关键代码位置**：`server/models/WebhookSubscription.ts:136-148`

### 1.3 事件处理流程

1. **事件处理器注册**：`WebhookProcessor` 监听所有事件（`applicableEvents: ["*"]`）
2. **订阅筛选**：查询团队内所有启用的订阅，筛选出匹配当前事件的订阅
3. **任务调度**：为每个匹配的订阅创建 `DeliverWebhookTask` 并调度执行

**关键代码位置**：`plugins/webhooks/server/processors/WebhookProcessor.ts`

### 1.4 订阅限制

- 每个团队最多可创建 `WebhookSubscriptionValidation.maxSubscriptions` 个订阅
- 需要管理员权限才能创建/更新/删除订阅

**关键代码位置**：
- `server/models/WebhookSubscription.ts:96-106`（数量限制）
- `plugins/webhooks/server/api/webhookSubscriptions.ts`（权限控制）

## 2. 请求签名机制

### 2.1 签名算法

使用 **HMAC-SHA256** 算法生成签名，格式为：

```
t={timestamp},s={signature}
```

其中：
- `t`：当前时间戳（毫秒）
- `s`：HMAC-SHA256 签名值（十六进制）

**签名生成逻辑**：

```typescript
public signature = (payload: string) => {
  if (isNil(this.secret)) {
    return;
  }

  const signTimestamp = Date.now();

  const signature = crypto
    .createHmac("sha256", this.secret)
    .update(`${signTimestamp}.${payload}`)
    .digest("hex");

  return `t=${signTimestamp},s=${signature}`;
};
```

**关键代码位置**：`server/models/WebhookSubscription.ts:157-170`

### 2.2 签名验证流程

接收方验证签名时应：

1. 从请求头 `Outline-Signature` 解析 `t`（时间戳）和 `s`（签名值）
2. 验证时间戳与当前时间的差值（防止重放攻击）
3. 使用相同的密钥和算法重新计算签名
4. 比较计算出的签名与请求中的签名是否一致

### 2.3 安全特性

1. **密钥加密存储**：`secret` 字段使用 `@Encrypted` 装饰器加密存储在数据库中
2. **密钥轮换**：提供 `rotateSecret()` 方法用于生成新密钥
3. **时间戳防重放**：签名包含时间戳，接收方可验证时效性

**密钥生成格式**：`ol_whs_{32位随机字符串}`

**关键代码位置**：`server/models/WebhookSubscription.ts:113-115`

## 3. 投递失败处理机制

### 3.1 投递记录

每次 webhook 投递都会创建 `WebhookDelivery` 记录，包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `status` | string | 状态：pending/success/failed |
| `statusCode` | number | HTTP 状态码 |
| `requestBody` | JSON | 请求体 |
| `requestHeaders` | JSON | 请求头 |
| `responseBody` | string | 响应体（最多1KB） |
| `responseHeaders` | JSON | 响应头 |

**关键代码位置**：`server/models/WebhookDelivery.ts`

### 3.2 失败检测

投递失败的情况包括：

1. **HTTP 非 2xx 响应**：`response.ok` 为 false
2. **网络错误**：`FetchError` 等异常
3. **超时**：请求超时时间为 5 秒

**请求配置**：

```typescript
response = await fetch(subscription.url, {
  method: "POST",
  headers: requestHeaders,
  body: JSON.stringify(requestBody),
  redirect: "error",
  timeout: 5000,
});
```

**关键代码位置**：`plugins/webhooks/server/tasks/DeliverWebhookTask.ts:758-764`

### 3.3 自动禁用策略

当 webhook 持续失败时，系统会自动禁用订阅：

#### 触发条件

1. **时间窗口**：默认 24 小时（可通过 `WEBHOOK_FAILURE_TIME_WINDOW` 环境变量调整）
2. **失败率阈值**：默认 80%（可通过 `WEBHOOK_FAILURE_RATE_THRESHOLD` 环境变量调整）
3. **最小数据点**：至少 10 次投递（`MIN_DELIVERIES_FOR_ANALYSIS = 10`）

#### 判定逻辑

```typescript
private async checkAndDisableSubscription(subscription: WebhookSubscription) {
  // 计算分析时间窗口
  const timeWindowSeconds = env.WEBHOOK_FAILURE_TIME_WINDOW;
  const failureRateThreshold = env.WEBHOOK_FAILURE_RATE_THRESHOLD;
  const timeWindowStart = new Date(Date.now() - timeWindowSeconds * 1000);

  // 获取时间窗口内的所有投递记录
  const deliveriesInWindow = await WebhookDelivery.findAll({
    where: {
      webhookSubscriptionId: subscription.id,
      createdAt: { [Op.gte]: timeWindowStart },
    },
    order: [["createdAt", "DESC"]],
  });

  // 计算失败率
  const failedDeliveries = deliveriesInWindow.filter(
    (delivery) => delivery.status === "failed"
  );
  const failureRate = (failedDeliveries.length / deliveriesInWindow.length) * 100;

  // 检查是否超过阈值
  if (
    failureRate >= failureRateThreshold &&
    deliveriesInWindow.length >= DeliverWebhookTask.MIN_DELIVERIES_FOR_ANALYSIS
  ) {
    // 禁用订阅
    await subscription.disable();
    // 发送邮件通知
    await new WebhookDisabledEmail({ ... }).schedule();
  }
}
```

**关键代码位置**：`plugins/webhooks/server/tasks/DeliverWebhookTask.ts:817-896`

### 3.4 环境变量配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `WEBHOOK_FAILURE_TIME_WINDOW` | 86400（24小时） | 失败分析时间窗口（秒） |
| `WEBHOOK_FAILURE_RATE_THRESHOLD` | 80 | 触发禁用的失败率阈值（百分比） |
| `ALLOWED_PRIVATE_IP_ADDRESSES` | 空 | 允许访问的私有 IP 地址列表 |

**关键代码位置**：`server/env.ts:782-810`

### 3.5 投递记录清理

系统会定期清理过期的投递记录：

- **清理周期**：每天执行（`TaskInterval.Day`）
- **保留期限**：7 天
- **任务类型**：`CleanupWebhookDeliveriesTask`

**关键代码位置**：`plugins/webhooks/server/tasks/CleanupWebhookDeliveriesTask.ts`

## 4. 整体架构

### 4.1 数据流

```
事件触发 → WebhookProcessor → 订阅筛选 → DeliverWebhookTask → HTTP POST → 外部服务
              ↓
         WebhookDelivery（记录投递状态）
              ↓
         失败率监控 → 自动禁用 → 邮件通知
```

### 4.2 核心组件

| 组件 | 文件位置 | 职责 |
|------|----------|------|
| WebhookSubscription | server/models/WebhookSubscription.ts | 订阅模型，事件匹配，签名生成 |
| WebhookDelivery | server/models/WebhookDelivery.ts | 投递记录模型 |
| WebhookProcessor | plugins/webhooks/server/processors/WebhookProcessor.ts | 事件监听与订阅筛选 |
| DeliverWebhookTask | plugins/webhooks/server/tasks/DeliverWebhookTask.ts | 执行投递，失败监控 |
| CleanupWebhookDeliveriesTask | plugins/webhooks/server/tasks/CleanupWebhookDeliveriesTask.ts | 定期清理投递记录 |
| webhookSubscriptions API | plugins/webhooks/server/api/webhookSubscriptions.ts | 订阅管理 API |

### 4.3 支持的事件类型

Webhook 支持的主要事件类型包括：

- **用户事件**：`users.create`, `users.signin`, `users.signout`, `users.update`, `users.suspend`, `users.activate`, `users.delete`, `users.invite`, `users.promote`, `users.demote`, `users.invite_accepted`
- **文档事件**：`documents.create`, `documents.publish`, `documents.unpublish`, `documents.delete`, `documents.permanent_delete`, `documents.archive`, `documents.unarchive`, `documents.restore`, `documents.move`, `documents.update`, `documents.title_change`, `documents.add_user`, `documents.remove_user`, `documents.add_group`, `documents.remove_group`
- **集合事件**：`collections.create`, `collections.update`, `collections.delete`, `collections.move`, `collections.permission_changed`, `collections.archive`, `collections.restore`, `collections.add_user`, `collections.remove_user`, `collections.add_group`, `collections.remove_group`
- **其他事件**：`revisions.create`, `fileOperations.*`, `groups.*`, `integrations.*`, `pins.*`, `stars.*`, `shares.*`, `comments.*`, `attachments.*`, `views.create`, `webhookSubscriptions.*`

**关键代码位置**：`plugins/webhooks/server/tasks/DeliverWebhookTask.ts:108-267`

## 5. 技术特点与限制

### 5.1 优点

1. **灵活的事件订阅**：支持通配符和前缀匹配
2. **安全的签名机制**：HMAC-SHA256 签名，密钥加密存储
3. **自动故障保护**：高失败率时自动禁用订阅并通知
4. **完整的审计追踪**：所有投递记录都被保存
5. **异步处理**：使用任务队列，不阻塞主流程

### 5.2 限制

1. **无重试机制**：当前实现不支持投递失败后的自动重试
2. **固定超时**：请求超时固定为 5 秒，不可配置
3. **响应体限制**：最多只保存 1KB 的响应体
4. **无去重机制**：同一事件可能因重试等原因多次发送

### 5.3 可优化建议

1. **实现重试机制**：考虑加入指数退避的重试策略
2. **可配置的超时**：允许为每个订阅配置不同的超时时间
3. **去重机制**：为事件添加唯一 ID，防止重复处理
4. **更细粒度的失败监控**：区分网络错误、超时、服务端错误等不同失败类型

## 6. 代码参考

- **订阅模型**：`server/models/WebhookSubscription.ts`
- **投递记录模型**：`server/models/WebhookDelivery.ts`
- **事件处理器**：`plugins/webhooks/server/processors/WebhookProcessor.ts`
- **投递任务**：`plugins/webhooks/server/tasks/DeliverWebhookTask.ts`
- **清理任务**：`plugins/webhooks/server/tasks/CleanupWebhookDeliveriesTask.ts`
- **订阅 API**：`plugins/webhooks/server/api/webhookSubscriptions.ts`
- **环境变量**：`server/env.ts`
- **事件辅助**：`shared/utils/EventHelper.ts`
