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
2. **精确匹配**：如 `"users.create"` 只匹配该具体事件
3. **前缀匹配**：如 `"documents"` 匹配所有以 `"documents."` 开头的事件（如 `"documents.create"`、`"documents.update"` 等）

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

### 1.3 订阅限制

- 每个团队最多可创建 `WebhookSubscriptionValidation.maxSubscriptions` 个订阅
- 需要管理员权限才能创建/更新/删除订阅

**关键代码位置**：
- `server/models/WebhookSubscription.ts:96-106`（数量限制）
- `plugins/webhooks/server/api/webhookSubscriptions.ts`（权限控制）

## 2. 事件入队到实际投递的完整调度路径

### 2.1 四层队列架构

Outline 的事件处理采用了四层队列架构，确保事件从产生到投递的可靠流转：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            第一层：globalEventQueue                            │
│  ┌─────────────┐     ┌─────────────────┐     ┌──────────────────────────┐  │
│  │ 事件产生    │────▶│ 事件入队         │────▶│ 分发给各 Processor       │  │
│  │ (Event.save)│     │ (@AfterSave)    │     │ (applicableEvents 匹配) │  │
│  └─────────────┘     └─────────────────┘     └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           第二层：processorEventQueue                          │
│  ┌─────────────────┐     ┌─────────────────┐     ┌──────────────────────┐  │
│  │ WebhookProcessor│────▶│ 筛选订阅         │────▶│ 调度 DeliverWebhookTask│  │
│  │ (监听 * 事件)   │     │ (validForEvent) │     │ (加入 taskQueue)      │  │
│  └─────────────────┘     └─────────────────┘     └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              第三层：taskQueue                                  │
│  ┌─────────────────┐     ┌─────────────────┐     ┌──────────────────────┐  │
│  │ 任务执行         │────▶│ 执行 HTTP POST  │────▶│ 记录投递结果          │  │
│  │ (Worker 消费)   │     │ (sendWebhook)   │     │ (WebhookDelivery)    │  │
│  └─────────────────┘     └─────────────────┘     └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 详细调度流程

#### 阶段 1：事件产生与入队

**触发点**：任何业务操作保存时，`Event` 模型的 `@AfterSave` 钩子触发

```typescript
// server/models/Event.ts:84-99
@AfterSave
static async enqueue(
  model: Event,
  options: SaveOptions<InferAttributes<Event>>
) {
  if (options.transaction) {
    // 事务提交后才入队
    (options.transaction.parent || options.transaction).afterCommit(
      () => void globalEventQueue().add(model)
    );
    return;
  }
  void globalEventQueue().add(model);
}
```

**队列配置**：
- `globalEventQueue` 配置：`attempts: 5`，`backoff: exponential`，初始延迟 1 秒
- **关键点**：如果在事务中，事件会在事务提交后才入队，确保数据一致性

**关键代码位置**：
- `server/models/Event.ts:84-99`（事件入队）
- `server/queues/index.ts:5-16`（globalEventQueue 配置）

#### 阶段 2：全局事件分发

**处理者**：Worker 服务的 `globalEventQueue.process()` 回调

```typescript
// server/services/worker.ts:21-79
globalEventQueue()
  .process(env.WORKER_CONCURRENCY_EVENTS, async function (job) {
    const event = job.data as Event;
    
    // 遍历所有注册的 Processor
    for (const name in processors) {
      const ProcessorClass = processors[name];
      
      // WebhookProcessor 的 applicableEvents 是 ["*"]，所以会匹配所有事件
      if (ProcessorClass.applicableEvents.includes(event.name) ||
          ProcessorClass.applicableEvents.includes("*")) {
        await processorEventQueue().add({ event, name });
      }
    }
  })
```

**关键代码位置**：`server/services/worker.ts:21-79`

#### 阶段 3：Processor 处理与订阅筛选

**处理者**：`WebhookProcessor.perform()`

