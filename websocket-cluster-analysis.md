# Outline WebSocket 多实例集群分析报告

## 1. 概述

Outline 是一个支持多实例部署的协作知识库系统，在多节点部署场景下，WebSocket 连接和实时协作需要通过 Redis pub/sub 机制实现跨实例的消息广播和同步。本文档详细分析 Outline 中 WebSocket 集群的实现机制和消息路由策略。

## 2. 系统架构

### 2.1 整体架构

Outline 的多实例部署架构如下：

```
                    ┌─────────────────┐
                    │   Load Balancer │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
      ┌─────▼─────┐    ┌─────▼─────┐    ┌─────▼─────┐
      │  Instance A│    │  Instance B│    │  Instance C│
      │  (Node.js) │    │  (Node.js) │    │  (Node.js) │
      └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
            │                │                │
            └────────────────┼────────────────┘
                             │
                    ┌────────▼────────┐
                    │     Redis       │
                    │  (Pub/Sub)      │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   PostgreSQL    │
                    └─────────────────┘
```

### 2.2 核心组件

Outline 的实时通信系统包含两个独立的 WebSocket 服务：

| 服务 | 路径 | 用途 | 技术实现 |
|------|------|------|----------|
| WebSocket 事件服务 | `/realtime` | 系统事件广播（文档创建、更新、删除等） | Socket.IO + socket.io-redis |
| 协作编辑服务 | `/collaboration` | 实时文档协作编辑 | Hocuspocus + @hocuspocus/extension-redis |

## 3. WebSocket 事件服务实现

### 3.1 初始化与配置

WebSocket 事件服务使用 Socket.IO 库实现，核心配置在 `server/services/websockets.ts` 中：

```typescript
// 初始化 Socket.IO 服务器
const io = new IO.Server(server, {
  path: "/realtime",
  serveClient: false,
  cookie: false,
  pingInterval: 15000,
  pingTimeout: 30000,
  cors: {
    origin: env.isCloudHosted ? "*" : env.URL,
    methods: ["GET", "POST"],
  },
});
```

### 3.2 Redis 适配器配置

为了实现多实例间的消息广播，Outline 使用 `socket.io-redis` 适配器：

```typescript
// 配置 Redis 适配器，实现跨实例消息广播
io.adapter(
  createAdapter({
    pubClient: Redis.defaultClient,
    subClient: Redis.defaultSubscriber,
  })
);
```

**关键点分析**：
- 使用两个独立的 Redis 客户端：`pubClient` 用于发布消息，`subClient` 用于订阅消息
- `Redis.defaultSubscriber` 配置了 `maxRetriesPerRequest: null`，适合阻塞式的订阅操作

### 3.3 Redis 客户端管理

Redis 客户端管理在 `server/storage/redis.ts` 中实现，提供了三种客户端：

```typescript
export default class RedisAdapter extends Redis {
  private static client: RedisAdapter;
  private static subscriber: RedisAdapter;
  private static collabClient: RedisAdapter;

  // 通用操作客户端
  public static get defaultClient(): RedisAdapter {
    return this.client || (this.client = new this(env.REDIS_URL, {
      connectionNameSuffix: "client",
    }));
  }

  // 订阅专用客户端（不参与健康检查）
  public static get defaultSubscriber(): RedisAdapter {
    return this.subscriber || (this.subscriber = new this(env.REDIS_URL, {
      maxRetriesPerRequest: null,
      connectionNameSuffix: "subscriber",
    }));
  }

  // 协作服务专用客户端（支持独立 Redis 实例）
  public static get collaborationClient(): RedisAdapter {
    if (!env.REDIS_COLLABORATION_URL) {
      return this.defaultClient;
    }
    return this.collabClient || (this.collabClient = new this(env.REDIS_COLLABORATION_URL, {
      connectionNameSuffix: "collab",
    }));
  }
}
```

**设计要点**：
1. **单例模式**：每个类型的客户端在应用生命周期内只创建一次
2. **连接分离**：发布和订阅使用不同连接，避免阻塞
3. **健康检查**：订阅客户端跳过健康检查，因为 PING 会阻塞在订阅命令之后
4. **协作隔离**：协作服务支持独立的 Redis 实例，通过 `REDIS_COLLABORATION_URL` 环境变量配置

### 3.4 房间（Room）机制

Socket.IO 使用房间机制实现消息的定向广播。Outline 中定义了以下房间类型：

| 房间格式 | 用途 | 示例 |
|----------|------|------|
| `team-{teamId}` | 团队级广播 | `team-abc123` |
| `user-{userId}` | 用户级广播 | `user-xyz789` |
| `collection-{collectionId}` | 文集级广播 | `collection-def456` |
| `group-{groupId}` | 用户组级广播 | `group-ghi012` |

