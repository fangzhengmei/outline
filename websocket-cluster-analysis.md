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

## 5. 消息路由流程图

### 5.1 Socket.IO 事件广播流程

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

### 5.2 协作编辑同步流程

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

## 6. 关键技术点分析

### 6.1 Socket.IO Redis 适配器工作原理

`socket.io-redis` 适配器的工作机制：

1. **房间状态同步**：所有实例的房间 membership 通过 Redis 同步
2. **消息广播**：当调用 `io.to(room).emit()` 时：
   - 适配器将消息发布到 Redis 频道 `socket.io#/#{room}#`
   - 所有订阅该频道的实例收到消息
   - 每个实例检查本地连接的客户端是否在该房间，然后发送消息

3. **自定义事件**：支持跨实例的自定义事件通信

### 6.2 连接升级处理

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

### 6.3 认证与授权

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

## 7. 部署与配置建议

### 7.1 环境变量配置

| 环境变量 | 用途 | 说明 |
|----------|------|------|
| `REDIS_URL` | 默认 Redis 连接 | 用于 Socket.IO 适配器、队列、缓存等 |
| `REDIS_COLLABORATION_URL` | 协作服务专用 Redis | 可选，配置后协作服务使用独立 Redis 实例 |
| `URL` | 服务 URL | 用于 CORS 和 Origin 验证 |

### 7.2 多实例部署最佳实践

1. **负载均衡配置**：
   - 启用 WebSocket 支持（Upgrade 头处理）
   - 使用 Sticky Sessions（虽然 Socket.IO 适配器不强制要求，但可以减少跨实例消息转发）

2. **Redis 配置**：
   - 使用 Redis Cluster 或 Sentinel 确保高可用
   - 考虑为协作服务使用独立的 Redis 实例（通过 `REDIS_COLLABORATION_URL`）
   - 配置合理的连接池大小

3. **监控与日志**：
   - 监控 WebSocket 连接数（`websockets.count` 指标）
   - 监控 Redis pub/sub 消息延迟
   - 关注 `socket.io#` 前缀的 Redis 频道消息量

### 7.3 故障恢复

1. **Redis 连接故障**：
   - 适配器会自动重试连接（可配置 `maxRetriesPerRequest`）
   - 连接恢复后，房间状态会重新同步

2. **实例故障**：
   - 连接到故障实例的客户端会自动重连到其他实例
   - Socket.IO 适配器确保消息能够到达存活实例上的客户端

## 8. 总结

Outline 的多实例 WebSocket 集群实现采用了成熟的技术方案：

1. **Socket.IO + socket.io-redis**：实现系统事件的跨实例广播
2. **Hocuspocus + @hocuspocus/extension-redis**：实现协作编辑的实时同步
3. **自定义 Redis pub/sub**：实现 API 更新到协作会话的同步
4. **多 Redis 客户端分离**：发布、订阅、协作使用独立连接，避免阻塞

这种架构确保了 Outline 在多实例部署场景下的实时性、可靠性和可扩展性，是一个设计良好的分布式实时通信系统实现。

## 9. 参考代码位置

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| WebSocket 服务初始化 | `server/services/websockets.ts` | 1-168 |
| Redis 适配器配置 | `server/services/websockets.ts` | 95-100 |
| Redis 客户端管理 | `server/storage/redis.ts` | 1-163 |
| 协作服务配置 | `server/services/collaboration.ts` | 1-143 |
| API 更新同步 | `server/collaboration/APIUpdateExtension.ts` | 1-210 |
| WebSocket 事件处理器 | `server/queues/processors/WebsocketsProcessor.ts` | 1-1031 |
| 房间管理逻辑 | `server/services/websockets.ts` | 170-227 |