```typescript
// plugins/webhooks/server/processors/WebhookProcessor.ts:6-33
export default class WebhookProcessor extends BaseProcessor {
  static applicableEvents: ["*"] = ["*"];  // 监听所有事件

  async perform(event: Event) {
    if (!event.teamId) {
      return;
    }

    // 查询团队内所有启用的订阅
    const webhookSubscriptions = await WebhookSubscription.findAll({
      where: {
        enabled: true,
        teamId: event.teamId,
      },
    });

    // 筛选匹配当前事件的订阅
    const applicableSubscriptions = webhookSubscriptions.filter((webhook) =>
      webhook.validForEvent(event)
    );

    // 为每个匹配的订阅调度投递任务
    await Promise.all(
      applicableSubscriptions.map((subscription) =>
        new DeliverWebhookTask().schedule({
          event,
          subscriptionId: subscription.id,
        })
      )
    );
  }
}
```

**队列配置**：
- `processorEventQueue` 配置：`attempts: 5`，`backoff: exponential`，初始延迟 10 秒

**关键代码位置**：
- `plugins/webhooks/server/processors/WebhookProcessor.ts:6-33`
- `server/queues/index.ts:18-30`（processorEventQueue 配置）

#### 阶段 4：实际投递执行

**处理者**：`DeliverWebhookTask.perform()`

```typescript
// plugins/webhooks/server/tasks/DeliverWebhookTask.ts:90-267
public async perform({ subscriptionId, event }: Props) {
  const subscription = await WebhookSubscription.findByPk(subscriptionId, {
    rejectOnEmpty: true,
  });

  if (!subscription.enabled) {
    // 订阅已被禁用，直接返回
    return;
  }

  // 根据事件类型调用对应的 handler
  switch (event.name) {
    case "users.signin":
    case "users.signout":
      // ...
      await this.handleUserEvent(subscription, event);
      return;
    // ... 其他事件类型
  }
}

// 实际发送逻辑
private async sendWebhook({ event, subscription, payload }: {...}) {
  // 1. 创建投递记录（状态为 pending）
  const delivery = await WebhookDelivery.create({
    webhookSubscriptionId: subscription.id,
    status: "pending",
  });

  try {
    // 2. 构建请求体和请求头
    requestBody = presentWebhook({ event, delivery, payload });
    requestHeaders = {
      "Content-Type": "application/json",
    };

    // 3. 生成签名（如果有 secret）
    const signature = subscription.signature(JSON.stringify(requestBody));
    if (signature) {
      requestHeaders["Outline-Signature"] = signature;
    }

    // 4. 发送 HTTP POST 请求
    response = await fetch(subscription.url, {
      method: "POST",
      headers: requestHeaders,
      body: JSON.stringify(requestBody),
      redirect: "error",
      timeout: 5000,  // 固定 5 秒超时
    });
    status = response.ok ? "success" : "failed";
  } catch (err) {
    // 捕获所有异常，记录为 failed
    status = "failed";
  }

  // 5. 更新投递记录状态
  await delivery.update({
    status,
    statusCode: response ? response.status : null,
    // ... 其他字段
  });

  // 6. 如果失败，检查是否需要禁用订阅
  if (status === "failed") {
    await this.checkAndDisableSubscription(subscription);
  }
}
```

**队列配置**：
- `taskQueue` 配置：`attempts: 5`，`backoff: exponential`，初始延迟 10 秒
- **关键点**：虽然 BaseTask 配置了 5 次重试，但由于 `sendWebhook()` 内部捕获了所有异常，**实际上不会触发队列级别的重试**

**关键代码位置**：
- `plugins/webhooks/server/tasks/DeliverWebhookTask.ts:90-267`（perform 方法）
- `plugins/webhooks/server/tasks/DeliverWebhookTask.ts:724-815`（sendWebhook 方法）
- `server/queues/index.ts:42-54`（taskQueue 配置）

### 2.3 队列关系总结

| 队列名称 | 职责 | 重试配置 | 消费端 |
|----------|------|----------|--------|
| `globalEventQueue` | 接收所有事件，分发给各 Processor | 5 次，指数退避（1秒起） | worker.ts |
| `processorEventQueue` | 执行各 Processor 的业务逻辑 | 5 次，指数退避（10秒起） | worker.ts |
| `taskQueue` | 执行异步任务（包括 webhook 投递） | 5 次，指数退避（10秒起） | worker.ts |
| `websocketQueue` | 处理 WebSocket 消息 | 无重试 | websocket 服务 |