**房间加入逻辑**（`server/services/websockets.ts:170-227`）：

```typescript
async function authenticated(io: IO.Server, socket: SocketWithAuth) {
  const { user } = socket.client;
  
  // 基础房间：团队和用户
  const rooms = [`team-${user.teamId}`, `user-${user.id}`];

  // 动态获取用户有权限的文集和用户组
  const [collectionIds, groupIds] = await Promise.all([
    user.collectionIds(),
    user.groupIds(),
  ]);

  collectionIds.forEach((colId) => rooms.push(`collection-${colId}`));
  groupIds.forEach((groupId) => rooms.push(`group-${groupId}`));

  // 加入所有房间
  await socket.join(rooms);
}
```

### 3.5 消息路由与广播

消息广播通过 `WebsocketsProcessor` 处理，位于 `server/queues/processors/WebsocketsProcessor.ts`。

**消息路由策略**：

1. **事件触发**：系统事件（如 `documents.create`）通过事件队列触发
2. **频道计算**：根据事件类型和相关实体计算目标频道
3. **Socket.IO 广播**：使用 `socketio.to(channels).emit()` 发送消息

**示例：文档创建事件广播**

```typescript
case "documents.create":
case "documents.publish":
case "documents.restore": {
  const document = await Document.findByPk(event.documentId, { paranoid: false });
  if (!document) { return; }
  
  // 计算目标频道
  const channels = await this.getDocumentEventChannels(event, document);
  
  // 通过 Socket.IO 广播（Redis 适配器自动跨实例）
  return socketio.to(channels).emit("entities", {
    event: event.name,
    documentIds: [{ id: document.id, updatedAt: document.updatedAt }],
  });
}
```

**频道计算逻辑**：

```typescript
private async getDocumentEventChannels(
  event: Event,
  document: Document
): Promise<string[]> {
  const channels = [];

  // 1. 事件操作者
  if (event.actorId) {
    channels.push(`user-${event.actorId}`);
  }

  // 2. 已发布文档：文集或团队频道
  if (document.publishedAt) {
    if (document.collection) {
      channels.push(...this.getCollectionEventChannels(event, document.collection));
    } else {
      channels.push(`collection-${document.collectionId}`);
    }
  }

  // 3. 文档直接成员（用户权限）
  const [userMemberships, groupMemberships] = await Promise.all([
    UserMembership.findAll({ where: { documentId: document.id } }),
    GroupMembership.findAll({ where: { documentId: document.id } }),
  ]);

  for (const membership of userMemberships) {
    channels.push(`user-${membership.userId}`);
  }
  for (const membership of groupMemberships) {
    channels.push(`group-${membership.groupId}`);
  }

  return uniq(channels);
}
```

## 4. 协作编辑服务实现

### 4.1 概述

协作编辑服务使用 Hocuspocus 库实现基于 Y.js 的实时协作编辑，支持多用户同时编辑同一文档。

### 4.2 服务配置

协作服务配置在 `server/services/collaboration.ts` 中：

```typescript
const hocuspocus = Server.configure({
  debounce: 3000,
  timeout: 30000,
  maxDebounce: 10000,
  extensions: [
    // Redis 扩展：实现跨实例协作同步
    ...(env.REDIS_COLLABORATION_URL
      ? [
          new Redis({
            redis: RedisAdapter.collaborationClient,
          }),
        ]
      : []),
    new Throttle({
      throttle: env.RATE_LIMITER_COLLABORATION_REQUESTS,
      consideredSeconds: env.RATE_LIMITER_DURATION_WINDOW,
      banTime: 5,
    }),
    new ConnectionLimitExtension(),
    new AuthenticationExtension(),
    new PersistenceExtension(),
    new APIUpdateExtension(),
    // ... 其他扩展
  ],
});
```

### 4.3 Redis 同步机制

当配置了 `REDIS_COLLABORATION_URL` 时，Hocuspocus 使用 `@hocuspocus/extension-redis` 扩展实现跨实例同步：

**工作原理**：
1. 每个协作文档在 Redis 中存储 Y.js 状态
2. 文档更新通过 Redis pub/sub 广播到所有实例
3. 其他实例收到更新后，同步到本地的 Y.js 文档

**配置要点**：
- 支持独立的协作 Redis 实例，通过 `REDIS_COLLABORATION_URL` 环境变量配置
- 未配置时使用默认 Redis 实例

### 4.4 API 更新同步机制

当文档通过 REST API 更新时（非协作编辑方式），需要将更新同步到正在进行的协作会话。这通过 `APIUpdateExtension` 实现：

**核心实现**（`server/collaboration/APIUpdateExtension.ts`）：

