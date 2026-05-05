# Outline 与 Slack、GitHub、Linear 双向集成分析报告

> 分析日期：2026-05-05
> 基于代码版本：Outline Monorepo

---

## 目录

1. [核心架构概览](#1-核心架构概览)
2. [事件触发源与分发机制](#2-事件触发源与分发机制)
3. [Slack 集成实现](#3-slack-集成实现)
4. [GitHub 集成实现](#4-github-集成实现)
5. [Linear 集成实现](#5-linear-集成实现)
6. [Webhook 验签机制](#6-webhook-验签机制)
7. [失败重试与容错策略](#7-失败重试与容错策略)
8. [跨系统数据流转](#8-跨系统数据流转)
9. [幂等性与去重机制](#9-幂等性与去重机制)
10. [状态一致性与边界条件](#10-状态一致性与边界条件)
11. [关键设计模式总结](#11-关键设计模式总结)
12. [附录](#12-附录)

---

## 1. 核心架构概览

### 1.1 数据模型

Outline 使用两个核心模型统一管理所有第三方集成：

#### Integration 模型

**文件位置**: `server/models/integration.ts`

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `IntegrationType` | 集成类型（Post/Command/Embed/Analytics/LinkedAccount/Import） |
| `service` | `IntegrationService` | 集成服务（Slack/GitHub/Linear 等） |
| `settings` | `JSONB` | 集成特定配置 |
| `events` | `string[]` | 订阅的事件列表 |
| `issueSources` | `IssueSource[] \| null` | Issue 追踪系统的源配置 |
| `userId` | `UUID` | 创建用户 ID |
| `teamId` | `UUID` | 团队 ID |
| `collectionId` | `UUID \| null` | 关联集合 ID（可选） |
| `authenticationId` | `UUID` | 关联认证记录 ID |

#### IntegrationAuthentication 模型

**文件位置**: `server/models/IntegrationAuthentication.ts`

| 字段 | 类型 | 加密 | 说明 |
|------|------|------|------|
| `service` | `IntegrationService` | 否 | 服务类型 |
| `token` | `string` | **是** | 访问令牌 |
| `refreshToken` | `string` | **是** | 刷新令牌 |
| `clientId` | `string \| null` | **是** | 客户端 ID（可选） |
| `clientSecret` | `string \| null` | **是** | 客户端密钥（可选） |
| `expiresAt` | `Date \| null` | 否 | 令牌过期时间 |
| `scopes` | `string[]` | 否 | OAuth 权限范围 |

**核心方法**:
- `isExpiringSoon(thresholdMs)`: 检查令牌是否即将过期
- `refreshTokenIfNeeded(refreshCallback, thresholdMs)`: 按需刷新令牌（带行级锁）

### 1.2 集成类型定义

```typescript
enum IntegrationType {
  Post = "post",           // Outline → 外部系统推送
  Command = "command",     // 外部系统 → Outline 命令
  Embed = "embed",         // 嵌入外部系统内容
  Analytics = "analytics", // 分析数据收集
  LinkedAccount = "linkedAccount",  // 用户账户关联
  Import = "import",       // 数据导入
}
```

### 1.3 服务类型定义

```typescript
enum IntegrationService {
  Slack = "slack",
  GitHub = "github",
  GitLab = "gitlab",
  Linear = "linear",
  Figma = "figma",
  Notion = "notion",
  // ... 其他服务
}
```

---

## 2. 事件触发源与分发机制

### 2.1 事件触发源

事件由模型操作自动触发，通过 Sequelize 生命周期钩子实现。

**文件位置**: `server/models/base/Model.ts`

#### 触发钩子

| 钩子 | 事件名格式 | 触发场景 |
|------|-----------|----------|
| `@AfterCreate` | `{namespace}.create` | 模型创建 |
| `@AfterUpdate` | `{namespace}.update` | 模型更新 |
| `@AfterDestroy` | `{namespace}.delete` | 模型删除 |
| `@AfterRestore` | `{namespace}.create` | 模型恢复 |
| `@AfterUpsert` | `{namespace}.create` | 模型创建或更新 |

#### 上下文方法

所有带 `*WithCtx` 后缀的方法会自动触发事件：

```typescript
// 自动触发事件的方法
model.saveWithCtx(ctx, options, eventOpts)
model.updateWithCtx(ctx, keys, eventOpts)
model.destroyWithCtx(ctx, eventOpts)
model.restoreWithCtx(ctx, eventOpts)

// 静态方法
Model.createWithCtx(ctx, values, eventOpts, createOpts)
Model.findOrCreateWithCtx(ctx, options, eventOpts)
```

### 2.2 事件入队机制

**文件位置**: `server/models/Event.ts`

事件保存后通过 `@AfterSave` 钩子入队：

```typescript
@AfterSave
static async enqueue(model: Event, options: SaveOptions) {
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

**关键特性**:
- **事务感知**: 确保事务成功提交后才分发事件
- **延迟执行**: 使用 `afterCommit` 钩子

### 2.3 队列系统

**文件位置**: `server/queues/index.ts`

| 队列 | 用途 | 重试策略 |
|------|------|----------|
| `globalEventQueue` | 全局事件分发 | 5次，指数退避（1s） |
| `processorEventQueue` | 处理器事件 | 5次，指数退避（10s） |
| `websocketQueue` | WebSocket 广播 | 超时 10s，无重试 |
| `taskQueue` | 异步任务 | 5次，指数退避（10s） |

**默认任务选项** (`server/queues/tasks/base/BaseTask.ts`):

```typescript
public get options(): JobOptions {
  return {
    priority: TaskPriority.Normal,
    attempts: 5,                          // 最多 5 次尝试
    backoff: {
      type: "exponential",               // 指数退避
      delay: 60 * 1000,                  // 初始延迟 60 秒
    },
  };
}
```

### 2.4 处理器分发

**文件位置**: `server/queues/processors/index.ts`

处理器通过 `applicableEvents` 声明要处理的事件：

```typescript
abstract class BaseProcessor {
  static applicableEvents: (Event["name"] | "*")[] = [];
  public abstract perform(event: Event): Promise<void>;
}
```

**示例处理器**：

| 处理器 | 监听事件 | 功能 |
|--------|----------|------|
| `SlackProcessor` | `documents.publish`, `revisions.create`, `integrations.create` | Slack 推送通知 |
| `WebhookProcessor` | `*` | 通用 Webhook 订阅 |
| `WebsocketsProcessor` | 大部分事件 | WebSocket 实时广播 |
| `SearchIndexProcessor` | 文档相关事件 | 搜索索引更新 |

---

## 3. Slack 集成实现

### 3.1 集成类型矩阵

| 类型 | 方向 | 功能描述 | 数据流向 |
|------|------|----------|----------|
| **Post** | Outline → Slack | 文档发布/更新时推送通知到 Slack 频道 | 单向推送 |
| **Command** | Slack → Outline | 支持 `/outline` 命令搜索文档 | 双向交互 |
| **LinkedAccount** | 双向 | 关联 Slack 用户与 Outline 用户 | 身份关联 |
| **Unfurl** | Slack → Outline | 链接展开预览 | 单向响应 |

### 3.2 事件同步机制（Outline → Slack）

**文件位置**: `plugins/slack/server/processors/SlackProcessor.ts`

#### 监听事件

```typescript
static applicableEvents: Event["name"][] = [
  "documents.publish",
  "revisions.create",
  "integrations.create",
];
```

#### 约束检查

```typescript
// 约束1: 忽略批量导入的文档
if (event.name === "documents.publish" && event.data?.source === "import") {
  return;
}

// 约束2: 忽略草稿文档
if (!document.publishedAt) {
  return;
}

// 约束3: 发布后1分钟内的更新合并通知
if (
  event.name === "revisions.create" &&
  differenceInMilliseconds(document.updatedAt, document.publishedAt) < Minute.ms
) {
  return;
}
```

### 3.3 Webhook 处理（Slack → Outline）

**文件位置**: `plugins/slack/server/api/hooks.ts`

#### 端点概览

| 端点 | 触发场景 | 功能 |
|------|----------|------|
| `hooks.unfurl` | 用户发送 Outline 链接 | 链接展开预览 |
| `hooks.interactive` | 用户点击消息按钮 | 处理"Post to Channel" |
| `hooks.slack` | `/outline` 命令 | 文档搜索 |

#### 验签处理

```typescript
function verifySlackToken(token: string) {
  if (!env.SLACK_VERIFICATION_TOKEN) {
    throw AuthenticationError(
      "SLACK_VERIFICATION_TOKEN is not present in environment"
    );
  }
  
  if (!safeEqual(env.SLACK_VERIFICATION_TOKEN, token)) {
    throw AuthenticationError("Invalid token");
  }
}
```

### 3.4 OAuth 认证流程

**文件位置**: `plugins/slack/server/auth/slack.ts`

#### 三种集成类型的创建

**1. Post 类型**（推送到特定集合）

```typescript
await Integration.create<Integration<IntegrationType.Post>>({
  service: IntegrationService.Slack,
  type: IntegrationType.Post,
  userId: user.id,
  teamId: user.teamId,
  authenticationId: authentication.id,
  collectionId,  // 目标集合
  events: ["documents.update", "documents.publish"],
  settings: {
    url: data.incoming_webhook.url,
    channel: data.incoming_webhook.channel,
    channelId: data.incoming_webhook.channel_id,
  },
});
```

**2. Command 类型**（命令支持）

```typescript
await Integration.create<Integration<IntegrationType.Command>>({
  service: IntegrationService.Slack,
  type: IntegrationType.Command,
  settings: {
    serviceTeamId: data.team_id,  // Slack 团队 ID
  },
});
```

**3. LinkedAccount 类型**（用户关联）

```typescript
await Integration.create<Integration<IntegrationType.LinkedAccount>>({
  service: IntegrationService.Slack,
  type: IntegrationType.LinkedAccount,
  userId: user.id,
  teamId: user.teamId,
  settings: {
    slack: {
      serviceUserId: data.user_id,  // Slack 用户 ID
      serviceTeamId: data.team_id,  // Slack 团队 ID
    },
  },
});
```

---

## 4. GitHub 集成实现

### 4.1 集成类型

GitHub 集成主要是 **Embed** 类型，支持：

| 功能 | 方向 | 描述 |
|------|------|------|
| 链接展开 | GitHub → Outline | 展开 Issue、PR、Project 链接 |
| Webhook 同步 | GitHub → Outline | 仓库变更事件同步到 Outline |
| Issue 追踪 | 双向 | 集成到 Outline 的 Issue 系统 |

### 4.2 认证机制

**文件位置**: `plugins/github/server/github.ts`

#### 三种认证模式

```typescript
// 1. App 级别认证
private static authenticateAsApp = () => {
  if (!GitHub.appOctokit) {
    GitHub.appOctokit = new CustomOctokit({
      authStrategy: createAppAuth,
      auth: {
        appId: GitHub.appId,
        privateKey: GitHub.appKey,
        clientId: GitHub.clientId,
        clientSecret: GitHub.clientSecret,
      },
    });
  }
  return GitHub.appOctokit;
};

// 2. 用户 OAuth 认证
public static authenticateAsUser = async (code, state) => {
  return GitHub.authenticateAsApp().auth({
    type: "oauth-user",
    code,
    state,
    factory: (options) => new CustomOctokit({
      authStrategy: createOAuthUserAuth,
      auth: options,
    }),
  });
};

// 3. 安装级别认证
public static authenticateAsInstallation = async (installationId) => {
  return GitHub.authenticateAsApp().auth({
    type: "installation",
    installationId,
    factory: (options) => new CustomOctokit({
      authStrategy: createAppAuth,
      auth: options,
    }),
  });
};
```

### 4.3 Webhook 处理（GitHub → Outline）

**文件位置**: `plugins/github/server/api/github.ts`

#### 路由定义

```typescript
router.post(
  "github.webhooks",
  validateWebhook({
    secretKey: env.GITHUB_WEBHOOK_SECRET!,
    getSignatureFromHeader: (ctx) => {
      const signatureHeader = headers["x-hub-signature-256"];
      return signature?.split("=")[1];
    },
  }),
  async (ctx) => {
    await new GitHubWebhookTask().schedule({
      payload: body,
      headers,
    });
    ctx.status = 202;  // 异步处理，立即返回
  }
);
```

---

## 5. Linear 集成实现

### 5.1 集成类型

Linear 集成是 **Embed** 类型，支持：

| 功能 | 方向 | 描述 |
|------|------|------|
| 链接展开 | Linear → Outline | 展开 Issue、Project 链接 |
| OAuth 2.0 | 双向 | 支持短期 token 自动刷新 |

### 5.2 认证机制

**文件位置**: `plugins/linear/server/linear.ts`

#### OAuth 流程

```typescript
// 1. 授权码换取 token
static async oauthAccess(code: string) {
  const body = new URLSearchParams();
  body.set("code", code);
  body.set("client_id", env.LINEAR_CLIENT_ID!);
  body.set("client_secret", env.LINEAR_CLIENT_SECRET!);
  body.set("redirect_uri", LinearUtils.callbackUrl());
  body.set("grant_type", "authorization_code");
  
  const res = await fetch(LinearUtils.tokenUrl, { method: "POST", body });
  return AccessTokenResponseSchema.parse(await res.json());
}

// 2. 刷新 token
static async refreshToken(refreshToken: string) {
  const body = new URLSearchParams();
  body.set("refresh_token", refreshToken);
  body.set("client_id", env.LINEAR_CLIENT_ID!);
  body.set("client_secret", env.LINEAR_CLIENT_SECRET!);
  body.set("grant_type", "refresh_token");
  
  const res = await fetch(LinearUtils.tokenUrl, { method: "POST", body });
  return AccessTokenResponseSchema.parse(await res.json());
}
```

### 5.3 Token 自动刷新机制

**文件位置**: `server/models/IntegrationAuthentication.ts`

#### 核心方法

```typescript
async refreshTokenIfNeeded(
  refreshCallback: TokenRefreshCallback,
  thresholdMs: number = 5 * Minute.ms
): Promise<string> {
  // ========== 快速检查（无锁）==========
  if (!this.isExpiringSoon(thresholdMs) || !this.refreshToken) {
    return this.token;
  }
  
  // ========== 事务 + 行级锁 ==========
  await this.sequelize.transaction(async (transaction) => {
    const lockedAuth = await this.findByPk(this.id, {
      transaction,
      lock: transaction.LOCK.UPDATE,  // 行级锁防止并发
      rejectOnEmpty: true,
    });
    
    // ========== 双重检查 ==========
    if (lockedAuth.isExpiringSoon(thresholdMs) && lockedAuth.refreshToken) {
      const tokenResponse = await refreshCallback(lockedAuth.refreshToken);
      
      await lockedAuth.update(
        {
          token: tokenResponse.access_token,
          refreshToken: tokenResponse.refresh_token || lockedAuth.refreshToken,
          expiresAt: addSeconds(Date.now(), tokenResponse.expires_in),
        },
        { transaction }
      );
    } else {
      refreshedToken = lockedAuth.token;
    }
  });
  
  return refreshedToken;
}
```

---

## 6. Webhook 验签机制

### 6.1 Outline 接收 Webhook 的验签

#### 通用中间件

**文件位置**: `server/middlewares/validateWebhook.ts`

```typescript
export default function validateWebhook({
  secretKey,
  getSignatureFromHeader,
  hmacSign = true,
}: {
  secretKey: string | ((ctx: APIContext) => Promise<string | undefined>);
  getSignatureFromHeader: (ctx: APIContext) => string | undefined;
  hmacSign?: boolean;
}) {
  return async function validateWebhookMiddleware(ctx, next) {
    const { body } = ctx.request;
    const signatureFromHeader = getSignatureFromHeader(ctx);
    
    if (!signatureFromHeader) {
      ctx.status = 401;
      ctx.body = "Missing signature header";
      return;
    }
    
    const key =
      typeof secretKey === "function" ? await secretKey(ctx) : secretKey;
    
    if (!key) {
      ctx.status = 401;
      ctx.body = "Invalid signature";
      return;
    }
    
    const computedSignature = hmacSign
      ? crypto
          .createHmac("sha256", key)
          .update(JSON.stringify(body))
          .digest("hex")
      : key;
    
    if (!safeEqual(computedSignature, signatureFromHeader)) {
      ctx.status = 401;
      ctx.body = "Invalid signature";
      return;
    }
    
    return next();
  };
}
```

#### 各平台验签实现

| 平台 | 签名方式 | 签名头 | 密钥来源 |
|------|----------|--------|----------|
| **GitHub** | HMAC-SHA256 | `x-hub-signature-256` | 环境变量 `GITHUB_WEBHOOK_SECRET` |
| **GitLab** | 简单 Token | `x-gitlab-token` | 环境变量或数据库 |
| **Slack** | 简单 Token | `body.token` | 环境变量 `SLACK_VERIFICATION_TOKEN` |

### 6.2 Outline 发送 Webhook 的签名

**文件位置**: `server/models/WebhookSubscription.ts`

#### 签名生成

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

**签名格式分析**：
- **时间戳**: `t=1714900000000` - 用于防止重放攻击
- **签名**: `s=abcdef123456...` - HMAC-SHA256 签名

**接收方验证步骤**：
1. 解析 `t` 和 `s` 值
2. 验证时间戳是否在可接受时间窗口内（防止重放）
3. 使用相同密钥和算法计算签名
4. 使用 `safeEqual` 比较签名

---

## 7. 失败重试与容错策略

### 7.1 任务重试机制

#### 默认重试策略

**文件位置**: `server/queues/tasks/base/BaseTask.ts`

```typescript
public get options(): JobOptions {
  return {
    priority: TaskPriority.Normal,
    attempts: 5,                          // 最多 5 次尝试
    backoff: {
      type: "exponential",               // 指数退避
      delay: 60 * 1000,                  // 初始延迟 60 秒
    },
  };
}
```

#### 各队列重试配置

| 队列 | attempts | backoff.type | backoff.delay |
|------|----------|--------------|---------------|
| `globalEventQueue` | 5 | exponential | 1 秒 |
| `processorEventQueue` | 5 | exponential | 10 秒 |
| `taskQueue` | 5 | exponential | 10 秒 |
| `websocketQueue` | - | - | timeout: 10 秒 |

#### 指数退避计算

```
第 1 次失败: 等待 60 秒
第 2 次失败: 等待 120 秒 (2^1 * 60)
第 3 次失败: 等待 240 秒 (2^2 * 60)
第 4 次失败: 等待 480 秒 (2^3 * 60)
第 5 次失败: 任务失败
```

### 7.2 Webhook 投递失败处理

**文件位置**: `plugins/webhooks/server/tasks/DeliverWebhookTask.ts`

#### 高失败率自动禁用

```typescript
private async checkAndDisableSubscription(subscription: WebhookSubscription) {
  const timeWindowSeconds = env.WEBHOOK_FAILURE_TIME_WINDOW;
  const failureRateThreshold = env.WEBHOOK_FAILURE_RATE_THRESHOLD;
  const timeWindowStart = new Date(Date.now() - timeWindowSeconds * 1000);
  
  const deliveriesInWindow = await WebhookDelivery.findAll({
    where: {
      webhookSubscriptionId: subscription.id,
      createdAt: { [Op.gte]: timeWindowStart },
    },
  });
  
  const failedDeliveries = deliveriesInWindow.filter(
    (delivery) => delivery.status === "failed"
  );
  const failureRate =
    (failedDeliveries.length / deliveriesInWindow.length) * 100;
  
  if (
    failureRate >= failureRateThreshold &&
    deliveriesInWindow.length >= DeliverWebhookTask.MIN_DELIVERIES_FOR_ANALYSIS
  ) {
    await subscription.disable();
    
    const [createdBy, team] = await Promise.all([
      User.findOne({ where: { id: subscription.createdById } }),
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

#### 请求约束

```typescript
response = await fetch(subscription.url, {
  method: "POST",
  headers: requestHeaders,
  body: JSON.stringify(requestBody),
  redirect: "error",   // 禁止重定向
  timeout: 5000,       // 5 秒超时
});

// 响应体大小限制
private static readonly MAX_RESPONSE_BODY_SIZE = 1024;
responseBody = text.slice(0, DeliverWebhookTask.MAX_RESPONSE_BODY_SIZE);
```

### 7.3 订阅数量限制

**文件位置**: `server/models/WebhookSubscription.ts`

```typescript
@BeforeCreate
static async checkLimit(model: WebhookSubscription) {
  const count = await this.count({
    where: { teamId: model.teamId },
  });
  
  if (count >= WebhookSubscriptionValidation.maxSubscriptions) {
    throw ValidationError(
      `You have reached the limit of ${WebhookSubscriptionValidation.maxSubscriptions} webhooks`
    );
  }
}
```

---

## 8. 跨系统数据流转

### 8.1 数据流架构总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Outline 内部事件系统                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────────────────┐  │
│  │ 模型操作      │─────▶│ 事件触发      │─────▶│  globalEventQueue        │  │
│  │ (CRUD)       │      │ (Sequelize   │      │ (Bull + Redis)           │  │
│  │              │      │  Hooks)      │      │                          │  │
│  └──────────────┘      └──────────────┘      └──────────────────────────┘  │
│                                                    │                         │
│                                                    ▼                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         处理器分发                                        │ │
│  ├────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                        │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐       │ │
│  │  │ SlackProcessor  │  │ WebhookProcessor│  │ Websockets      │       │ │
│  │  │ (Slack 推送)    │  │ (通用 Webhook)  │  │ (实时广播)      │       │ │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘       │ │
│  │                                                                        │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐       │ │
│  │  │ SearchIndex     │  │ Notifications   │  │ ... 其他         │       │ │
│  │  │ (索引更新)      │  │ (通知处理)      │  │                 │       │ │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 入站与出站链路详解

#### 8.2.1 入站链路（外部系统 → Outline）

入站链路处理外部系统主动发送的 Webhook 事件和 API 回调。

| 维度 | Slack | GitHub | Linear |
|------|-------|--------|--------|
| **触发方式** | Events API 推送 | Webhook 推送 | 无（仅 OAuth 回调） |
| **协议** | HTTPS POST | HTTPS POST | HTTPS（OAuth 回调） |
| **认证方式** | Verification Token | HMAC-SHA256 签名 | OAuth 2.0 Code Flow |
| **响应状态** | 200 OK | 202 Accepted | 302 Redirect |
| **处理模式** | 同步处理 | 异步任务 | 同步处理 |

##### Slack 入站链路

**文件位置**: `plugins/slack/server/api/hooks.ts`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Slack → Outline 入站链路                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Slack 发送事件                                                           │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  POST https://outline.example.com/api/hooks.{endpoint}                  │ │
│  │  Headers:                                                                │ │
│  │    Content-Type: application/json                                        │ │
│  │    X-Slack-Signature: v0=xxxxxxxx (如果启用签名验证)                   │ │
│  │  Body:                                                                   │ │
│  │    {                                                                     │ │
│  │      "token": "SLACK_VERIFICATION_TOKEN",  // 验证用                    │ │
│  │      "team_id": "T01234567",                                            │ │
│  │      "event": {                                                          │ │
│  │        "type": "link_shared" | "message" | "app_mention",              │ │
│  │        "user": "U01234567",                                             │ │
│  │        "channel": "C01234567",                                          │ │
│  │        ...                                                               │ │
│  │      }                                                                   │ │
│  │    }                                                                     │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  2. 验签处理                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  verifySlackToken(body.token)                                            │ │
│  │                                                                          │ │
│  │  验证步骤:                                                               │ │
│  │  1. 检查 SLACK_VERIFICATION_TOKEN 环境变量是否存在                       │ │
│  │  2. 使用 safeEqual() 比较 body.token 和环境变量                         │ │
│  │  3. 如果不匹配，抛出 AuthenticationError                                 │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  3. 用户查找                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  查找优先级:                                                             │ │
│  │  ┌────────────────────────────────────────────────────────────────────┐ │ │
│  │  │ 1. LinkedAccount 集成                                               │ │ │
│  │  │    Integration.findOne({                                           │ │ │
│  │  │      where: {                                                      │ │ │
│  │  │        service: IntegrationService.Slack,                          │ │ │
│  │  │        type: IntegrationType.LinkedAccount,                        │ │ │
│  │  │        settings: {                                                 │ │ │
│  │  │          slack: { serviceUserId: user_id, serviceTeamId: team_id } │ │ │
│  │  │        }                                                           │ │ │
│  │  │      }                                                             │ │ │
│  │  │    })                                                              │ │ │
│  │  └────────────────────────────────────────────────────────────────────┘ │ │
│  │  ┌────────────────────────────────────────────────────────────────────┐ │ │
│  │  │ 2. OAuth 认证提供者                                                 │ │ │
│  │  │    UserAuthentication.findOne({                                    │ │ │
│  │  │      where: { providerId: user_id, provider: "slack" }            │ │ │
│  │  │    })                                                               │ │ │
│  │  └────────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  4. 业务处理                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  根据事件类型路由:                                                       │ │
│  │                                                                          │ │
│  │  a) hooks.unfurl (link_shared 事件)                                    │ │
│  │     - 解析文档 URL → 提取 slug                                          │ │
│  │     - 检查阅读权限 can(user, "read", document)                          │ │
│  │     - 调用 Slack API: POST chat.unfurl                                  │ │
│  │       { unfurls: { url: { title, text, color } } }                      │ │
│  │                                                                          │ │
│  │  b) hooks.slack (slash 命令)                                            │ │
│  │     - 解析搜索关键词: body.text                                          │ │
│  │     - 执行搜索: SearchProviderManager.searchForUser()                   │ │
│  │     - 构建 ephemeral 消息:                                              │ │
│  │       { response_type: "ephemeral", attachments: [...] }                │ │
│  │     - 仅发送者可见，不公开到频道                                        │ │
│  │                                                                          │ │
│  │  c) hooks.interactive (按钮点击)                                        │ │
│  │     - 解析 callback_id → 提取 documentId                                │ │
│  │     - 查找文档: Document.findByPk()                                     │ │
│  │     - 构建 in_channel 消息:                                             │ │
│  │       { response_type: "in_channel", text, attachments }                │ │
│  │     - 公开到原频道，所有人可见                                           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

##### GitHub 入站链路

**文件位置**: `plugins/github/server/api/github.ts` + `plugins/github/server/tasks/GitHubWebhookTask.ts`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      GitHub → Outline 入站链路                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. GitHub App 发送 Webhook                                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  POST https://outline.example.com/api/github.webhooks                    │ │
│  │  Headers:                                                                │ │
│  │    X-GitHub-Event: installation_repositories                            │ │
│  │    X-GitHub-Delivery: a1b2c3d4-e5f6-7890-abcd-ef1234567890          │ │
│  │    X-Hub-Signature-256: sha256=abcdef1234567890abcdef1234567890      │ │
│  │  Body:                                                                   │ │
│  │    {                                                                     │ │
│  │      "action": "added" | "removed" | "new_permissions_accepted",       │ │
│  │      "installation": {                                                  │ │
│  │        "id": 123456,                                                    │ │
│  │        "account": { "login": "org-name", "id": 789012 },              │ │
│  │        "permissions": { "issues": "read", "metadata": "read" },       │ │
│  │      },                                                                   │ │
│  │      "repositories_added": [ ... ],                                     │ │
│  │      "repositories_removed": [ ... ],                                   │ │
│  │    }                                                                     │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  2. HMAC-SHA256 验签                                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  validateWebhook({                                                       │ │
│  │    secretKey: env.GITHUB_WEBHOOK_SECRET!,                               │ │
│  │    getSignatureFromHeader: (ctx) => {                                   │ │
│  │      const signatureHeader = headers["x-hub-signature-256"];          │ │
│  │      return signature?.split("=")[1];  // 提取 sha256= 后的部分       │ │
│  │    },                                                                    │ │
│  │  })                                                                      │ │
│  │                                                                          │ │
│  │  验证步骤:                                                               │ │
│  │  1. 提取签名头: signature = "abcdef123456..."                          │ │
│  │  2. 计算 HMAC-SHA256:                                                   │ │
│  │     computedSignature = crypto                                          │ │
│  │       .createHmac("sha256", GITHUB_WEBHOOK_SECRET)                    │ │
│  │       .update(JSON.stringify(body))                                     │ │
│  │       .digest("hex")                                                    │ │
│  │  3. safeEqual(computedSignature, signature)                             │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  3. 异步任务调度（非阻塞）                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  await new GitHubWebhookTask().schedule({                               │ │
│  │    payload: body,                                                        │ │
│  │    headers,                                                              │ │
│  │  });                                                                      │ │
│  │                                                                          │ │
│  │  ctx.status = 202;  // 立即返回 Accepted                                │ │
│  │                                                                          │ │
│  │  为什么异步处理:                                                         │ │
│  │  - GitHub Webhook 超时限制短（约 10 秒）                                │ │
│  │  - 可能需要数据库操作和外部 API 调用                                     │ │
│  │  - 防止重复发送（GitHub 会重试超时的请求）                               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  4. 任务处理                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  GitHubWebhookTask.perform({ headers, payload })                        │ │
│  │                                                                          │ │
│  │  // 通过 PluginManager 委托给 IssueProvider                              │ │
│  │  const plugins = PluginManager.getHooks(Hook.IssueProvider);           │ │
│  │  const plugin = plugins.find(                                           │ │
│  │    (p) => p.value.service === IntegrationService.GitHub                  │ │
│  │  );                                                                      │ │
│  │                                                                          │ │
│  │  await plugin.value.handleWebhook({ headers, payload });                │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  5. 事件处理（GitHubIssueProvider）                                          │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  handleWebhook({ headers, payload })                                     │ │
│  │                                                                          │ │
│  │  const event = headers["x-github-event"];                               │ │
│  │  const action = payload.action;                                          │ │
│  │                                                                          │ │
│  │  switch (event) {                                                        │ │
│  │    case "installation":                                                  │ │
│  │      if (action === "new_permissions_accepted") {                       │ │
│  │        // 权限更新: 重新获取仓库列表，更新 scopes                        │ │
│  │        this.handleInstallationEvent(payload);                            │ │
│  │      }                                                                    │ │
│  │      break;                                                               │ │
│  │    case "installation_repositories":                                     │ │
│  │      // 仓库变更: 添加/移除 issueSources                                 │ │
│  │      this.handleInstallationRepositoriesEvent(payload);                  │ │
│  │      break;                                                               │ │
│  │    case "repository":                                                    │ │
│  │      if (action === "renamed") {                                         │ │
│  │        // 仓库重命名: 更新 source.name                                   │ │
│  │        this.handleRepositoryEvent(payload);                              │ │
│  │      }                                                                    │ │
│  │      break;                                                               │ │
│  │  }                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

##### Linear 入站链路

**文件位置**: `plugins/linear/server/linear.ts`

Linear 没有 Webhook 推送，只有 OAuth 2.0 授权回调：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Linear → Outline 入站链路                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 用户授权后 Linear 回调                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  GET https://outline.example.com/auth/linear.callback                   │ │
│  │  Query:                                                                  │ │
│  │    code=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx                                 │ │
│  │    state=teamId%3Dxxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx               │ │
│  │                                                                          │ │
│  │  state 参数格式: URL 编码的 JSON:                                        │ │
│  │  { "teamId": "uuid", "nonce": "random-string" }                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  2. OAuth 回调处理                                                           │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  router.get("linear.callback", ..., async (ctx) => {                   │ │
│  │    const { code, state } = ctx.input.query;                            │ │
│  │                                                                          │ │
│  │    // 1. 解析并验证 state                                                │ │
│  │    const parsedState = LinearUtils.parseState(state);                  │ │
│  │    verifyOAuthStateNonce(ctx, LinearOAuthNonceCookie, parsedState.nonce); │ │
│  │                                                                          │ │
│  │    // 2. 用 code 换取 access_token                                      │ │
│  │    const tokenResponse = await Linear.oauthAccess(code);                │ │
│  │    // 包含: access_token, refresh_token, expires_in, scope              │ │
│  │                                                                          │ │
│  │    // 3. 获取工作区信息                                                  │ │
│  │    const workspace = await Linear.getInstalledWorkspace(                │ │
│  │      tokenResponse.access_token                                          │ │
│  │    );                                                                    │ │
│  │                                                                          │ │
│  │    // 4. 创建认证记录                                                    │ │
│  │    await IntegrationAuthentication.create({                              │ │
│  │      service: IntegrationService.Linear,                                │ │
│  │      userId: user.id,                                                    │ │
│  │      teamId: user.teamId,                                                │ │
│  │      token: tokenResponse.access_token,                                  │ │
│  │      refreshToken: tokenResponse.refresh_token,                          │ │
│  │      expiresAt: addSeconds(Date.now(), tokenResponse.expires_in),      │ │
│  │      scopes: tokenResponse.scope.split(" "),                            │ │
│  │    });                                                                   │ │
│  │                                                                          │ │
│  │    // 5. 创建集成记录                                                    │ │
│  │    await Integration.createWithCtx(ctx, {                               │ │
│  │      service: IntegrationService.Linear,                                │ │
│  │      type: IntegrationType.Embed,                                       │ │
│  │      settings: {                                                         │ │
│  │        linear: { workspace: { key: workspace.key, name: workspace.name } } │ │
│  │      },                                                                  │ │
│  │    });                                                                   │ │
│  │  })                                                                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 8.2.2 出站链路（Outline → 外部系统）

出站链路处理 Outline 主动向外部系统发送的通知和请求。

| 维度 | Slack | GitHub | Linear |
|------|-------|--------|--------|
| **触发方式** | 内部事件驱动 | 链接展开时调用 | 链接展开时调用 |
| **协议** | HTTPS POST (Incoming Webhook) | REST API | GraphQL API |
| **认证方式** | Webhook URL 包含 Token | Installation Token | OAuth Access Token |
| **重试机制** | 无（处理器失败则丢失） | 无（单此调用） | 无（单此调用） |
| **幂等性** | 依赖时间约束 | 依赖请求唯一性 | 依赖请求唯一性 |

##### Slack 出站链路

**文件位置**: `plugins/slack/server/processors/SlackProcessor.ts`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Outline → Slack 出站链路                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 事件触发                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  触发事件列表:                                                           │ │
│  │  - documents.publish   ──▶ 文档首次发布                                 │ │
│  │  - revisions.create    ──▶ 文档更新（保存新版本）                        │ │
│  │  - integrations.create ──▶ 集成创建（欢迎消息）                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  2. 约束检查（去重与过滤）                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  // 约束1: 忽略批量导入的文档                                           │ │
│  │  if (event.name === "documents.publish" && event.data?.source === "import") { │ │
│  │    return;                                                               │ │
│  │  }                                                                       │ │
│  │                                                                          │ │
│  │  // 约束2: 忽略草稿文档                                                 │ │
│  │  if (!document.publishedAt) {                                           │ │
│  │    return;                                                               │ │
│  │  }                                                                       │ │
│  │                                                                          │ │
│  │  // 约束3: 发布后1分钟内的更新合并（去重）                               │ │
│  │  if (                                                                     │ │
│  │    event.name === "revisions.create" &&                                  │ │
│  │    differenceInMilliseconds(document.updatedAt, document.publishedAt) < Minute.ms │ │
│  │  ) {                                                                     │ │
│  │    return;                                                               │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  3. 智能延迟                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  // 等待 5 秒，让文档摘要有时间生成                                      │ │
│  │  await sleep(5000);                                                     │ │
│  │                                                                          │ │
│  │  为什么需要延迟:                                                         │ │
│  │  - 文档摘要通过异步任务生成（RollupDocumentInsightsTask）               │ │
│  │  - 消息附件需要摘要内容                                                 │ │
│  │  - 避免推送消息缺少摘要信息                                             │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  4. 集成匹配                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  const integration = await Integration.findOne({                        │ │
│  │    where: {                                                             │ │
│  │      teamId: document.teamId,           // 同一团队                      │ │
│  │      collectionId: document.collectionId, // 同一集合                    │ │
│  │      service: IntegrationService.Slack,  // Slack 服务                  │ │
│  │      type: IntegrationType.Post,          // Post 类型                  │ │
│  │      events: {                                                           │ │
│  │        [Op.contains]: [                                                 │ │
│  │          event.name === "revisions.create"                              │ │
│  │            ? "documents.update"                                          │ │
│  │            : event.name,                                                 │ │
│  │        ],                                                                │ │
│  │      },                                                                  │ │
│  │    },                                                                    │ │
│  │  });                                                                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  5. 推送消息                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  await fetch(integration.settings.url, {                                │ │
│  │    method: "POST",                                                       │ │
│  │    headers: { "Content-Type": "application/json" },                     │ │
│  │    body: JSON.stringify({                                                │ │
│  │      text: `${document.updatedBy.name} updated "${document.title}"`,    │ │
│  │      attachments: [presentMessageAttachment(document, team, collection)], │ │
│  │    }),                                                                   │ │
│  │  });                                                                      │ │
│  │                                                                          │ │
│  │  integration.settings.url 是 Slack Incoming Webhook URL:                │ │
│  │  https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXX │ │
│  │  其中最后的路径段就是认证 Token，无需额外 Header                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

##### 通用 Webhook 出站链路

**文件位置**: `plugins/webhooks/server/tasks/DeliverWebhookTask.ts`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              Outline → 外部系统 通用 Webhook 出站链路                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. WebhookProcessor 分发                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  async perform(event: Event) {                                          │ │
│  │    const webhookSubscriptions = await WebhookSubscription.findAll({     │ │
│  │      where: { enabled: true, teamId: event.teamId },                   │ │
│  │    });                                                                   │ │
│  │                                                                          │ │
│  │    const applicableSubscriptions = webhookSubscriptions.filter((webhook) => │ │
│  │      webhook.validForEvent(event)                                       │ │
│  │    );                                                                    │ │
│  │                                                                          │ │
│  │    await Promise.all(                                                    │ │
│  │      applicableSubscriptions.map((subscription) =>                      │ │
│  │        new DeliverWebhookTask().schedule({                              │ │
│  │          event,                                                          │ │
│  │          subscriptionId: subscription.id,                                │ │
│  │        })                                                                 │ │
│  │      )                                                                   │ │
│  │    );                                                                    │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  2. 投递任务执行                                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  public async perform({ subscriptionId, event }: Props) {               │ │
│  │    const subscription = await WebhookSubscription.findByPk(subscriptionId, { │ │
│  │      rejectOnEmpty: true,                                                │ │
│  │    });                                                                    │ │
│  │                                                                          │ │
│  │    // 检查订阅是否已被禁用                                               │ │
│  │    if (!subscription.enabled) {                                          │ │
│  │      Logger.info("task", `WebhookSubscription was disabled before delivery`); │ │
│  │      return;                                                              │ │
│  │    }                                                                      │ │
│  │                                                                          │ │
│  │    // 根据事件类型路由处理                                               │ │
│  │    switch (event.name) {                                                 │ │
│  │      case "documents.create":                                            │ │
│  │      case "documents.publish":                                           │ │
│  │      case "documents.update":                                            │ │
│  │      case "documents.delete":                                            │ │
│  │        await this.handleDocumentEvent(subscription, event);              │ │
│  │        return;                                                            │ │
│  │      // ... 其他事件类型                                                 │ │
│  │    }                                                                      │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  3. 发送 Webhook（核心方法）                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  private async sendWebhook({ event, subscription, payload }) {           │ │
│  │    // ========== 步骤1: 创建投递记录 ==========                          │ │
│  │    const delivery = await WebhookDelivery.create({                       │ │
│  │      webhookSubscriptionId: subscription.id,                             │ │
│  │      status: "pending",                                                   │ │
│  │    });                                                                     │ │
│  │                                                                          │ │
│  │    // ========== 步骤2: 构建请求 ==========                              │ │
│  │    requestBody = presentWebhook({                                        │ │
│  │      event,                                                                │ │
│  │      delivery,                                                             │ │
│  │      payload,                                                              │ │
│  │    });                                                                     │ │
│  │                                                                          │ │
│  │    // 签名（如果配置了 secret）                                           │ │
│  │    requestHeaders = { "Content-Type": "application/json" };              │ │
│  │    const signature = subscription.signature(JSON.stringify(requestBody)); │ │
│  │    if (signature) {                                                       │ │
│  │      requestHeaders["Outline-Signature"] = signature;                   │ │
│  │      // 格式: t=1714900000000,s=abcdef123456...                        │ │
│  │    }                                                                      │ │
│  │                                                                          │ │
│  │    // ========== 步骤3: 发送请求 ==========                              │ │
│  │    let status: WebhookDeliveryStatus;                                    │ │
│  │    try {                                                                   │ │
│  │      response = await fetch(subscription.url, {                           │ │
│  │        method: "POST",                                                    │ │
│  │        headers: requestHeaders,                                           │ │
│  │        body: JSON.stringify(requestBody),                                 │ │
│  │        redirect: "error",   // 禁止重定向                               │ │
│  │        timeout: 5000,        // 5 秒超时                                │ │
│  │      });                                                                   │ │
│  │      status = response.ok ? "success" : "failed";                        │ │
│  │    } catch (err) {                                                         │ │
│  │      status = "failed";                                                   │ │
│  │    }                                                                      │ │
│  │                                                                          │ │
│  │    // ========== 步骤4: 更新投递记录 ==========                          │ │
│  │    await delivery.update({                                                │ │
│  │      status,                                                               │ │
│  │      statusCode: response ? response.status : null,                       │ │
│  │      requestBody,                                                          │ │
│  │      requestHeaders,                                                       │ │
│  │      responseBody: text.slice(0, MAX_RESPONSE_BODY_SIZE),  // 最多 1KB │ │
│  │      responseHeaders: response ? Object.fromEntries(response.headers.entries()) : {}, │ │
│  │    });                                                                     │ │
│  │                                                                          │ │
│  │    // ========== 步骤5: 检查失败率 ==========                            │ │
│  │    if (status === "failed") {                                             │ │
│  │      await this.checkAndDisableSubscription(subscription);                │ │
│  │    }                                                                      │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. 幂等性与去重机制

### 9.1 幂等性设计概述

幂等性是指同一操作执行多次与执行一次产生的效果相同。Outline 的集成系统通过以下方式实现幂等性：

| 层级 | 机制 | 应用场景 | 实现位置 |
|------|------|----------|----------|
| **请求层** | Delivery ID + 时间戳签名 | Webhook 接收方去重 | `WebhookSubscription.signature()` |
| **数据层** | 数据库唯一约束 | 防止重复创建 | 模型定义 |
| **业务层** | 时间窗口约束 | 合并重复事件 | `SlackProcessor.ts` |
| **状态层** | 事务 + 行级锁 | 防止并发更新 | `IntegrationAuthentication.ts` |

### 9.2 通用 Webhook 幂等键

**文件位置**: `plugins/webhooks/server/presenters/webhook.ts` + `server/models/WebhookSubscription.ts`

#### Webhook 请求结构

```typescript
// Webhook 载荷格式
interface WebhookPresentation {
  id: string;                    // 幂等键: Delivery UUID
  actorId: string;               // 操作者 ID
  webhookSubscriptionId: string; // 订阅 ID
  event: string;                 // 事件名: documents.create
  payload: WebhookPayload;       // 业务数据
  createdAt: Date;               // 创建时间
}

// 签名格式
public signature = (payload: string) => {
  if (isNil(this.secret)) {
    return;
  }
  
  const signTimestamp = Date.now();  // 时间戳: 用于重放攻击防护
  
  const signature = crypto
    .createHmac("sha256", this.secret)
    .update(`${signTimestamp}.${payload}`)  // 格式: t.payload
    .digest("hex");
  
  return `t=${signTimestamp},s=${signature}`;  // 完整格式
};
```

#### 幂等键分析

| 字段 | 类型 | 用途 | 唯一性 |
|------|------|------|--------|
| `id` | UUID | **核心幂等键** | 全局唯一，每次投递不同 |
| `payload.id` | string | 业务资源 ID | 同一资源相同 |
| `event` | string | 事件类型 | 标识操作类型 |
| `t=` (签名中) | timestamp | 重放攻击防护 | 每次投递不同 |

#### 接收方如何实现幂等

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    外部系统接收 Webhook 时的幂等处理                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  推荐实现步骤:                                                                │
│                                                                              │
│  1. 验证签名（可选但推荐）                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  const signatureHeader = req.headers["outline-signature"];              │ │
│  │  // 格式: t=1714900000000,s=abcdef123456...                           │ │
│  │                                                                          │ │
│  │  // 解析时间戳和签名                                                    │ │
│  │  const [t, s] = signatureHeader.split(",").map(part => part.split("=")[1]); │ │
│  │  const timestamp = parseInt(t);                                          │ │
│  │  const signature = s;                                                    │ │
│  │                                                                          │ │
│  │  // 防重放: 检查时间戳是否在可接受窗口内（例如 5 分钟）                  │ │
│  │  if (Date.now() - timestamp > 5 * 60 * 1000) {                          │ │
│  │    return res.status(403).send("Request expired");                      │ │
│  │  }                                                                       │ │
│  │                                                                          │ │
│  │  // 验证签名                                                             │ │
│  │  const expectedSignature = crypto                                        │ │
│  │    .createHmac("sha256", WEBHOOK_SECRET)                                │ │
│  │    .update(`${timestamp}.${JSON.stringify(req.body)}`)                  │ │
│  │    .digest("hex");                                                       │ │
│  │                                                                          │ │
│  │  if (!safeEqual(signature, expectedSignature)) {                         │ │
│  │    return res.status(403).send("Invalid signature");                    │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  2. 使用 Delivery ID 去重                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  const deliveryId = req.body.id;  // WebhookPresentation.id             │ │
│  │                                                                          │ │
│  │  // 方案A: 数据库唯一约束（推荐）                                         │ │
│  │  try {                                                                     │ │
│  │    await WebhookDeliveryLog.create({                                     │ │
│  │      deliveryId,                                                          │ │
│  │      event: req.body.event,                                               │ │
│  │      resourceId: req.body.payload.id,                                     │ │
│  │      processedAt: new Date(),                                             │ │
│  │    });                                                                     │ │
│  │  } catch (err) {                                                           │ │
│  │    if (err.code === "23505") {  // PostgreSQL unique violation           │ │
│  │      // 重复请求，直接返回成功                                            │ │
│  │      return res.status(200).send("Already processed");                   │ │
│  │    }                                                                       │ │
│  │    throw err;                                                              │ │
│  │  }                                                                       │ │
│  │                                                                          │ │
│  │  // 方案B: 分布式缓存（Redis）                                            │ │
│  │  const cacheKey = `webhook:processed:${deliveryId}`;                     │ │
│  │  const exists = await redis.set(cacheKey, "1", "EX", 3600, "NX");      │ │
│  │  if (!exists) {                                                           │ │
│  │    return res.status(200).send("Already processed");                     │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  3. 执行业务逻辑（确保幂等）                                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  // 使用 UPSERT 而不是 INSERT                                            │ │
│  │  await Resource.upsert({                                                 │ │
│  │    id: req.body.payload.id,                                              │ │
│  │    title: req.body.payload.model.title,                                  │ │
│  │    updatedAt: req.body.createdAt,                                        │ │
│  │  });                                                                      │ │
│  │                                                                          │ │
│  │  // 或者使用乐观锁                                                       │ │
│  │  const resource = await Resource.findByPk(req.body.payload.id);          │ │
│  │  if (resource && resource.updatedAt >= new Date(req.body.createdAt)) {  │ │
│  │    // 已有更新版本，跳过                                                 │ │
│  │    return res.status(200).send("No update needed");                     │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Slack 推送去重机制

**文件位置**: `plugins/slack/server/processors/SlackProcessor.ts`

Slack 推送通过**业务规则**实现去重，而非技术层面的幂等键：

#### 去重规则分析

| 规则 | 条件 | 效果 | 代码位置 |
|------|------|------|----------|
| **导入忽略** | `event.data?.source === "import"` | 批量导入文档不推送 | 第 85-87 行 |
| **草稿忽略** | `!document.publishedAt` | 草稿文档不推送 | 第 97-99 行 |
| **更新合并** | 发布后 1 分钟内的 `revisions.create` | 与发布事件合并，不单独推送 | 第 101-109 行 |

#### 时间窗口去重原理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Slack 推送时间窗口去重原理                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  场景: 用户发布文档后，立即进行多次编辑保存                                   │
│                                                                              │
│  时间线:                                                                      │
│                                                                              │
│  T=0s:  documents.publish 事件触发                                          │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  SlackProcessor.perform()                                               │ │
│  │                                                                          │ │
│  │  1. await sleep(5000)  // 等待 5 秒                                    │ │
│  │  2. 检查约束: document.publishedAt 存在 ✅                              │ │
│  │  3. 查找集成: collection 匹配 ✅                                        │ │
│  │  4. 推送消息: "User published 'Document Title'"                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  T=10s: revisions.create 事件触发（第一次编辑保存）                          │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  SlackProcessor.perform()                                               │ │
│  │                                                                          │ │
│  │  检查约束:                                                               │ │
│  │  differenceInMilliseconds(                                               │ │
│  │    document.updatedAt (T=10s),                                          │ │
│  │    document.publishedAt (T=0s)                                          │ │
│  │  ) = 10000ms < 60000ms (Minute.ms)                                       │ │
│  │                                                                          │ │
│  │  结果: return;  // 跳过，不推送                                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  T=70s: revisions.create 事件触发（1分10秒后再次编辑）                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  SlackProcessor.perform()                                               │ │
│  │                                                                          │ │
│  │  检查约束:                                                               │ │
│  │  differenceInMilliseconds(                                               │ │
│  │    document.updatedAt (T=70s),                                          │ │
│  │    document.publishedAt (T=0s)                                          │ │
│  │  ) = 70000ms > 60000ms (Minute.ms)                                       │ │
│  │                                                                          │ │
│  │  结果: 推送消息: "User updated 'Document Title'"                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  最终效果:                                                                    │
│  - T=0s:  推送 1 条 "published" 消息                                         │
│  - T=10s: 跳过（1分钟窗口内）                                                 │
│  - T=70s: 推送 1 条 "updated" 消息                                           │
│  - 总共: 2 条消息，而非 3 条                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 限制与注意事项

```typescript
// 当前实现的局限性:
// 1. 没有技术层面的幂等键
// 2. 推送失败不会重试（SlackProcessor 没有重试机制）
// 3. 可能存在以下问题:

// 问题A: 推送失败后不会重试
// 如果 Slack Incoming Webhook 临时不可用，消息将丢失
// 因为 SlackProcessor 是同步执行的，没有加入任务队列

// 问题B: 时间窗口内的更新完全丢失
// 如果用户在发布后 1 分钟内进行了重要更新（如修正错误）
// 该更新通知将不会发送到 Slack

// 问题C: 没有 Delivery ID
// 接收方无法基于 ID 去重，只能基于内容或时间判断
```

### 9.4 GitHub Webhook 幂等性

**文件位置**: `plugins/github/server/api/github.ts`

GitHub Webhook 通过以下机制实现幂等：

#### X-GitHub-Delivery 头

```typescript
// GitHub 发送的请求头包含唯一 Delivery ID
// 参考: https://docs.github.com/en/webhooks-and-events/webhooks/webhook-events-and-payloads

// 示例请求头:
// X-GitHub-Event: installation_repositories
// X-GitHub-Delivery: a1b2c3d4-e5f6-7890-abcd-ef1234567890  // 全局唯一 UUID
// X-Hub-Signature-256: sha256=abcdef1234567890...

// Outline 当前实现: 没有使用 X-GitHub-Delivery 去重
// 但使用了异步任务 + 202 Accepted 来避免重复处理

// 路由实现:
router.post(
  "github.webhooks",
  validateWebhook({ ... }),
  async (ctx) => {
    await new GitHubWebhookTask().schedule({
      payload: body,
      headers,  // 包含 X-GitHub-Delivery
    });
    
    ctx.status = 202;  // 立即返回 Accepted
  }
);

// 为什么 202 Accepted 能帮助幂等:
// 1. GitHub 收到 2xx 状态码就不会重试
// 2. 即使任务执行失败，GitHub 也不会知道，不会重新发送
// 3. 避免了"请求超时 → 重试 → 重复处理"的问题
```

#### 去重建议（当前实现未包含）

```typescript
// 如果需要更强的幂等保证，可以:

// 1. 使用 X-GitHub-Delivery 作为幂等键
const deliveryId = headers["x-github-delivery"];

// 2. 检查是否已处理
const cacheKey = `github:webhook:${deliveryId}`;
const exists = await redis.get(cacheKey);

if (exists) {
  ctx.status = 202;
  return;
}

// 3. 标记为处理中（带过期时间）
await redis.set(cacheKey, "processing", "EX", 3600);  // 1 小时过期

// 4. 执行任务
await new GitHubWebhookTask().schedule({ payload, headers });

// 5. 标记为完成
await redis.set(cacheKey, "completed", "EX", 86400);  // 保留 1 天
```

### 9.5 Linear Token 刷新幂等性

**文件位置**: `server/models/IntegrationAuthentication.ts`

Linear Token 刷新使用**双重检查锁定 + 行级锁**实现幂等和并发安全：

#### 双重检查锁定原理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              Linear Token 刷新双重检查锁定原理                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  场景: 同一用户的多个请求同时触发 Token 刷新                                   │
│                                                                              │
│  请求A                     请求B                                              │
│    │                          │                                               │
│    ▼                          ▼                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  ========== 第一次检查（无锁，快速返回）==========                       │ │
│  │                                                                          │ │
│  │  请求A: this.isExpiringSoon() → true                                    │ │
│  │  请求B: this.isExpiringSoon() → true                                    │ │
│  │                                                                          │ │
│  │  两个请求都判断需要刷新                                                  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│    │                          │                                               │
│    ▼                          ▼                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  ========== 进入事务 + 行级锁 ==========                                │ │
│  │                                                                          │ │
│  │  请求A: 启动事务，请求行级锁                                             │ │
│  │  请求B: 启动事务，请求行级锁                                             │ │
│  │                                                                          │ │
│  │  数据库层面:                                                             │ │
│  │  - 请求A 先获得锁                                                        │ │
│  │  - 请求B 等待...                                                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│    │                          │                                               │
│    ▼                          │ (等待)                                        │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  ========== 第二次检查（锁内）==========                                │ │
│  │                                                                          │ │
│  │  请求A 获得锁后:                                                         │ │
│  │  ┌────────────────────────────────────────────────────────────────────┐ │ │
│  │  │ const lockedAuth = await this.findByPk(this.id, {                   │ │ │
│  │  │   transaction,                                                        │ │ │
│  │  │   lock: transaction.LOCK.UPDATE,  // 行级锁                         │ │ │
│  │  │   rejectOnEmpty: true,                                                │ │ │
│  │  │ });                                                                   │ │ │
│  │  │                                                                      │ │ │
│  │  │ // 再次检查是否需要刷新                                               │ │ │
│  │  │ if (lockedAuth.isExpiringSoon(thresholdMs) && lockedAuth.refreshToken) { │ │ │
│  │  │   // 执行刷新                                                        │ │ │
│  │  │   const tokenResponse = await refreshCallback(lockedAuth.refreshToken); │ │ │
│  │  │                                                                      │ │ │
│  │  │   await lockedAuth.update({                                          │ │ │
│  │  │     token: tokenResponse.access_token,                                │ │ │
│  │  │     expiresAt: ...,  // 更新过期时间                                 │ │ │
│  │  │   }, { transaction });                                                │ │ │
│  │  │ }                                                                     │ │ │
│  │  └────────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                          │ │
│  │  请求A 提交事务，释放锁                                                  │ │
│  │  - token 已更新                                                          │ │
│  │  - expiresAt 已延长                                                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│    │                          │                                               │
│    │                          ▼                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  ========== 请求B 获得锁 ==========                                     │ │
│  │                                                                          │ │
│  │  请求B 获得锁后:                                                         │ │
│  │  ┌────────────────────────────────────────────────────────────────────┐ │ │
│  │  │ const lockedAuth = await this.findByPk(this.id, {                   │ │ │
│  │  │   transaction,                                                        │ │ │
│  │  │   lock: transaction.LOCK.UPDATE,                                     │ │ │
│  │  │   rejectOnEmpty: true,                                                │ │ │
│  │  │ });                                                                   │ │ │
│  │  │                                                                      │ │ │
│  │  │ // 再次检查                                                           │ │ │
│  │  │ if (lockedAuth.isExpiringSoon(thresholdMs) && lockedAuth.refreshToken) { │ │ │
│  │  │   // 结果: false！                                                    │ │ │
│  │  │   // 因为请求A 已经更新了 token 和 expiresAt                        │ │ │
│  │  │   // lockedAuth.isExpiringSoon() 返回 false                          │ │ │
│  │  │ } else {                                                              │ │ │
│  │  │   // 已被其他进程刷新，使用当前值                                     │ │ │
│  │  │   refreshedToken = lockedAuth.token;                                  │ │ │
│  │  │ }                                                                     │ │ │
│  │  └────────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                          │ │
│  │  结果:                                                                    │ │
│  │  - 请求A: 实际执行了刷新                                                 │ │
│  │  - 请求B: 跳过刷新，直接使用请求A 刷新后的 token                        │ │
│  │                                                                          │ │
│  │  幂等性保证:                                                             │ │
│  │  无论多少个并发请求同时触发刷新                                          │ │
│  │  最终只有一个请求实际执行刷新操作                                        │ │
│  │  其他请求直接使用已刷新的 token                                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 代码实现细节

```typescript
async refreshTokenIfNeeded(
  refreshCallback: TokenRefreshCallback,
  thresholdMs: number = 5 * Minute.ms
): Promise<string> {
  let refreshedToken = this.token;
  
  // ========== 快速检查（无锁）==========
  // 这一步是性能优化，避免不必要的数据库操作
  if (!this.isExpiringSoon(thresholdMs) || !this.refreshToken) {
    return this.token;
  }
  
  // ========== 进入事务 + 行级锁 ==========
  await this.sequelize.transaction(async (transaction) => {
    // 使用 SELECT ... FOR UPDATE 获取行级锁
    const lockedAuth = await this.findByPk(this.id, {
      transaction,
      lock: transaction.LOCK.UPDATE,  // 关键：行级锁
      rejectOnEmpty: true,
    });
    
    // ========== 双重检查（锁内）==========
    // 这一步是关键，防止以下情况:
    // 1. 请求A 和 请求B 都通过了快速检查
    // 2. 请求A 先获得锁，执行刷新
    // 3. 请求A 释放锁
    // 4. 请求B 获得锁
    // 5. 如果没有这步检查，请求B 会再次刷新
    if (lockedAuth.isExpiringSoon(thresholdMs) && lockedAuth.refreshToken) {
      const tokenResponse = await refreshCallback(lockedAuth.refreshToken);
      
      await lockedAuth.update(
        {
          token: tokenResponse.access_token,
          refreshToken: tokenResponse.refresh_token || lockedAuth.refreshToken,
          expiresAt: addSeconds(Date.now(), tokenResponse.expires_in),
        },
        { transaction }
      );
      
      refreshedToken = tokenResponse.access_token;
    } else {
      // 已被其他进程刷新，使用当前值
      refreshedToken = lockedAuth.token;
    }
  });
  
  return refreshedToken;
}
```

### 9.6 幂等性机制总览

| 组件 | 幂等机制 | 实现方式 | 失败处理 | 代码位置 |
|------|----------|----------|----------|----------|
| **通用 Webhook 投递** | Delivery ID + 时间戳签名 | `WebhookPresentation.id` + `Outline-Signature` | 无（依赖接收方） | `plugins/webhooks/server/presenters/webhook.ts` |
| **通用 Webhook 禁用** | 失败率熔断 | 时间窗口 + 失败率计算 | 自动禁用 + 邮件通知 | `plugins/webhooks/server/tasks/DeliverWebhookTask.ts` |
| **Slack 推送** | 业务规则约束 | 时间窗口 + 草稿/导入过滤 | 无（失败则丢失） | `plugins/slack/server/processors/SlackProcessor.ts` |
| **GitHub Webhook** | 异步任务 + 202 Accepted | `ctx.status = 202` | 无（依赖 GitHub 不重试） | `plugins/github/server/api/github.ts` |
| **Linear Token 刷新** | 双重检查锁定 + 行级锁 | `transaction.LOCK.UPDATE` | 幂等（并发安全） | `server/models/IntegrationAuthentication.ts` |
| **数据库操作** | 事务 + 乐观锁 | Sequelize `transaction` + `version` 字段 | 回滚 + 错误抛出 | `server/models/base/Model.ts` |

---

## 10. 状态一致性与边界条件

### 10.1 状态一致性模型

#### 10.1.1 Webhook 投递状态机

**文件位置**: `server/models/WebhookDelivery.ts`

```typescript
// WebhookDelivery 状态定义
type WebhookDeliveryStatus = "pending" | "success" | "failed";

// 状态流转:
// 
//  1. 创建时 → pending
//  2. 发送成功 → success
//  3. 发送失败 → failed
//
//                    ┌───────────┐
//                    │  创建    │
//                    └─────┬─────┘
//                          │
//                          ▼
//                    ┌───────────┐
//                    │  pending  │
//                    └─────┬─────┘
//                          │
//              ┌───────────┴───────────┐
//              │                       │
//              ▼                       ▼
//        ┌───────────┐           ┌───────────┐
//        │  success  │           │  failed   │
//        └───────────┘           └─────┬─────┘
//                                      │
//                                      ▼
//                              ┌───────────────┐
//                              │ 检查失败率    │
//                              │ ───────────── │
//                              │ 失败率 >= 阈值 │
//                              │ 且 投递数 >= 10│
//                              └───────┬───────┘
//                                      │
//                                      ▼
//                              ┌───────────────┐
//                              │ 禁用订阅 +   │
//                              │ 邮件通知     │
//                              └───────────────┘
```

#### 10.1.2 状态更新原子性

**文件位置**: `plugins/webhooks/server/tasks/DeliverWebhookTask.ts`

```typescript
private async sendWebhook({ event, subscription, payload }) {
  // ========== 步骤1: 创建投递记录（pending 状态）==========
  const delivery = await WebhookDelivery.create({
    webhookSubscriptionId: subscription.id,
    status: "pending",
  });

  try {
    // ========== 步骤2: 发送请求 ==========
    response = await fetch(subscription.url, {
      method: "POST",
      headers: requestHeaders,
      body: JSON.stringify(requestBody),
      redirect: "error",
      timeout: 5000,
    });
    status = response.ok ? "success" : "failed";
  } catch (err) {
    status = "failed";
  }

  // ========== 步骤3: 更新状态（原子操作）==========
  await delivery.update({
    status,
    statusCode: response ? response.status : null,
    requestBody,
    requestHeaders,
    responseBody,
    responseHeaders,
  });

  // ========== 步骤4: 副作用处理（如果失败）==========
  if (status === "failed") {
    await this.checkAndDisableSubscription(subscription);
  }
}
```

#### 10.1.3 重试失败后的状态一致性

**问题分析**:

| 场景 | 问题 | 当前处理 | 建议改进 |
|------|------|----------|----------|
| **Slack 推送失败** | 消息丢失，无重试 | 直接丢弃 | 加入任务队列，指数退避重试 |
| **通用 Webhook 失败** | 单条失败不重试，仅累计失败率 | 累计失败，到达阈值禁用订阅 | 可增加单条重试机制 |
| **GitHub Webhook 任务失败** | 202 返回后任务失败无重试 | 任务队列自动重试（5次） | 已有重试机制 |
| **Linear Token 刷新失败** | 并发时可能使用过期 token | 双重检查锁定 + 行级锁 | 已有保障 |

**通用 Webhook 禁用逻辑**:

```typescript
private async checkAndDisableSubscription(subscription: WebhookSubscription) {
  // 1. 计算时间窗口
  const timeWindowSeconds = env.WEBHOOK_FAILURE_TIME_WINDOW;
  const failureRateThreshold = env.WEBHOOK_FAILURE_RATE_THRESHOLD;
  const timeWindowStart = new Date(Date.now() - timeWindowSeconds * 1000);

  // 2. 获取时间窗口内的所有投递记录
  const deliveriesInWindow = await WebhookDelivery.findAll({
    where: {
      webhookSubscriptionId: subscription.id,
      createdAt: { [Op.gte]: timeWindowStart },
    },
  });

  // 3. 计算失败率
  const failedDeliveries = deliveriesInWindow.filter(
    (delivery) => delivery.status === "failed"
  );
  const failureRate =
    (failedDeliveries.length / deliveriesInWindow.length) * 100;

  // 4. 禁用条件:
  //    - 失败率 >= 阈值
  //    - 投递数 >= 10（有足够数据点）
  if (
    failureRate >= failureRateThreshold &&
    deliveriesInWindow.length >= DeliverWebhookTask.MIN_DELIVERIES_FOR_ANALYSIS
  ) {
    // ========== 状态一致性保障 ==========
    // 禁用订阅和发送邮件通知在同一逻辑流中
    // 但不是同一数据库事务

    // 步骤A: 禁用订阅（数据库操作）
    await subscription.disable();

    // 步骤B: 获取创建者和团队信息
    const [createdBy, team] = await Promise.all([
      User.findOne({
        where: {
          id: subscription.createdById,
          suspendedAt: { [Op.is]: null },
        },
      }),
      subscription.$get("team"),
    ]);

    // 步骤C: 发送邮件通知（异步任务）
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

### 10.2 边界条件分析

#### 10.2.1 网络边界条件

| 边界条件 | 触发场景 | 当前处理 | 风险等级 | 代码位置 |
|----------|----------|----------|----------|----------|
| **请求超时** | 网络延迟、外部系统过载 | `timeout: 5000` 毫秒 | 中 | `DeliverWebhookTask.ts:763` |
| **重定向** | URL 变更、负载均衡 | `redirect: "error"` 禁止重定向 | 低 | `DeliverWebhookTask.ts:762` |
| **响应体过大** | 外部系统返回大量数据 | 截断为 `1024` 字节 | 低 | `DeliverWebhookTask.ts:722` |
| **连接失败** | DNS 解析失败、端口未开放 | 捕获异常，标记为失败 | 中 | `DeliverWebhookTask.ts:766-779` |
| **SSL 错误** | 证书过期、证书不匹配 | 捕获异常，标记为失败 | 高 | `DeliverWebhookTask.ts:766-779` |

#### 10.2.2 数据边界条件

| 边界条件 | 触发场景 | 当前处理 | 风险等级 | 代码位置 |
|----------|----------|----------|----------|----------|
| **订阅数量限制** | 创建新 Webhook 订阅 | `maxSubscriptions` 限制，抛出错误 | 低 | `WebhookSubscription.ts:96-106` |
| **事件名称长度** | 事件名过长 | 数据库约束 `max: 255` | 低 | `Event.ts:42-47` |
| **JSON 序列化** | 循环引用、特殊字符 | 失败时标记为 `failed` | 中 | `DeliverWebhookTask.ts:744-756` |
| **空值处理** | 模型已删除、关联丢失 | `findByPk` + null 检查 | 中 | 多处 |
| **并行更新** | 同一 Token 多请求刷新 | 双重检查锁定 + 行级锁 | 低 | `IntegrationAuthentication.ts` |

#### 10.2.3 时间边界条件

| 边界条件 | 触发场景 | 当前处理 | 风险等级 | 代码位置 |
|----------|----------|----------|----------|----------|
| **Token 过期** | OAuth Access Token 过期 | `refreshTokenIfNeeded()` 自动刷新 | 低 | `IntegrationAuthentication.ts` |
| **时间窗口计算** | 失败率分析时间边界 | 滑动时间窗口 | 低 | `DeliverWebhookTask.ts:819-832` |
| **发布后更新合并** | 草稿发布后快速编辑 | `differenceInMilliseconds < Minute.ms` | 低 | `SlackProcessor.ts:101-109` |
| **摘要生成延迟** | 文档发布后摘要异步生成 | `await sleep(5000)` | 中 | `SlackProcessor.ts:30` |
| **时钟回拨** | NTP 同步、服务器重启 | 无特殊处理 | 高 | 无 |

### 10.3 详细边界条件分析

#### 10.3.1 Slack 推送边界条件

**文件位置**: `plugins/slack/server/processors/SlackProcessor.ts`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Slack 推送边界条件分析                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  边界条件 1: 文档已删除                                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  async documentUpdated(event) {                                          │ │
│  │    const [document, team] = await Promise.all([                         │ │
│  │      Document.findByPk(event.documentId),  // 可能返回 null              │ │
│  │      Team.findByPk(event.teamId),                                        │ │
│  │    ]);                                                                    │ │
│  │                                                                          │ │
│  │    if (!document || !team) {             // 边界条件检查                  │ │
│  │      return;                              // 静默处理，不报错             │ │
│  │    }                                                                      │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  边界条件 2: 集成已删除/禁用                                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  const integration = await Integration.findOne({                        │ │
│  │    where: {                                                             │ │
│  │      teamId: document.teamId,                                           │ │
│  │      collectionId: document.collectionId,                               │ │
│  │      service: IntegrationService.Slack,                                 │ │
│  │      type: IntegrationType.Post,                                        │ │
│  │      events: { [Op.contains]: [...] },                                  │ │
│  │    },                                                                    │ │
│  │  });                                                                     │ │
│  │                                                                          │ │
│  │  if (!integration) {                    // 无匹配集成                      │ │
│  │    return;                              // 静默处理                       │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  边界条件 3: 5 秒延迟后状态变化                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  async perform(event) {                                                  │ │
│  │    switch (event.name) {                                                 │ │
│  │      case "documents.publish":                                           │ │
│  │      case "revisions.create":                                            │ │
│  │        await sleep(5000);  // 智能延迟 5 秒                            │ │
│  │        return this.documentUpdated(event);                               │ │
│  │    }                                                                      │ │
│  │  }                                                                       │ │
│  │                                                                          │ │
│  │  // 风险点:                                                              │ │
│  │  // - 5 秒内文档可能已被删除                                             │ │
│  │  // - 5 秒内集合可能已被修改                                             │ │
│  │  // - 5 秒内集成可能已被删除                                             │ │
│  │  //                                                                      │ │
│  │  // 处理方式:                                                             │ │
│  │  // documentUpdated() 中有 null 检查                                    │ │
│  │  // 但没有检查文档状态是否仍符合推送条件                                 │ │
│  │  // （如文档是否仍在原集合、是否仍已发布等）                             │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  边界条件 4: Slack Incoming Webhook 调用失败                                │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  await fetch(integration.settings.url, {                                │ │
│  │    method: "POST",                                                       │ │
│  │    headers: { "Content-Type": "application/json" },                     │ │
│  │    body: JSON.stringify({ ... }),                                        │ │
│  │  });  // 没有 try-catch！                                                 │ │
│  │                                                                          │ │
│  │  // 风险:                                                                │ │
│  │  // - fetch 失败会抛出异常                                               │ │
│  │  // - 异常会向上传播，可能导致处理器失败                                 │ │
│  │  // - 没有重试机制，消息丢失                                             │ │
│  │                                                                          │ │
│  │  // 与通用 Webhook 对比:                                                 │ │
│  │  // DeliverWebhookTask 有完整的 try-catch                              │ │
│  │  │ 状态记录                                                              │ │
│  │  │ 失败率累积                                                            │ │
│  │  │ 自动禁用                                                              │ │
│  │  // SlackProcessor 都没有这些保障                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 10.3.2 GitHub Webhook 边界条件

**文件位置**: `plugins/github/server/api/github.ts`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GitHub Webhook 边界条件分析                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  边界条件 1: 验签失败                                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  validateWebhook({                                                       │ │
│  │    secretKey: env.GITHUB_WEBHOOK_SECRET!,                              │ │
│  │    getSignatureFromHeader: (ctx) => {                                   │ │
│  │      const signatureHeader = headers["x-hub-signature-256"];          │ │
│  │      const signature = Array.isArray(signatureHeader)                  │ │
│  │        ? signatureHeader[0]                                              │ │
│  │        : signatureHeader;                                                │ │
│  │      return signature?.split("=")[1];                                   │ │
│  │    },                                                                    │ │
│  │  })                                                                      │ │
│  │                                                                          │ │
│  │  验签失败处理:                                                           │ │
│  │  - ctx.status = 401                                                     │ │
│  │  - ctx.body = "Invalid signature"                                       │ │
│  │  - 不执行后续处理                                                        │ │
│  │                                                                          │ │
│  │  边界情况:                                                               │ │
│  │  - 缺少签名头: 401 "Missing signature header"                           │ │
│  │  - 签名头格式错误: 401 "Invalid signature"                              │ │
│  │  - 签名不匹配: 401 "Invalid signature"                                  │ │
│  │  - 环境变量未设置: 401 (密钥获取失败)                                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  边界条件 2: 异步任务调度                                                     │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  router.post(                                                            │ │
│  │    "github.webhooks",                                                    │ │
│  │    validateWebhook(...),                                                 │ │
│  │    async (ctx) => {                                                      │ │
│  │      await new GitHubWebhookTask().schedule({                           │ │
│  │        payload: body,                                                    │ │
│  │        headers,                                                          │ │
│  │      });                                                                 │ │
│  │                                                                          │ │
│  │      ctx.status = 202;  // 立即返回 Accepted                            │ │
│  │    }                                                                     │ │
│  │  );                                                                      │ │
│  │                                                                          │ │
│  │  风险点:                                                                 │ │
│  │  - 任务队列可能满，schedule() 可能失败                                   │ │
│  │  - schedule() 失败但 202 已返回，GitHub 不会重试                        │ │
│  │  - 事件丢失                                                              │ │
│  │                                                                          │ │
│  │  与同步处理的对比:                                                       │ │
│  │  // 同步处理:                                                            │ │
│  │  // try {                                                                │ │
│  │  //   await processTask();                                               │ │
│  │  //   ctx.status = 200;                                                 │ │
│  │  // } catch (err) {                                                      │ │
│  │  //   ctx.status = 500;                                                 │ │
│  │  //   // GitHub 会重试                                                   │ │
│  │  // }                                                                    │ │
│  │                                                                          │ │
│  │  // 当前异步处理:                                                        │ │
│  │  // 优点: 避免超时导致 GitHub 重复发送                                   │ │
│  │  // 缺点: 任务调度失败时，事件丢失，无重试                               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  边界条件 3: 事件类型不支持                                                   │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  // GitHubIssueProvider.handleWebhook() 只处理:                        │ │
│  │  // - "installation" (action: "new_permissions_accepted")              │ │
│  │  // - "installation_repositories"                                        │ │
│  │  // - "repository" (action: "renamed")                                  │ │
│  │                                                                          │ │
│  │  // 其他事件类型:                                                        │ │
│  │  // - 静默忽略                                                           │ │
│  │  // - 不报错                                                             │ │
│  │  // - 返回 202 Accepted                                                  │ │
│  │                                                                          │ │
│  │  // 这是设计选择:                                                        │ │
│  │  // - GitHub 可能发送多种事件类型                                         │ │
│  │  // - Outline 只处理需要的事件                                           │ │
│  │  // - 忽略不影响功能                                                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 10.3.3 Linear 链接展开边界条件

**文件位置**: `plugins/linear/server/linear.ts`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Linear 链接展开边界条件分析                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  边界条件 1: 无集成匹配                                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  static unfurl: UnfurlSignature = async (url, actor) => {              │ │
│  │    const integrations = await Integration.scope("withAuthentication")   │ │
│  │      .findAll({                                                          │ │
│  │        where: {                                                          │ │
│  │          service: IntegrationService.Linear,                             │ │
│  │          teamId: actor.teamId,                                           │ │
│  │        },                                                                 │ │
│  │      });                                                                  │ │
│  │                                                                          │ │
│  │    if (integrations.length === 0) {   // 无集成                         │ │
│  │      return;                          // 静默返回                         │ │
│  │    }                                                                      │ │
│  │                                                                          │ │
│  │    // 优先匹配工作区                                                      │ │
│  │    const integration =                                                    │ │
│  │      integrations.find(                                                  │ │
│  │        (int) => int.settings.linear?.workspace.key === resource.workspaceKey │ │
│  │      ) ?? integrations[0];                                              │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  边界条件 2: Token 刷新失败                                                   │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  try {                                                                    │ │
│  │    const accessToken = await integration.authentication.refreshTokenIfNeeded( │ │
│  │      async (refreshToken: string) => Linear.refreshToken(refreshToken),  │ │
│  │      5 * Minute.ms                                                        │ │
│  │    );                                                                     │ │
│  │                                                                          │ │
│  │    // 使用 token 调用 Linear API                                         │ │
│  │    const client = new LinearClient({ accessToken });                    │ │
│  │    const issue = await client.issue(id);                                 │ │
│  │  } catch (err) {                                                          │ │
│  │    Logger.warn("Failed to fetch resource from Linear", err);            │ │
│  │    return { error: err.message || "Unknown error" };  // 返回错误信息   │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  边界条件 3: 资源不存在/无权限                                                │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  private static async unfurlIssue(client, id, actor) {                  │ │
│  │    const issue = await client.issue(id);                                 │ │
│  │                                                                          │ │
│  │    if (!issue) {                        // 资源不存在                      │ │
│  │      return { error: "Resource not found" };                            │ │
│  │    }                                                                      │ │
│  │                                                                          │ │
│  │    // 获取关联数据                                                       │ │
│  │    const [author, state, labels] = await Promise.all([                 │ │
│  │      issue.creator,                                                      │ │
│  │      issue.state,                                                        │ │
│  │      issue.paginate((args) => issue.labels(args), {}),                  │ │
│  │    ]);                                                                    │ │
│  │                                                                          │ │
│  │    if (!state || !labels) {              // 辅助数据获取失败              │ │
│  │      return { error: "Failed to fetch auxiliary data from Linear" };     │ │
│  │    }                                                                      │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  边界条件 4: URL 解析失败                                                     │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  private static parseUrl(url: string) {                                 │ │
│  │    try {                                                                  │ │
│  │      const { hostname, pathname } = new URL(url);                        │ │
│  │                                                                          │ │
│  │      if (hostname !== "linear.app") {     // 非 Linear 域名              │ │
│  │        return;                                                           │ │
│  │      }                                                                    │ │
│  │                                                                          │ │
│  │      const parts = pathname.split("/");                                  │ │
│  │      const type = parts[2] as UnfurlResourceType;                       │ │
│  │                                                                          │ │
│  │      if (!type || !Linear.supportedUnfurls.includes(type)) {             │ │
│  │        return;                          // 不支持的资源类型               │ │
│  │      }                                                                    │ │
│  │                                                                          │ │
│  │      return { workspaceKey, type, id, name };                           │ │
│  │    } catch (_err) {                                                       │ │
│  │      return;                            // URL 格式错误                   │ │
│  │    }                                                                      │ │
│  │  }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.4 状态一致性保障机制

#### 10.4.1 事务保障

```typescript
// 关键事务使用场景:

// 1. Token 刷新（双重检查锁定 + 行级锁）
await this.sequelize.transaction(async (transaction) => {
  const lockedAuth = await this.findByPk(this.id, {
    transaction,
    lock: transaction.LOCK.UPDATE,
    rejectOnEmpty: true,
  });
  
  if (lockedAuth.isExpiringSoon(thresholdMs) && lockedAuth.refreshToken) {
    const tokenResponse = await refreshCallback(lockedAuth.refreshToken);
    await lockedAuth.update({ token, refreshToken, expiresAt }, { transaction });
  }
});

// 2. 集成创建（OAuth 回调）
await this.sequelize.transaction(async (transaction) => {
  const authentication = await IntegrationAuthentication.create({ ... }, { transaction });
  await Integration.createWithCtx(createContext({ user, transaction }), {
    authenticationId: authentication.id,
    ...
  });
});

// 3. 事件触发（事务感知）
@AfterSave
static async enqueue(model: Event, options: SaveOptions) {
  if (options.transaction) {
    (options.transaction.parent || options.transaction).afterCommit(
      () => void globalEventQueue().add(model)
    );
    return;
  }
  void globalEventQueue().add(model);
}
```

#### 10.4.2 补偿机制

| 场景 | 补偿机制 | 实现方式 | 代码位置 |
|------|----------|----------|----------|
| **Webhook 高失败率** | 禁用订阅 + 通知 | 失败率计算 → `subscription.disable()` → 邮件通知 | `DeliverWebhookTask.ts:817-895` |
| **Token 刷新失败** | 使用现有 Token | 双重检查锁定中如果刷新失败，使用 `lockedAuth.token` | `IntegrationAuthentication.ts:155-162` |
| **链接展开失败** | 返回错误信息 | `catch` 块返回 `{ error: message }` | `linear.ts:157-159` |
| **文档已删除** | 静默处理 | `!document` 时 `return` | `SlackProcessor.ts:92-94` |

#### 10.4.3 缺失的保障

| 场景 | 当前状态 | 风险 | 建议改进 |
|------|----------|------|----------|
| **Slack 推送重试** | 无重试 | 消息永久丢失 | 加入任务队列，使用指数退避 |
| **通用 Webhook 单条重试** | 无单条重试 | 单条失败不重试，仅累计 | 增加 `attempts` 字段，单条重试 |
| **GitHub 任务调度失败** | 无处理 | 事件丢失 | `schedule()` 失败时记录日志，考虑重试 |
| **时钟回拨** | 无处理 | Token 刷新逻辑异常、时间窗口计算错误 | 使用单调时钟或增加时间同步检测 |
| **部分成功** | 无原子性 | 订阅禁用但邮件发送失败 | 将邮件发送也纳入异步任务，确保最终执行 |

---

## 11. 关键设计模式总结

### 11.1 架构模式

#### 1. 插件化架构

**核心组件**: `PluginManager` + `Hook` 机制

**文件位置**: `server/utils/PluginManager.ts`

```typescript
// Hook 类型
enum Hook {
  IssueProvider = "issueProvider",
  Processor = "processor",
  Unfurl = "unfurl",
  // ...
}

// 注册 Hook
PluginManager.addHook(Hook.IssueProvider, {
  service: IntegrationService.GitHub,
  component: GitHubButton,
  handleWebhook: (params) => new GitHubIssueProvider().handleWebhook(params),
  fetchSources: (integration) => new GitHubIssueProvider().fetchSources(integration),
});

// 使用 Hook
const plugins = PluginManager.getHooks(Hook.IssueProvider);
const plugin = plugins.find(
  (p) => p.value.service === IntegrationService.GitHub
);
await plugin.value.handleWebhook({ headers, payload });
```

**优势**:
- 灵活扩展：新增集成只需添加插件
- 按需加载：仅在需要时初始化组件
- 接口统一：所有同类型插件遵循相同接口

#### 2. 事件驱动架构

**核心组件**: `Event` 模型 + `BaseProcessor` + Bull 队列

```typescript
// 事件定义
interface Event {
  name: string;           // 事件名: documents.create
  modelId: UUID;          // 关联模型 ID
  teamId: UUID;           // 团队 ID
  actorId: UUID;          // 操作者 ID
  data: JSON;             // 额外数据
  changes: JSON;          // 变更字段
}

// 处理器定义
abstract class BaseProcessor {
  static applicableEvents: (Event["name"] | "*")[] = [];
  abstract perform(event: Event): Promise<void>;
}

// 示例处理器
class SlackProcessor extends BaseProcessor {
  static applicableEvents = [
    "documents.publish",
    "revisions.create",
    "integrations.create",
  ];
  
  async perform(event: Event) {
    // 处理事件...
  }
}
```

**数据流**:
```
模型操作 → 生命周期钩子 → Event.create → afterCommit → 队列 → 处理器
```

**优势**:
- 解耦：事件产生者和消费者解耦
- 异步：非阻塞处理
- 重试：队列内置重试机制
- 可观测：易于监控和调试

#### 3. 策略模式

**应用场景**: 不同集成的认证、展开、Webhook 处理

**示例**: `refreshTokenIfNeeded` 方法

```typescript
// 策略接口
type TokenRefreshCallback = (
  refreshToken: string
) => Promise<{
  access_token: string;
  refresh_token?: string;
  expires_in?: number;
}>;

// 策略使用
async refreshTokenIfNeeded(
  refreshCallback: TokenRefreshCallback,  // 策略函数
  thresholdMs: number = 5 * Minute.ms
): Promise<string> {
  // 通用逻辑...
  
  // 调用策略函数
  const tokenResponse = await refreshCallback(lockedAuth.refreshToken);
  
  // 通用逻辑...
}

// Linear 集成的策略
const accessToken = await integration.authentication.refreshTokenIfNeeded(
  async (refreshToken: string) => Linear.refreshToken(refreshToken),  // 策略
  5 * Minute.ms
);
```

**优势**:
- 可扩展：新增集成只需实现策略函数
- 可测试：策略函数可单独测试
- 关注点分离：通用逻辑和特定逻辑分离

### 11.2 并发安全模式

#### 1. 双重检查锁定 + 行级锁

**应用场景**: Token 刷新、集成配置更新

**代码位置**: `server/models/IntegrationAuthentication.ts`

```typescript
async refreshTokenIfNeeded(
  refreshCallback: TokenRefreshCallback,
  thresholdMs: number = 5 * Minute.ms
): Promise<string> {
  // ========== 第一次检查（无锁，快速返回）==========
  if (!this.isExpiringSoon(thresholdMs) || !this.refreshToken) {
    return this.token;
  }
  
  // ========== 进入事务 + 行级锁 ==========
  await this.sequelize.transaction(async (transaction) => {
    const lockedAuth = await this.findByPk(this.id, {
      transaction,
      lock: transaction.LOCK.UPDATE,  // 行级锁防止并发
      rejectOnEmpty: true,
    });
    
    // ========== 第二次检查（锁内）==========
    if (lockedAuth.isExpiringSoon(thresholdMs) && lockedAuth.refreshToken) {
      // 执行刷新
      const tokenResponse = await refreshCallback(lockedAuth.refreshToken);
      
      // 更新记录
      await lockedAuth.update(
        {
          token: tokenResponse.access_token,
          refreshToken: tokenResponse.refresh_token || lockedAuth.refreshToken,
          expiresAt: addSeconds(Date.now(), tokenResponse.expires_in),
        },
        { transaction }
      );
    } else {
      // 已被其他进程刷新
      refreshedToken = lockedAuth.token;
    }
  });
  
  return refreshedToken;
}
```

**解决的问题**:
- **竞态条件**: 多个请求同时刷新同一 token
- **双重检查**: 减少不必要的锁竞争
- **行级锁**: 确保同一时间只有一个进程刷新

#### 2. 乐观锁（Sequelize 内置）

**应用场景**: 模型更新

**Sequelize 提供**:
- 版本号字段 (`version`)
- 并发时抛出 `OptimisticLockError`

**优势**:
- 无锁开销
- 适用于冲突较少的场景

### 11.3 安全模式

#### 1. 时序攻击防护

**核心机制**: `safeEqual` (时间常数比较)

**代码位置**: `server/utils/crypto.ts`

```typescript
// 不安全的比较（时序攻击）
if (a === b) { ... }

// 安全的比较
if (safeEqual(a, b)) { ... }

// 实现原理
export function safeEqual(a: string, b: string): boolean {
  if (a.length !== b.length) {
    return false;
  }
  
  let result = 0;
  for (let i = 0; i < a.length; i++) {
    result |= a.charCodeAt(i) ^ b.charCodeAt(i);
  }
  
  return result === 0;
}
```

**应用场景**:
- Webhook 签名验证
- Token 验证
- 密码比较（虽然 Outline 使用 bcrypt）

#### 2. 数据加密

**核心机制**: `@Encrypted` 装饰器

**代码位置**: `server/models/decorators/Encrypted.ts`

```typescript
class IntegrationAuthentication extends Model {
  @Column(DataType.BLOB)
  @Encrypted
  token: string;
  
  @Column(DataType.BLOB)
  @Encrypted
  refreshToken: string;
  
  @AllowNull
  @Column(DataType.BLOB)
  @Encrypted
  clientId: string | null;
  
  @AllowNull
  @Column(DataType.BLOB)
  @Encrypted
  clientSecret: string | null;
}
```

**工作原理**:
- `@Encrypted` 装饰器注册 `getter` 和 `setter`
- 写入时加密，读取时解密
- 使用 AES-256-GCM 加密

#### 3. CSRF 防护

**核心机制**: OAuth State + Nonce Cookie

**代码位置**: `server/utils/oauth.ts`

```typescript
// 生成 state
const nonce = generateOAuthStateNonce(ctx, OAuthNonceCookie);

// 验证 state
const parsedState = LinearUtils.parseState(state);
if (!parsedState) {
  throw ValidationError("Invalid state");
}
verifyOAuthStateNonce(ctx, LinearOAuthNonceCookie, parsedState.nonce);
```

**工作原理**:
1. 生成随机 nonce，存储在 cookie 中
2. nonce 编码在 OAuth state 参数中
3. 回调时比较 cookie 和 state 中的 nonce

### 11.4 错误处理模式

#### 1. 指数退避重试

**核心机制**: Bull 队列的 backoff 配置

```typescript
// 默认配置
public get options(): JobOptions {
  return {
    priority: TaskPriority.Normal,
    attempts: 5,
    backoff: {
      type: "exponential",
      delay: 60 * 1000,
    },
  };
}
```

**重试间隔**:
```
第 1 次失败: 等待 60 秒
第 2 次失败: 等待 120 秒 (2^1 * 60)
第 3 次失败: 等待 240 秒 (2^2 * 60)
第 4 次失败: 等待 480 秒 (2^3 * 60)
第 5 次失败: 任务失败
```

**优势**:
- 降低服务器压力
- 给外部系统恢复时间
- 可配置的重试策略

#### 2. 熔断器模式

**应用场景**: Webhook 高失败率禁用

**代码位置**: `plugins/webhooks/server/tasks/DeliverWebhookTask.ts`

```typescript
private async checkAndDisableSubscription(subscription: WebhookSubscription) {
  // 1. 收集时间窗口内的投递记录
  const deliveriesInWindow = await WebhookDelivery.findAll({
    where: {
      webhookSubscriptionId: subscription.id,
      createdAt: { [Op.gte]: timeWindowStart },
    },
  });
  
  // 2. 计算失败率
  const failureRate =
    (failedDeliveries.length / deliveriesInWindow.length) * 100;
  
  // 3. 检查条件
  if (
    failureRate >= failureRateThreshold &&
    deliveriesInWindow.length >= MIN_DELIVERIES_FOR_ANALYSIS
  ) {
    // 熔断：禁用订阅
    await subscription.disable();
    
    // 通知管理员
    await new WebhookDisabledEmail({...}).schedule();
  }
}
```

**熔断条件**:
- 失败率 >= 阈值（可配置）
- 时间窗口内投递数 >= 10（有足够数据点）

### 11.5 设计模式总览表

| 模式 | 应用场景 | 核心组件 | 优势 |
|------|----------|----------|------|
| **插件化架构** | 第三方集成 | `PluginManager` + `Hook` | 灵活扩展、按需加载 |
| **事件驱动** | 内部消息传递 | `Event` + `BaseProcessor` + Bull | 解耦、异步、可重试 |
| **策略模式** | 不同集成的实现差异 | `TokenRefreshCallback` 等 | 可扩展、可测试 |
| **双重检查锁定** | Token 刷新、配置更新 | `transaction.LOCK.UPDATE` | 防竞态、减少锁竞争 |
| **时序攻击防护** | 签名验证 | `safeEqual` | 安全比较 |
| **数据加密** | Token 存储 | `@Encrypted` 装饰器 | 安全存储 |
| **指数退避** | 外部调用重试 | Bull backoff | 容错、降低压力 |
| **熔断器** | Webhook 投递 | 失败率计算 + 禁用 | 防止雪崩 |

---

## 12. 附录

### 12.1 环境变量配置

#### Slack 集成

| 变量名 | 说明 | 必填 |
|--------|------|------|
| `SLACK_CLIENT_ID` | OAuth Client ID | 是 |
| `SLACK_CLIENT_SECRET` | OAuth Client Secret | 是 |
| `SLACK_VERIFICATION_TOKEN` | Webhook 验证 Token | 是 |
| `SLACK_MESSAGE_ACTIONS` | 是否启用消息动作按钮 | 否 |

#### GitHub 集成

| 变量名 | 说明 | 必填 |
|--------|------|------|
| `GITHUB_APP_ID` | GitHub App ID | 是 |
| `GITHUB_APP_PRIVATE_KEY` | App 私钥（Base64） | 是 |
| `GITHUB_CLIENT_ID` | OAuth Client ID | 是 |
| `GITHUB_CLIENT_SECRET` | OAuth Client Secret | 是 |
| `GITHUB_WEBHOOK_SECRET` | Webhook 签名密钥 | 是 |

#### Linear 集成

| 变量名 | 说明 | 必填 |
|--------|------|------|
| `LINEAR_CLIENT_ID` | OAuth Client ID | 是 |
| `LINEAR_CLIENT_SECRET` | OAuth Client Secret | 是 |

#### 通用 Webhook

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `WEBHOOK_FAILURE_TIME_WINDOW` | 失败分析时间窗口（秒） | - |
| `WEBHOOK_FAILURE_RATE_THRESHOLD` | 失败率阈值（百分比） | - |

### 12.2 集成对照表

| 维度 | Slack | GitHub | Linear |
|------|-------|--------|--------|
| **主要方向** | 双向交互 | 嵌入为主 | 嵌入为主 |
| **认证方式** | OAuth + Token | GitHub App | OAuth 2.0 |
| **Token 刷新** | 不支持 | 不支持 | **支持** |
| **Webhook 验签** | 简单 Token | HMAC-SHA256 | 无（仅 OAuth） |
| **事件类型** | 文档发布/更新 | 安装/仓库变更 | 无（仅嵌入） |
| **实时性** | Webhook 即时响应 | Webhook 异步任务 | 按需 API 调用 |

### 12.3 关键文件位置

| 功能 | 文件路径 |
|------|----------|
| 集成模型 | `server/models/integration.ts` |
| 认证模型 | `server/models/IntegrationAuthentication.ts` |
| 事件模型 | `server/models/Event.ts` |
| 事件队列 | `server/queues/index.ts` |
| 处理器基类 | `server/queues/processors/BaseProcessor.ts` |
| Webhook 验签 | `server/middlewares/validateWebhook.ts` |
| Slack 集成 | `plugins/slack/server/` |
| GitHub 集成 | `plugins/github/server/` |
| Linear 集成 | `plugins/linear/server/` |
| 通用 Webhook | `plugins/webhooks/server/` |
| 插件管理器 | `server/utils/PluginManager.ts` |

### 12.4 风险评估与建议

#### 高风险项

| 风险项 | 描述 | 建议 |
|--------|------|------|
| **Slack 推送无重试** | 推送失败消息丢失 | 加入任务队列，使用指数退避重试 |
| **时钟回拨** | 影响时间相关逻辑 | 使用单调时钟或增加时间同步检测 |
| **部分成功** | 订阅禁用但邮件失败 | 将邮件发送也纳入异步任务，确保最终执行 |

#### 中风险项

| 风险项 | 描述 | 建议 |
|--------|------|------|
| **GitHub 任务调度失败** | 事件丢失 | 增加失败处理逻辑，考虑重试机制 |
| **5 秒延迟期间状态变化** | Slack 推送时文档/集成已变化 | 延迟后增加状态检查 |
| **通用 Webhook 单条无重试** | 单条失败不重试 | 增加 `attempts` 字段，单条重试 |

#### 低风险项

| 风险项 | 描述 | 建议 |
|--------|------|------|
| **响应体截断** | 响应体限制 1KB | 当前设计合理，已记录日志 |
| **重定向禁止** | 不支持 URL 重定向 | 符合安全要求，文档应使用稳定 URL |
| **草稿忽略** | 草稿不推送 | 符合业务逻辑 |

---

**报告生成时间**: 2026-05-05  
**分析代码版本**: Outline Monorepo (`h:\fz\solo-dogfeeding\code\151-outline`)

**覆盖内容**:
- ✅ 事件触发源与分发机制
- ✅ 入站/出站链路详解
- ✅ 幂等性与去重机制
- ✅ 状态一致性与边界条件
- ✅ 关键设计模式总结
- ✅ 风险评估与建议