## 3. 请求签名机制

### 3.1 签名算法

使用 **HMAC-SHA256** 算法生成签名，格式为：

```
t={timestamp},s={signature}
```

其中：
- `t`：当前时间戳（毫秒）
- `s`：HMAC-SHA256 签名值（十六进制）

**签名生成逻辑**：

```typescript
// server/models/WebhookSubscription.ts:157-170
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

### 3.2 关于时间戳的重要说明

**时间戳仅参与签名计算，服务端不做任何验证**

需要特别澄清：

1. **服务端行为**：时间戳 `t` 只是签名算法的一部分，用于增加签名的随机性和防止简单的重放攻击，但**服务端本身不会对时间戳进行任何验证**。

2. **接收方责任**：如果接收方需要防止重放攻击，**必须自己实现时间戳验证逻辑**：
   - 解析 `Outline-Signature` 中的 `t` 值
   - 验证时间戳与当前时间的差值（例如：拒绝超过 5 分钟的请求）
   - 这不是 Outline 服务端的职责

3. **签名验证流程（接收方应实现）**：
   1. 从请求头 `Outline-Signature` 解析 `t`（时间戳）和 `s`（签名值）
   2. **（可选，自行实现）** 验证时间戳与当前时间的差值（防止重放攻击）
   3. 使用相同的密钥和算法重新计算签名：`HMAC-SHA256(secret, `${t}.${payload}`)`
   4. 比较计算出的签名与请求中的签名是否一致

### 3.3 安全特性

1. **密钥加密存储**：`secret` 字段使用 `@Encrypted` 装饰器加密存储在数据库中
2. **密钥轮换**：提供 `rotateSecret()` 方法用于生成新密钥
3. **密钥生成格式**：`ol_whs_{32位随机字符串}`

**关键代码位置**：`server/models/WebhookSubscription.ts:113-115`

## 4. 投递失败处理机制

### 4.1 投递记录

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

### 4.2 失败检测

投递失败的情况包括：

1. **HTTP 非 2xx 响应**：`response.ok` 为 false
2. **网络错误**：`FetchError` 等异常
3. **超时**：请求超时时间为 5 秒（固定值，不可配置）

**请求配置**：

```typescript
response = await fetch(subscription.url, {
  method: "POST",
  headers: requestHeaders,
  body: JSON.stringify(requestBody),
  redirect: "error",
  timeout: 5000,  // 固定 5 秒
});
```

**关键代码位置**：`plugins/webhooks/server/tasks/DeliverWebhookTask.ts:758-764`

### 4.3 重要澄清：无自动重试机制

**这是需要特别强调的一点**：

虽然 `BaseTask` 默认配置了 5 次重试（`attempts: 5`），但 **DeliverWebhookTask 实际上不会触发任何自动重试**。

**原因分析**：

```typescript
// plugins/webhooks/server/tasks/DeliverWebhookTask.ts:724-815
private async sendWebhook({...}) {
  // ...
  try {
    // 发送请求...
    response = await fetch(subscription.url, {...});
    status = response.ok ? "success" : "failed";
  } catch (err) {
    // 捕获所有异常！
    Logger.error("Failed to send webhook", err, {...});
    status = "failed";  // 只是设置状态，不抛出异常
  }

  // 更新投递记录
  await delivery.update({ status, ... });

  // 检查是否需要禁用订阅
  if (status === "failed") {
    await this.checkAndDisableSubscription(subscription);
  }
  
  // 方法正常返回，没有抛出异常
}
```

**关键发现**：

1. **异常被内部捕获**：`sendWebhook()` 方法内部使用了完整的 try-catch，所有异常（包括网络错误、超时、HTTP 错误）都被捕获
2. **状态记录为 failed**：异常发生时，只是将投递记录状态设置为 `"failed"`，然后继续执行
3. **任务正常完成**：由于没有抛出异常，Bull 队列认为任务成功执行，**不会触发任何重试**
4. **执行失败监控**：然后调用 `checkAndDisableSubscription()` 检查是否需要禁用订阅

**这意味着**：
- 单次投递失败后，**不会自动重试**
- 失败的投递记录会被保存到数据库
- 系统会监控失败率，当达到阈值时**自动禁用订阅**
- 订阅被禁用后，**不会再发送任何事件**，直到用户手动重新启用

### 4.4 自动禁用策略

当 webhook 持续失败时，系统会自动禁用订阅。这是 Outline 的主要失败保护机制，而不是重试。

#### 触发条件

1. **时间窗口**：默认 24 小时（可通过 `WEBHOOK_FAILURE_TIME_WINDOW` 环境变量调整）
2. **失败率阈值**：默认 80%（可通过 `WEBHOOK_FAILURE_RATE_THRESHOLD` 环境变量调整）
3. **最小数据点**：至少 10 次投递（`MIN_DELIVERIES_FOR_ANALYSIS = 10`）

#### 判定逻辑

```typescript
// plugins/webhooks/server/tasks/DeliverWebhookTask.ts:817-896
private async checkAndDisableSubscription(subscription: WebhookSubscription) {
  // 1. 计算分析时间窗口
  const timeWindowSeconds = env.WEBHOOK_FAILURE_TIME_WINDOW;  // 默认 86400 秒（24小时）
  const failureRateThreshold = env.WEBHOOK_FAILURE_RATE_THRESHOLD;  // 默认 80%
  const timeWindowStart = new Date(Date.now() - timeWindowSeconds * 1000);

  // 2. 获取时间窗口内的所有投递记录
  const deliveriesInWindow = await WebhookDelivery.findAll({
    where: {
      webhookSubscriptionId: subscription.id,
      createdAt: { [Op.gte]: timeWindowStart },
    },
    order: [["createdAt", "DESC"]],
  });

  // 3. 计算失败率
  const failedDeliveries = deliveriesInWindow.filter(
    (delivery) => delivery.status === "failed"
  );
  const failureRate = (failedDeliveries.length / deliveriesInWindow.length) * 100;

  // 4. 检查是否超过阈值
  if (
    failureRate >= failureRateThreshold &&
    deliveriesInWindow.length >= DeliverWebhookTask.MIN_DELIVERIES_FOR_ANALYSIS
  ) {
    // 5. 禁用订阅
    await subscription.disable();

    // 6. 发送邮件通知给订阅创建者
    const [createdBy, team] = await Promise.all([
      User.findOne({ where: { id: subscription.createdById, ... } }),
      subscription.$get("team"),
    ]);

    if (createdBy && team) {
      await new WebhookDisabledEmail({
        to: createdBy.email,
        language: createdBy.language,
        teamUrl: team.url,
        webhookName: subscription.name,
      }).schedule();
    }
  }
}
```

**关键代码位置**：`plugins/webhooks/server/tasks/DeliverWebhookTask.ts:817-896`

#### 禁用流程总结

```
投递失败
    │
    ▼