```typescript
const CHANNEL_PREFIX = "collaboration:api-update";

export class APIUpdateExtension implements Extension {
  private documents: Map<string, HocuspocusDocument> = new Map();
  private subscriber: RedisAdapter | null = null;

  async onConfigure(_data: onConfigurePayload): Promise<void> {
    // 创建专用订阅客户端
    this.subscriber = new RedisAdapter(
      env.REDIS_COLLABORATION_URL || env.REDIS_URL,
      {
        connectionNameSuffix: "collab-api-updates",
        maxRetriesPerRequest: null,
      }
    );

    // 模式订阅：监听所有文档的 API 更新
    await this.subscriber.psubscribe(`${CHANNEL_PREFIX}:*`);

    // 处理收到的更新消息
    this.subscriber.on("pmessage", this.handleMessage);
  }

  private handleMessage = async (
    _pattern: string,
    channel: string,
    message: string
  ): Promise<void> => {
    // 解析文档 ID
    const documentId = channel.replace(`${CHANNEL_PREFIX}:`, "");
    const document = this.documents.get(documentId);

    // 如果文档未加载到当前实例，忽略
    if (!document) { return; }

    const data = JSON.parse(message);

    // 从数据库获取最新状态
    const dbDocument = await Document.unscoped().findOne({
      attributes: ["state", "content", "text"],
      where: { id: documentId },
    });

    // 计算增量更新
    const dbYdoc = new Y.Doc();
    Y.applyUpdate(dbYdoc, dbDocument.state);

    const currentStateVector = Y.encodeStateVector(document);
    const update = Y.encodeStateAsUpdate(dbYdoc, currentStateVector);

    // 应用更新到协作会话
    if (update.length > 0) {
      Y.applyUpdate(document, update);
    }
  };

  // 静态方法：发布更新通知（由文档更新命令调用）
  static async notifyUpdate(documentId: string, actorId: string): Promise<void> {
    const channel = `${CHANNEL_PREFIX}:${documentId}`;
    const message = JSON.stringify({ actorId, timestamp: Date.now() });

    await RedisAdapter.defaultClient.publish(channel, message);
  }
}
```

**工作流程**：

1. **API 更新触发**：当文档通过 REST API 更新时，调用 `APIUpdateExtension.notifyUpdate()`
2. **Redis 发布**：更新通知发布到 `collaboration:api-update:{documentId}` 频道
3. **所有实例订阅**：每个运行协作服务的实例都通过模式订阅（`psubscribe`）监听该频道
4. **本地同步**：收到通知的实例检查文档是否在本地加载，如果是则从数据库获取最新状态并同步到 Y.js 文档
5. **协作广播**：Y.js 更新自动通过 Hocuspocus Redis 扩展同步到其他实例的协作会话

## 5. 关键分析：协作服务未配置专用 Redis 时的限制

### 5.1 代码实现分析

**环境变量定义**（`server/env.ts:197-201`）：

```typescript
/**
 * The url of redis for horizontally scaling the collaboration service. If not
 * set then the collaboration service must be ran as a singleton.
 */
public REDIS_COLLABORATION_URL = environment.REDIS_COLLABORATION_URL;
```

**进程数量限制逻辑**（`server/index.ts:34-47`）：

```typescript
// The number of processes to run, defaults to the number of CPU's available
// for the web service, and 1 for collaboration unless REDIS_COLLABORATION_URL is set.
let webProcessCount = env.WEB_CONCURRENCY;

if (env.SERVICES.includes("collaboration") && !env.REDIS_COLLABORATION_URL) {
  if (webProcessCount !== 1) {
    Logger.info(
      "lifecycle",
      "Note: Restricting process count to 1 due to use of collaborative service without REDIS_COLLABORATION_URL"
    );
  }

  webProcessCount = 1;
}
```

**Hocuspocus Redis 扩展条件加载**（`server/services/collaboration.ts:48-55`）：

```typescript
extensions: [
  // Redis 扩展：实现跨实例协作同步
  ...(env.REDIS_COLLABORATION_URL
    ? [
        new Redis({
          redis: RedisAdapter.collaborationClient,
        }),
      ]
    : []),
  // ...
]
```

### 5.2 多实例部署前提条件

| 条件 | 要求 | 代码位置 |
|------|------|----------|
| 协作服务水平扩展 | 必须配置 `REDIS_COLLABORATION_URL` | `server/env.ts:198-199` |
| 多进程/多实例运行 | 必须配置 `REDIS_COLLABORATION_URL` | `server/index.ts:38-46` |
| 跨实例协作同步 | 必须配置 `REDIS_COLLABORATION_URL` | `server/services/collaboration.ts:49-55` |

### 5.3 进程限制机制

当未配置 `REDIS_COLLABORATION_URL` 且启用了协作服务时：

1. **进程数强制设置为 1**（`server/index.ts:46`）
   - 无论 `WEB_CONCURRENCY` 环境变量配置为何值
   - 系统会输出日志提示：`"Note: Restricting process count to 1 due to use of collaborative service without REDIS_COLLABORATION_URL"`

2. **Hocuspocus Redis 扩展不加载**（`server/services/collaboration.ts:49-55`）
   - 扩展数组中不包含 `@hocuspocus/extension-redis`
   - Y.js 文档状态仅存储在各实例内存中

3. **协作状态隔离**
   - 每个实例的协作文档状态独立
   - 连接到不同实例的用户无法看到彼此的实时编辑

### 5.4 未配置专用 Redis 时的部署限制

**允许的部署方式**：
- ✅ 单实例部署（1 个进程）
- ✅ 单实例多进程（但进程数被强制限制为 1）

**不允许/不推荐的部署方式**：
- ❌ 多实例部署（负载均衡 + 多个 Node.js 实例）
  - 原因：协作状态无法跨实例同步
  - 后果：连接到不同实例的用户无法实时协作

**架构对比**：

```
┌─────────────────────────────────────────────────────────────┐
│  场景 A：未配置 REDIS_COLLABORATION_URL（仅支持单实例）       │
└─────────────────────────────────────────────────────────────┘

                    ┌─────────────────┐
                    │   Load Balancer │
                    │   (无法使用)     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Single Node   │
                    │  (webProcess    │
                    │    Count = 1)   │
                    │                 │
                    │  协作状态存储在  │
                    │    内存中        │
                    └─────────────────┘


┌─────────────────────────────────────────────────────────────┐
│  场景 B：配置了 REDIS_COLLABORATION_URL（支持多实例）          │
└─────────────────────────────────────────────────────────────┘

                    ┌─────────────────┐
                    │   Load Balancer │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
      ┌─────▼─────┐    ┌─────▼─────┐    ┌─────▼─────┐
      │  Instance A│    │  Instance B│    │  Instance C│
      │            │    │            │    │            │
      │ 协作状态通过 │    │ 协作状态通过 │    │ 协作状态通过 │
      │ Redis 同步  │    │ Redis 同步  │    │ Redis 同步  │
      └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
            │                │                │
            └────────────────┼────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Redis (协作专用) │
                    │ REDIS_COLLAB    │
                    └─────────────────┘
```

## 6. 关键分析：HTTP 更新后发布订阅链路及断链条件

### 6.1 链路组件分析

**发布端**（`server/collaboration/APIUpdateExtension.ts:193-209`）：

```typescript
static async notifyUpdate(
  documentId: string,
  actorId: string
): Promise<void> {
  const channel = `${CHANNEL_PREFIX}:${documentId}`;
  const message = JSON.stringify({
    actorId,
    timestamp: Date.now(),
  });

  // 关键：总是使用 defaultClient (REDIS_URL)
  await RedisAdapter.defaultClient.publish(channel, message);
}
```

**订阅端**（`server/collaboration/APIUpdateExtension.ts:53-61`）：

```typescript
this.subscriber = new RedisAdapter(
  // 关键：优先使用 REDIS_COLLABORATION_URL，没有则使用 REDIS_URL
  env.REDIS_COLLABORATION_URL || env.REDIS_URL,
  {
    connectionNameSuffix: "collab-api-updates",
    maxRetriesPerRequest: null,
  }
);
```

**触发条件**（`server/models/Document.ts:585-599`）：

```typescript
@AfterUpdate
static notifyCollaborationServer(model: Document, ctx: HookContext) {
  // 当 state 字段变化且有认证用户时触发
  if (model.changed("state") && ctx.auth?.user?.id) {
    const actorId = ctx.auth.user.id;
    const notify = async () => {
      await APIUpdateExtension.notifyUpdate(model.id, actorId);
    };

    // 事务提交后执行
    if (ctx.transaction) {
      const transaction = ctx.transaction.parent || ctx.transaction;
      transaction.afterCommit(notify);
    } else {
      void notify();
    }
  }
}
```

### 6.2 发布订阅链路不对称性

| 组件 | Redis 选择逻辑 | 使用的环境变量 |
|------|----------------|----------------|
| **发布端** (`notifyUpdate`) | 硬编码使用 `defaultClient` | `REDIS_URL` |
| **订阅端** (`onConfigure`) | 优先使用协作 Redis | `REDIS_COLLABORATION_URL \|\| REDIS_URL` |
| **协作同步** (Hocuspocus Redis) | 仅当配置协作 Redis 时启用 | `REDIS_COLLABORATION_URL` |