记录 WebhookDelivery(status="failed")
    │
    ▼
检查时间窗口内的失败率
    │
    ├── 失败率 < 80% 或 投递数 < 10 ──▶ 不做处理，等待下次事件
    │
    └── 失败率 >= 80% 且 投递数 >= 10
                    │
                    ▼
              禁用订阅 (enabled=false)
                    │
                    ▼
              发送邮件通知创建者
                    │
                    ▼
              后续事件不再投递（直到手动启用）
```

### 4.5 环境变量配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `WEBHOOK_FAILURE_TIME_WINDOW` | 86400（24小时） | 失败分析时间窗口（秒） |
| `WEBHOOK_FAILURE_RATE_THRESHOLD` | 80 | 触发禁用的失败率阈值（百分比） |
| `ALLOWED_PRIVATE_IP_ADDRESSES` | 空 | 允许访问的私有 IP 地址列表 |

**关键代码位置**：`server/env.ts:782-810`

### 4.6 投递记录清理

系统会定期清理过期的投递记录：

- **清理周期**：每天执行（`TaskInterval.Day`）
- **保留期限**：7 天
- **任务类型**：`CleanupWebhookDeliveriesTask`

**关键代码位置**：`plugins/webhooks/server/tasks/CleanupWebhookDeliveriesTask.ts`

## 5. 整体架构

### 5.1 完整数据流

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                              事件产生层                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────────────────────────┐   │
│  │ 业务操作    │───▶│ Event.save  │───▶│ @AfterSave → globalEventQueue   │   │
│  │ (如登录)    │    │ (事务中)    │    │ (事务提交后才入队)               │   │
│  └─────────────┘    └─────────────┘    └──────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│                              事件分发层                                          │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ Worker 消费 globalEventQueue                                               │ │
│  │   ├─> 遍历所有 Processor                                                    │ │
│  │   └─> 匹配 applicableEvents，加入 processorEventQueue                      │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                           │
│                                      ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ WebhookProcessor 执行                                                       │ │
│  │   ├─> 查询团队内所有启用的订阅                                               │ │
│  │   ├─> validForEvent 筛选匹配的订阅                                          │ │
│  │   └─> 为每个匹配订阅调度 DeliverWebhookTask → 加入 taskQueue               │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│                              投递执行层                                          │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ Worker 消费 taskQueue，执行 DeliverWebhookTask.perform()                  │ │
│  │   ├─> 检查订阅是否已启用                                                     │ │
│  │   ├─> 根据事件类型调用对应 handler                                          │ │
│  │   └─> 调用 sendWebhook()                                                    │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                           │
│                                      ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ sendWebhook() 执行流程                                                      │ │
│  │   1. 创建 WebhookDelivery(status="pending")                                 │ │
│  │   2. 构建请求体和签名                                                        │ │
│  │   3. 发送 HTTP POST (超时 5 秒)                                             │ │
│  │   4. 捕获所有异常，设置 status="success"/"failed"                            │ │
│  │   5. 更新 WebhookDelivery 状态                                               │ │
│  │   6. 如果 failed → 调用 checkAndDisableSubscription()                        │ │
│  │      (注意：不抛出异常，所以不会触发队列重试)                                 │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│                              失败保护层                                          │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ checkAndDisableSubscription() 执行                                          │ │
│  │   ├─> 统计过去 24 小时内的投递记录                                          │ │
│  │   ├─> 计算失败率                                                            │ │
│  │   ├─> 判断条件：失败率 >= 80% 且 投递数 >= 10                               │ │
│  │   └─> 满足条件：                                                             │ │
│  │       ├─> 禁用订阅 (enabled=false)                                          │ │
│  │       └─> 发送邮件通知创建者                                                 │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 核心组件

| 组件 | 文件位置 | 职责 |
|------|----------|------|
| Event | server/models/Event.ts | 事件模型，@AfterSave 钩子入队 |
| WebhookSubscription | server/models/WebhookSubscription.ts | 订阅模型，事件匹配，签名生成 |
| WebhookDelivery | server/models/WebhookDelivery.ts | 投递记录模型 |
| WebhookProcessor | plugins/webhooks/server/processors/WebhookProcessor.ts | 事件监听与订阅筛选 |
| DeliverWebhookTask | plugins/webhooks/server/tasks/DeliverWebhookTask.ts | 执行投递，失败监控 |
| CleanupWebhookDeliveriesTask | plugins/webhooks/server/tasks/CleanupWebhookDeliveriesTask.ts | 定期清理投递记录 |
| webhookSubscriptions API | plugins/webhooks/server/api/webhookSubscriptions.ts | 订阅管理 API |

### 5.3 支持的事件类型

Webhook 支持的主要事件类型包括：

- **用户事件**：`users.create`, `users.signin`, `users.signout`, `users.update`, `users.suspend`, `users.activate`, `users.delete`, `users.invite`, `users.promote`, `users.demote`, `users.invite_accepted`
- **文档事件**：`documents.create`, `documents.publish`, `documents.unpublish`, `documents.delete`, `documents.permanent_delete`, `documents.archive`, `documents.unarchive`, `documents.restore`, `documents.move`, `documents.update`, `documents.title_change`, `documents.add_user`, `documents.remove_user`, `documents.add_group`, `documents.remove_group`
- **集合事件**：`collections.create`, `collections.update`, `collections.delete`, `collections.move`, `collections.permission_changed`, `collections.archive`, `collections.restore`, `collections.add_user`, `collections.remove_user`, `collections.add_group`, `collections.remove_group`
- **其他事件**：`revisions.create`, `fileOperations.*`, `groups.*`, `integrations.*`, `pins.*`, `stars.*`, `shares.*`, `comments.*`, `attachments.*`, `views.create`, `webhookSubscriptions.*`

**关键代码位置**：`plugins/webhooks/server/tasks/DeliverWebhookTask.ts:108-267`

## 6. 技术特点与限制

### 6.1 优点

1. **灵活的事件订阅**：支持通配符和前缀匹配
2. **安全的签名机制**：HMAC-SHA256 签名，密钥加密存储
3. **自动故障保护**：高失败率时自动禁用订阅并通知
4. **完整的审计追踪**：所有投递记录都被保存
5. **异步处理**：使用四层任务队列，不阻塞主流程
6. **事务一致性**：事件在事务提交后才入队

### 6.2 限制

1. **无自动重试机制**：投递失败后不会自动重试，只会记录失败并监控失败率
2. **固定超时**：请求超时固定为 5 秒，不可配置
3. **响应体限制**：最多只保存 1KB 的响应体
4. **无去重机制**：同一事件可能因队列重试等原因多次发送
5. **服务端不验证时间戳**：时间戳仅参与签名，服务端不做重放验证，需接收方自行实现

### 6.3 可优化建议

1. **实现可选的重试机制**：考虑加入指数退避的重试策略，作为可配置选项
2. **可配置的超时**：允许为每个订阅配置不同的超时时间
3. **去重机制**：为事件添加唯一 ID，防止重复处理
4. **更细粒度的失败监控**：区分网络错误、超时、服务端错误等不同失败类型
5. **服务端可选的时间戳验证**：提供可选的时间戳验证配置
6. **死信队列**：对于持续失败的投递，可以考虑加入死信队列

## 7. 关键澄清总结

### 7.1 关于调度路径

- **四层队列**：`globalEventQueue` → `processorEventQueue` → `taskQueue`
- **事务一致性**：事件在事务提交后才入队
- **订阅筛选**：`WebhookProcessor` 监听所有事件，筛选匹配的订阅后调度投递任务

### 7.2 关于签名时间戳

- **时间戳仅参与签名**：服务端不会对时间戳进行任何验证
- **接收方责任**：如果需要防止重放攻击，接收方必须自己实现时间戳验证逻辑
- **签名格式**：`t={timestamp},s={HMAC-SHA256(secret, `${t}.${payload}`)}`

### 7.3 关于失败重试

- **无自动重试**：虽然 BaseTask 配置了 5 次重试，但由于 `sendWebhook()` 内部捕获了所有异常，实际上不会触发任何重试
- **失败处理方式**：失败后记录 `WebhookDelivery(status="failed")`，然后检查失败率
- **自动禁用**：当 24 小时内失败率 >= 80% 且投递数 >= 10 时，自动禁用订阅并发送邮件通知
- **手动恢复**：订阅被禁用后，需要用户手动重新启用才能恢复投递

## 8. 代码参考

- **事件模型**：`server/models/Event.ts`
- **订阅模型**：`server/models/WebhookSubscription.ts`
- **投递记录模型**：`server/models/WebhookDelivery.ts`
- **队列定义**：`server/queues/index.ts`
- **Worker 服务**：`server/services/worker.ts`
- **事件处理器**：`plugins/webhooks/server/processors/WebhookProcessor.ts`
- **投递任务**：`plugins/webhooks/server/tasks/DeliverWebhookTask.ts`
- **清理任务**：`plugins/webhooks/server/tasks/CleanupWebhookDeliveriesTask.ts`
- **订阅 API**：`plugins/webhooks/server/api/webhookSubscriptions.ts`
- **环境变量**：`server/env.ts`
- **BaseTask**：`server/queues/tasks/base/BaseTask.ts`