### 6.3 场景分析

#### 场景 1：仅配置 `REDIS_URL`（无 `REDIS_COLLABORATION_URL`）

**配置**：
- `REDIS_URL=redis://localhost:6379`
- `REDIS_COLLABORATION_URL=`（未设置）

**链路**：
```
HTTP 更新触发
    │
    ▼
notifyUpdate() 使用 defaultClient (REDIS_URL)
    │
    ▼
发布到 Redis (REDIS_URL) 的 collaboration:api-update:{docId} 频道
    │
    ▼
订阅端使用 REDIS_COLLABORATION_URL || REDIS_URL → REDIS_URL
    │
    ▼
同一 Redis 实例，消息正常接收
```

**状态**：✅ **链路正常**

**限制**：此时协作服务被强制限制为单进程运行（见第 5 节）

---

#### 场景 2：配置 `REDIS_URL` 和 `REDIS_COLLABORATION_URL`（指向同一 Redis）

**配置**：
- `REDIS_URL=redis://localhost:6379`
- `REDIS_COLLABORATION_URL=redis://localhost:6379`（同一实例）

**链路**：
```
HTTP 更新触发
    │
    ▼
notifyUpdate() 使用 defaultClient → REDIS_URL (同一实例)
    │
    ▼
发布到 Redis 的 collaboration:api-update:{docId} 频道
    │
    ▼
订阅端使用 REDIS_COLLABORATION_URL → 同一实例
    │
    ▼
同一 Redis 实例，消息正常接收
```

**状态**：✅ **链路正常**

---

#### 场景 3：配置 `REDIS_URL` 和 `REDIS_COLLABORATION_URL`（指向不同 Redis）

**配置**：
- `REDIS_URL=redis://redis-a:6379`（Redis A）
- `REDIS_COLLABORATION_URL=redis://redis-b:6379`（Redis B，不同实例）

**链路**：
```
HTTP 更新触发
    │
    ▼
notifyUpdate() 使用 defaultClient → REDIS_URL (Redis A)
    │
    ▼
发布到 Redis A 的 collaboration:api-update:{docId} 频道
    │
    │  ⚠️ 断链点！发布和订阅在不同的 Redis 实例
    │
    ▼
订阅端使用 REDIS_COLLABORATION_URL → Redis B
    │
    ▼
Redis B 上没有收到消息（消息在 Redis A）
    │
    ▼
协作会话无法收到 API 更新通知
```

**状态**：❌ **断链！**

**影响**：
- 通过 HTTP 接口更新的文档不会同步到正在进行的协作会话
- 协作编辑的用户看不到 API 更新的内容
- 持久化时可能产生冲突

### 6.4 断链条件总结

**断链发生的必要条件**（必须同时满足）：

| 条件 | 说明 | 代码位置 |
|------|------|----------|
| 1. 配置了 `REDIS_COLLABORATION_URL` | 协作服务启用了专用 Redis | `server/services/collaboration.ts:49` |
| 2. `REDIS_URL ≠ REDIS_COLLABORATION_URL` | 两个环境变量指向不同的 Redis 实例 | 部署配置 |
| 3. 文档通过 HTTP 接口更新 | 触发 `state` 字段变化 | `server/models/Document.ts:587` |

**断链发生位置**：

```
┌─────────────────────────────────────────────────────────────────┐
│                        断链示意图                                 │
└─────────────────────────────────────────────────────────────────┘

  HTTP API 更新
       │
       ▼
┌──────────────┐
│  Instance X  │
│  (处理请求)   │
└──────┬───────┘
       │
       ▼ notifyUpdate()
       │ 使用 RedisAdapter.defaultClient
       │
       ▼
┌──────────────────┐
│   Redis A        │  ← REDIS_URL
│                  │
│  发布消息到频道：  │
│  collaboration:  │
│  api-update:doc1 │
└──────────────────┘
       │
       │ ⚠️ 消息在这里 "消失" 了
       │ 因为订阅在另一个 Redis
       ▼
┌──────────────────┐
│   Redis B        │  ← REDIS_COLLABORATION_URL
│                  │
│  订阅频道：        │
│  collaboration:  │
│  api-update:*    │
│                  │
│  ❌ 没有收到消息  │
└──────────────────┘
       │
       ▼
┌──────────────┐    ┌──────────────┐
│  Instance A  │    │  Instance B  │
│  (协作服务)   │    │  (协作服务)   │
│              │    │              │
│  ❌ 无法同步  │    │  ❌ 无法同步  │
└──────────────┘    └──────────────┘
```

### 6.5 完整消息路由流程图（含断链场景）

```
┌──────────────────────────────────────────────────────────────────────┐
│  完整链路：HTTP 更新 → 协作会话同步（配置独立协作 Redis 时）            │
└──────────────────────────────────────────────────────────────────────┘

  REST API (PUT /documents/:id)
            │
            │ 更新文档 state 字段
            ▼
┌─────────────────────────────────────┐
│  Document.notifyCollaborationServer │  ← server/models/Document.ts:585
│  @AfterUpdate Hook                  │
│  条件：changed("state") && auth.user │
└───────────────────┬─────────────────┘
                    │
                    ▼ transaction.afterCommit()
                    │
┌─────────────────────────────────────┐
│  APIUpdateExtension.notifyUpdate()  │  ← server/collaboration/APIUpdateExtension.ts:193
│                                     │
│  使用 RedisAdapter.defaultClient    │  ⚠️ 硬编码使用 REDIS_URL
│  .publish(channel, message)         │
└───────────────────┬─────────────────┘
                    │
                    ▼
           ┌────────────────┐
           │   Redis A      │  ← REDIS_URL
           │                │
           │  频道：         │
           │  collaboration │
           │  :api-update   │
           │  :{documentId} │
           └────────────────┘
                    │
                    │ ⚠️ 断链风险点
                    │
    ┌───────────────┴───────────────┐
    │                               │
    ▼ 配置相同 Redis                 ▼ 配置不同 Redis
┌──────────────┐              ┌──────────────┐
│ Redis A ==   │              │ Redis A !=   │
│ Redis B      │              │ Redis B      │
│ (同一实例)    │              │ (不同实例)    │
└──────┬───────┘              └──────┬───────┘
       │                             │
       ▼ 正常                        ▼ 断链
┌─────────────────────────┐    ┌─────────────────────────┐
│  订阅端使用              │    │  订阅端使用              │
│  REDIS_COLLABORATION_URL│    │  REDIS_COLLABORATION_URL│
│  = Redis A               │    │  = Redis B               │
│                         │    │                         │
│  ✅ 收到消息             │    │  ❌ 收不到消息           │
│  (同一实例)              │    │  (不同实例)              │
└──────────┬──────────────┘    └─────────────────────────┘
           │
           ▼
┌─────────────────────────┐
│  APIUpdateExtension     │
│  .handleMessage()       │
│                         │
│  从 DB 读取最新 state    │
│  计算 Y.js 增量更新      │
│  applyUpdate 到协作文档  │
└──────────┬──────────────┘
           │
           ▼ (如果配置了 Hocuspocus Redis 扩展)
┌─────────────────────────┐
│  @hocuspocus/           │
│  extension-redis        │
│                         │
│  通过 Redis 同步到其他   │
│  实例的协作会话          │
└─────────────────────────┘
```

## 7. 消息路由流程图

### 7.1 Socket.IO 事件广播流程

```
┌──────────────┐
│   Instance A │
│  (文档更新)   │
└──────┬───────┘
       │
       ▼ 1. 触发事件
┌──────────────┐
│  Event Queue │
│   (Bull)     │
└──────┬───────┘
       │
       ▼ 2. 处理事件
┌─────────────────────┐
│ WebsocketsProcessor │
│  (计算目标频道)      │
└──────────┬──────────┘
           │
           ▼ 3. socketio.to().emit()
┌─────────────────────┐
│  Socket.IO Server   │
│  (Redis Adapter)    │
└──────────┬──────────┘
           │
    ┌──────┴──────┐
    │             │
    ▼ 4a. 本地    ▼ 4b. Redis Pub
┌──────────┐   ┌──────────┐
│Instance A│   │  Redis   │
│ (直接发送)│   │          │
└──────────┘   └────┬─────┘
                     │
           ┌─────────┼─────────┐
           │         │         │
           ▼ 5a. Sub ▼ 5b. Sub ▼ 5c. Sub
      ┌──────────┐┌──────────┐┌──────────┐
      │Instance A││Instance B││Instance C│
      │ (已处理)  ││  (接收)   ││  (接收)   │
      └──────────┘└────┬─────┘└────┬─────┘
                        │            │
                        ▼ 6. 发送    ▼ 6. 发送
                   ┌──────────┐  ┌──────────┐
                   │  Client  │  │  Client  │
                   │ (B连接)  │  │ (C连接)  │
                   └──────────┘  └──────────┘
```

### 7.2 协作编辑同步流程

```
┌─────────────────────────────────────────────────────────────┐
│                      场景1: 协作编辑同步                       │
└─────────────────────────────────────────────────────────────┘

Client A ──► Instance A ──► Redis (Y.js状态 + Pub/Sub) ──► Instance B ──► Client B
   ▲              ▲                                               ▲              ▲
   │              │                                               │              │
   └──────────────┴───────────────────────────────────────────────┴──────────────┘
                        Hocuspocus Redis Extension 自动同步


┌─────────────────────────────────────────────────────────────┐
│                      场景2: API 更新同步                       │
└─────────────────────────────────────────────────────────────┘

REST API ──► Instance A ──► DB 更新
                │
                ▼
           notifyUpdate()
                │
                ▼
           Redis Pub: collaboration:api-update:doc123
                │
     ┌──────────┼──────────┐
     ▼          ▼          ▼
Instance A  Instance B  Instance C
(已加载文档) (未加载)    (已加载文档)
     │                     │
     ▼                     ▼
  从DB获取最新状态      从DB获取最新状态
     │                     │
     ▼                     ▼
  Y.applyUpdate()      Y.applyUpdate()
     │                     │
     ▼                     ▼
  广播到本地客户端      广播到本地客户端
```

## 8. 关键技术点分析

### 8.1 Socket.IO Redis 适配器工作原理

`socket.io-redis` 适配器的工作机制：

1. **房间状态同步**：所有实例的房间 membership 通过 Redis 同步
2. **消息广播**：当调用 `io.to(room).emit()` 时：
   - 适配器将消息发布到 Redis 频道 `socket.io#/#{room}#`
   - 所有订阅该频道的实例收到消息
   - 每个实例检查本地连接的客户端是否在该房间，然后发送消息

3. **自定义事件**：支持跨实例的自定义事件通信

### 8.2 连接升级处理

Outline 支持两种 WebSocket 服务在同一端口运行，通过路径区分：

```typescript
server.on(
  "upgrade",
  function (req: IncomingMessage, socket: Duplex, head: Buffer) {
    // 1. 实时事件服务：/realtime
    if (req.url?.startsWith("/realtime") && ioHandleUpgrade) {
      ioHandleUpgrade(req, socket, head);
      return;
    }

    // 2. 协作编辑服务：/collaboration
    if (req.url?.startsWith("/collaboration")) {
      wss.handleUpgrade(req, socket, head, (client) => {
        hocuspocus.handleConnection(client, req, documentId);
      });
      return;
    }

    // 未知路径，关闭连接
    socket.end(`HTTP/1.1 400 Bad Request\r\n`);
  }
);
```

### 8.3 认证与授权

WebSocket 连接的认证流程：

1. **连接建立**：客户端通过 Cookie 中的 `accessToken` 进行认证
2. **JWT 验证**：服务器验证 JWT 并获取用户信息
3. **超时断开**：1秒内未认证的连接会被断开
4. **房间授权**：根据用户权限加入相应的房间

```typescript
async function authenticate(socket: SocketWithAuth) {
  const cookies = socket.request.headers.cookie
    ? cookie.parse(socket.request.headers.cookie)
    : {};
  const { accessToken } = cookies;

  if (!accessToken) {
    throw AuthenticationError("No access token");
  }

  const { user } = await getUserForJWT(accessToken);
  socket.client.user = user;
  return user;
}
```

## 9. 部署与配置建议

### 9.1 环境变量配置

| 环境变量 | 用途 | 说明 |
|----------|------|------|
| `REDIS_URL` | 默认 Redis 连接 | 用于 Socket.IO 适配器、队列、缓存、API 更新发布 |
| `REDIS_COLLABORATION_URL` | 协作服务专用 Redis | 可选，配置后用于协作同步和 API 更新订阅 |
| `WEB_CONCURRENCY` | 进程数量 | 未配置协作 Redis 时被强制设置为 1 |
| `SERVICES` | 启用的服务列表 | 包含 `collaboration` 时启用协作服务 |

### 9.2 配置最佳实践

**推荐配置方式**：

1. **方式 A：单 Redis 实例（简单部署）**
   ```bash
   REDIS_URL=redis://redis:6379
   # REDIS_COLLABORATION_URL 不设置
   ```
   - 协作服务被限制为单进程
   - API 更新链路正常
   - 适合小型部署

2. **方式 B：单 Redis 实例 + 多进程协作**
   ```bash
   REDIS_URL=redis://redis:6379
   REDIS_COLLABORATION_URL=redis://redis:6379  # 同一 URL
   ```
   - 协作服务支持多进程/多实例
   - API 更新链路正常（同一 Redis）
   - ✅ **推荐配置**

3. **方式 C：双 Redis 实例（隔离部署）⚠️ 需注意**
   ```bash
   REDIS_URL=redis://redis-default:6379
   REDIS_COLLABORATION_URL=redis://redis-collab:6379  # 不同实例
   ```
   - ⚠️ **API 更新链路断链**（发布在 default，订阅在 collab）
   - 需要修复代码中的不对称问题

### 9.3 多实例部署最佳实践

1. **负载均衡配置**：
   - 启用 WebSocket 支持（Upgrade 头处理）
   - 使用 Sticky Sessions（虽然 Socket.IO 适配器不强制要求，但可以减少跨实例消息转发）

2. **Redis 配置**：
   - 使用 Redis Cluster 或 Sentinel 确保高可用
   - **强烈建议**：`REDIS_URL` 和 `REDIS_COLLABORATION_URL` 配置为同一 Redis 实例，或修复代码中的不对称性
   - 配置合理的连接池大小

3. **协作服务扩展前提**：
   - 必须配置 `REDIS_COLLABORATION_URL`
   - 确保该 Redis 实例可被所有协作服务实例访问
   - 如果配置了独立协作 Redis，需注意 API 更新链路问题

4. **监控与日志**：
   - 监控 WebSocket 连接数（`websockets.count` 指标）
   - 监控 Redis pub/sub 消息延迟
   - 关注 `socket.io#` 前缀的 Redis 频道消息量
   - 监控 `collaboration:api-update:*` 频道的消息发布/订阅情况

### 9.4 故障恢复

1. **Redis 连接故障**：
   - 适配器会自动重试连接（可配置 `maxRetriesPerRequest`）
   - 连接恢复后，房间状态会重新同步

2. **实例故障**：
   - 连接到故障实例的客户端会自动重连到其他实例
   - Socket.IO 适配器确保消息能够到达存活实例上的客户端

3. **API 更新断链恢复**：
   - 如果配置了双 Redis 且发现断链：
     - 临时方案：将 `REDIS_COLLABORATION_URL` 改为与 `REDIS_URL` 相同
     - 长期方案：修复代码中的发布/订阅不对称性

## 10. 总结

### 10.1 核心架构

Outline 的多实例 WebSocket 集群实现采用了成熟的技术方案：

1. **Socket.IO + socket.io-redis**：实现系统事件的跨实例广播
2. **Hocuspocus + @hocuspocus/extension-redis**：实现协作编辑的实时同步
3. **自定义 Redis pub/sub**：实现 API 更新到协作会话的同步
4. **多 Redis 客户端分离**：发布、订阅、协作使用独立连接，避免阻塞

### 10.2 关键发现

**发现 1：协作服务多实例部署的强制条件**
- 未配置 `REDIS_COLLABORATION_URL` 时，协作服务被强制限制为单进程运行
- 原因：Hocuspocus Redis 扩展只在配置了协作 Redis 时才加载
- 影响：多实例部署时协作状态无法同步

**发现 2：API 更新发布订阅链路的不对称性**
- 发布端（`notifyUpdate`）硬编码使用 `REDIS_URL`
- 订阅端优先使用 `REDIS_COLLABORATION_URL`
- 当配置两个不同的 Redis 实例时，链路断链
- 影响：HTTP 更新无法同步到协作会话

### 10.3 部署建议

| 部署场景 | 推荐配置 | 注意事项 |
|----------|----------|----------|
| 单实例小型部署 | 不配置 `REDIS_COLLABORATION_URL` | 协作服务单进程 |
| 多实例标准部署 | `REDIS_URL` = `REDIS_COLLABORATION_URL` | ✅ 推荐，链路正常 |
| 双 Redis 隔离部署 | 需修复代码不对称性 | ⚠️ 当前实现存在断链风险 |

## 11. 参考代码位置

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| WebSocket 服务初始化 | `server/services/websockets.ts` | 1-168 |
| Redis 适配器配置 | `server/services/websockets.ts` | 95-100 |
| Redis 客户端管理 | `server/storage/redis.ts` | 1-163 |
| 协作服务配置 | `server/services/collaboration.ts` | 1-143 |
| API 更新同步扩展 | `server/collaboration/APIUpdateExtension.ts` | 1-210 |
| API 更新发布（硬编码 defaultClient） | `server/collaboration/APIUpdateExtension.ts` | 193-209 |
| API 更新订阅（优先协作 Redis） | `server/collaboration/APIUpdateExtension.ts` | 53-61 |
| 进程数量限制逻辑 | `server/index.ts` | 34-47 |
| 协作 Redis 扩展条件加载 | `server/services/collaboration.ts` | 48-55 |
| 文档更新触发通知 | `server/models/Document.ts` | 585-599 |
| WebSocket 事件处理器 | `server/queues/processors/WebsocketsProcessor.ts` | 1-1031 |
| 房间管理逻辑 | `server/services/websockets.ts` | 170-227 |
| 环境变量定义（协作 Redis 注释） | `server/env.ts` | 197-201 |
