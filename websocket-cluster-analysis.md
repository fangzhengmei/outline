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

---

## 3. 严格路由条件矩阵：API 更新能否同步到协作会话

### 3.1 完整条件链路图

API 更新同步到协作会话是一个多阶段的过程，**每个阶段的所有条件必须同时满足**，否则链路会在该阶段中断。

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    API 更新同步到协作会话的完整条件链路（严格版）                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

  HTTP API 更新文档（如 PUT /documents/:id）
         │
         │ 触发 Document 模型的 @AfterUpdate Hook
         ▼
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ 【阶段 1：发布端条件】 —— 决定是否调用 notifyUpdate() 并发布消息到 Redis              │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼ C1.1 检查
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ C1.1: 文档 state 字段是否发生变化？                                                  │
  │                                                                                    │
  │ 代码证据: server/models/Document.ts:587                                            │
  │   if (model.changed("state") && ctx.auth?.user?.id) {                              │
  │       ...                                                                            │
  │   }                                                                                  │
  │                                                                                    │
  │ 真实生效逻辑:                                                                        │
  │   - model.changed("state") 必须返回 true                                            │
  │   - 这意味着本次更新必须实际修改了 state 字段的值                                     │
  │   - 如果只修改 title 等其他字段，此条件不满足                                        │
  │                                                                                    │
  │ 测试证据: server/commands/documentUpdater.test.ts:97-116                           │
  │   it("should notify collaboration server when text changes", ...) → ✅ 调用        │
  │   it("should not notify collaboration server when only title changes", ...) → ❌  │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ├─────────────── 否 ───────────────► ❌ 不触发 notifyUpdate()，链路终止
         │
         │ 是
         ▼ C1.2 检查
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ C1.2: 请求上下文中是否有认证用户？                                                   │
  │                                                                                    │
  │ 代码证据: server/models/Document.ts:587                                            │
  │   if (model.changed("state") && ctx.auth?.user?.id) {                              │
  │       ...                                                                            │
  │   }                                                                                  │
  │                                                                                    │
  │ 真实生效逻辑:                                                                        │
  │   - ctx.auth 必须存在                                                                │
  │   - ctx.auth.user 必须存在                                                          │
  │   - ctx.auth.user.id 必须存在                                                       │
  │   - 这意味着更新必须来自已认证的 API 请求                                             │
  │   - 如果是后台任务或系统级更新，此条件可能不满足                                      │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ├─────────────── 否 ───────────────► ❌ 不触发 notifyUpdate()，链路终止
         │
         │ 是
         ▼ C1.3 检查
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ C1.3: 数据库事务是否已提交（如有事务）？                                             │
  │                                                                                    │
  │ 代码证据: server/models/Document.ts:593-598                                        │
  │   if (ctx.transaction) {                                                            │
  │       const transaction = ctx.transaction.parent || ctx.transaction;                │
  │       transaction.afterCommit(notify);  // 事务提交后才执行                         │
  │   } else {                                                                           │
  │       void notify();  // 无事务时立即执行                                            │
  │   }                                                                                  │
  │                                                                                    │
  │ 真实生效逻辑:                                                                        │
  │   - 如果更新在数据库事务中执行：                                                      │
  │     → 必须等待事务提交，notifyUpdate() 才会被调用                                    │
  │     → 如果事务回滚，notifyUpdate() 不会被调用                                        │
  │   - 如果更新不在事务中执行：                                                          │
  │     → notifyUpdate() 立即执行（void notify() 无 await）                              │
  │                                                                                    │
  │ 注意: void notify() 意味着即使 notify() 内部出错，也不会影响主流程                   │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ├─────────────── 事务回滚 ────────► ❌ 不执行 notifyUpdate()，链路终止
         │
         │ 事务已提交（或无事务）
         ▼ C1.4 检查
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ C1.4: 发布端 Redis 连接是否正常？                                                    │
  │                                                                                    │
  │ 代码证据: server/collaboration/APIUpdateExtension.ts:193-209                      │
  │   static async notifyUpdate(documentId: string, actorId: string): Promise<void> { │
  │       const channel = `${CHANNEL_PREFIX}:${documentId}`;                           │
  │       const message = JSON.stringify({ actorId, timestamp: Date.now() });          │
  │       // ⚠️ 关键：硬编码使用 defaultClient                                          │
  │       await RedisAdapter.defaultClient.publish(channel, message);                   │
  │   }                                                                                  │
  │                                                                                    │
  │ RedisAdapter.defaultClient 定义: server/storage/redis.ts:129-136                   │
  │   public static get defaultClient(): RedisAdapter {                                 │
  │       return (                                                                       │
  │           this.client ||                                                             │
  │           (this.client = new this(env.REDIS_URL, {  // ⚠️ 只看 REDIS_URL          │
  │               connectionNameSuffix: "client",                                        │
  │           }))                                                                         │
  │       );                                                                              │
  │   }                                                                                  │
  │                                                                                    │
  │ 真实生效逻辑:                                                                        │
  │   - 必须使用 env.REDIS_URL 指定的 Redis 实例                                         │
  │   - RedisAdapter.defaultClient.publish() 必须成功执行                                │
  │   - 如果 Redis 连接失败或超时，消息不会被发布                                        │
  │                                                                                    │
  │ ⚠️ 关键点：发布端**硬编码**使用 REDIS_URL，不考虑 REDIS_COLLABORATION_URL           │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ├─────────────── Redis 错误 ───────► ❌ 消息未发布，链路终止
         │
         │ 正常
         ▼
  ✅ 消息已成功发布到 Redis (REDIS_URL 对应的实例)
  频道格式: collaboration:api-update:{documentId}
         │
         ▼
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ 【阶段 2：订阅端条件】 —— 决定订阅端是否能收到消息                                   │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼ C2.1 检查
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ C2.1: 协作服务是否已启用？                                                          │
  │                                                                                    │
  │ 代码证据 1: server/index.ts:181-188                                                │
  │   // loop through requested services at startup                                     │
  │   for (const name of env.SERVICES) {                                               │
  │       if (!Object.keys(services).includes(name)) {                                 │
  │           throw new Error(`Unknown service ${name}`);                               │
  │       }                                                                              │
  │       Logger.info("lifecycle", `Starting ${name} service`);                        │
  │       const init = services[name as keyof typeof services];                        │
  │       await init(app, server as https.Server, env.SERVICES);  // 才会调用         │
  │   }                                                                                  │
  │                                                                                    │
  │ 代码证据 2: server/services/collaboration.ts:41-75                                 │
  │   const hocuspocus = Server.configure({                                             │
  │       extensions: [                                                                 │
  │           // ...                                                                    │
  │           new APIUpdateExtension(),  // 只有协作服务启动时才会加载此扩展              │
  │           // ...                                                                    │
  │       ],                                                                             │
  │   });                                                                                │
  │                                                                                    │
  │ 真实生效逻辑:                                                                        │
  │   - env.SERVICES 数组必须包含 "collaboration" 字符串                                │
  │   - 只有这样，services.collaboration.init() 才会被调用                              │
  │   - 只有这样，Hocuspocus 服务器才会被创建                                            │
  │   - 只有这样，APIUpdateExtension 才会被加入 extensions 数组                          │
  │   - 只有这样，APIUpdateExtension.onConfigure() 才会被调用                            │
  │   - 只有这样，Redis 订阅才会建立                                                     │
  │                                                                                    │
  │ 如果协作服务未启用：                                                                 │
  │   - APIUpdateExtension 不会被加载                                                   │
  │   - 没有任何订阅存在                                                                 │
  │   - 即使消息发布到 Redis，也没有订阅者接收                                           │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ├─────────────── 未启用 ─────────────► ❌ 无订阅者，链路终止
         │
         │ 已启用
         ▼ C2.2 检查
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ C2.2: 订阅端 Redis 连接是否正常且订阅成功？                                          │
  │                                                                                    │
  │ 代码证据: server/collaboration/APIUpdateExtension.ts:47-90                        │
  │   async onConfigure(_data: onConfigurePayload): Promise<void> {                   │
  │       if (this.configured) { return; }                                             │
  │       this.configured = true;                                                       │
  │                                                                                    │
  │       try {                                                                         │
  │           // 创建专用订阅客户端                                                      │
  │           this.subscriber = new RedisAdapter(                                      │
  │               // ⚠️ 关键：优先使用协作 Redis                                         │
  │               env.REDIS_COLLABORATION_URL || env.REDIS_URL,                        │
  │               {                                                                     │
  │                   connectionNameSuffix: "collab-api-updates",                       │
  │                   maxRetriesPerRequest: null,                                        │
  │               }                                                                     │
  │           );                                                                        │
  │                                                                                    │
  │           // 模式订阅所有 API 更新频道                                                │
  │           await this.subscriber.psubscribe(`${CHANNEL_PREFIX}:*`, (err) => {     │
  │               if (err) {                                                            │
  │                   Logger.error("Failed to subscribe to API update channel", err);   │
  │                   return;                                                            │
  │               }                                                                      │
  │               Logger.debug(                                                          │
  │                   "multiplayer",                                                     │
  │                   `Subscribed to ${CHANNEL_PREFIX}:* for API updates`              │
  │               );                                                                     │
  │           });                                                                        │
  │                                                                                    │
  │           // 注册消息处理回调                                                        │
  │           this.subscriber.on("pmessage", this.handleMessage);                       │
  │       } catch (error) {                                                              │
  │           // ⚠️ 关键：失败时的处理                                                   │
  │           Logger.error(                                                              │
  │               "Failed to configure APIUpdateExtension Redis subscriber",            │
  │               error as Error                                                         │
  │           );                                                                          │
  │           this.subscriber = null;                                                    │
  │           this.configured = false;                                                   │
  │       }                                                                              │
  │   }                                                                                  │
  │                                                                                    │
  │ 真实生效逻辑:                                                                        │
  │   1. onConfigure() 被调用（协作服务已启用）                                          │
  │   2. 创建 Redis 订阅客户端：                                                          │
  │      - 如果 env.REDIS_COLLABORATION_URL 已配置 → 使用它                             │
  │      - 否则 → 使用 env.REDIS_URL                                                    │
  │   3. 执行 psubscribe("collaboration:api-update:*")                                  │
  │   4. 如果任何步骤失败：                                                              │
  │      - 输出错误日志："Failed to configure APIUpdateExtension Redis subscriber"        │
  │      - this.subscriber = null                                                        │
  │      - this.configured = false                                                        │
  │      - 没有订阅建立，无法接收消息                                                     │
  │                                                                                    │
  │ ⚠️ 关键点：订阅端**优先**使用 REDIS_COLLABORATION_URL，与发布端逻辑不同！             │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ├─────────────── 订阅失败 ─────────► ❌ 无法接收消息，链路终止
         │
         │ 订阅成功
         ▼ C2.3 检查
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ C2.3: 发布与订阅是否使用同一 Redis 实例？（最容易断链的条件）                         │
  │                                                                                    │
  │ 【代码证据对比表】                                                                  │
  │                                                                                    │
  │ ┌──────────────────────────────────────────────────────────────────────────────┐ │
  │ │ 组件          │ 代码位置                          │ Redis 选择逻辑             │ │
  │ ├───────────────┼───────────────────────────────────┼───────────────────────────┤ │
  │ │ 发布端        │ server/collaboration/APIUpdate    │ 硬编码使用                │ │
  │ │               │ Extension.ts:203                   │ RedisAdapter.defaultClient │ │
  │ │               │                                   │ → 始终 env.REDIS_URL       │ │
  │ ├───────────────┼───────────────────────────────────┼───────────────────────────┤ │
  │ │ 订阅端        │ server/collaboration/APIUpdate    │ 优先使用                  │ │
  │ │               │ Extension.ts:56                    │ env.REDIS_COLLABORATION   │ │
  │ │               │                                   │ _URL \|\| env.REDIS_URL     │ │
  │ └──────────────────────────────────────────────────────────────────────────────┘ │
  │                                                                                    │
  │ 【真值表：发布订阅 Redis 一致性检查】                                                │
  │                                                                                    │
  │ ┌──────────────────────────────────────────────────────────────────────────────┐ │
  │ │ REDIS_COLLABORATION_URL │ 发布端使用 │ 订阅端使用 │ 是否同一实例 │ 链路状态   │ │
  │ ├──────────────────────────┼────────────┼────────────┼──────────────┼────────────┤ │
  │ │ 未配置（falsy）          │ REDIS_URL  │ REDIS_URL  │ ✅ 是       │ ✅ 正常    │ │
  │ │ = REDIS_URL              │ REDIS_URL  │ REDIS_URL  │ ✅ 是       │ ✅ 正常    │ │
  │ │ ≠ REDIS_URL（不同实例）  │ REDIS_URL  │ REDIS_COLL │ ❌ 否       │ ❌ 断链！  │ │
  │ │                          │            │ ABORATION  │              │            │ │
  │ │                          │            │ _URL       │              │            │ │
  │ └──────────────────────────────────────────────────────────────────────────────┘ │
  │                                                                                    │
  │ 【断链场景的真实生效逻辑】                                                          │
  │                                                                                    │
  │ 必须**同时满足**以下所有条件才会断链：                                              │
  │                                                                                    │
  │ 条件 A: env.SERVICES.includes("collaboration") → true                            │
  │         → 协作服务已启用（否则没有订阅端）                                          │
  │                                                                                    │
  │ 条件 B: env.REDIS_COLLABORATION_URL 已配置（非空、非 undefined）                   │
  │         → 订阅端使用 REDIS_COLLABORATION_URL                                      │
  │                                                                                    │
  │ 条件 C: env.REDIS_COLLABORATION_URL !== env.REDIS_URL                            │
  │         → 两者指向不同的 Redis 实例                                                 │
  │         → 注意：即使是同一主机的不同端口，也是不同实例！                              │
  │                                                                                    │
  │ 条件 D: 文档通过 HTTP 接口更新且 state 字段变化                                     │
  │         → model.changed("state") → true                                           │
  │         → 导致 notifyUpdate() 被调用                                               │
  │                                                                                    │
  │ 【断链示意图】                                                                      │
  │                                                                                    │
  │   HTTP API 更新                                                                    │
  │        │                                                                           │
  │        ▼                                                                           │
  │   ┌─────────────────┐                                                              │
  │   │   任意实例      │                                                              │
  │   │  (处理请求)     │                                                              │
  │   └────────┬────────┘                                                              │
  │        │                                                                           │
  │        ▼ notifyUpdate()                                                            │
  │        │ 使用 RedisAdapter.defaultClient                                           │
  │        │ → env.REDIS_URL                                                           │
  │        ▼                                                                           │
  │   ┌─────────────────┐                                                              │
  │   │   Redis A       │  ← REDIS_URL                                                 │
  │   │                 │                                                              │
  │   │  发布消息到：    │                                                              │
  │   │  collaboration: │                                                              │
  │   │  api-update:X   │                                                              │
  │   └─────────────────┘                                                              │
  │        │                                                                           │
  │        │ ⚠️ 消息在这里 "消失" 了                                                   │
  │        │ 因为订阅端在另一个 Redis 实例                                              │
  │        ▼                                                                           │
  │   ┌─────────────────┐                                                              │
  │   │   Redis B       │  ← REDIS_COLLABORATION_URL                                  │
  │   │                 │                                                              │
  │   │  订阅频道：      │                                                              │
  │   │  collaboration: │                                                              │
  │   │  api-update:*   │                                                              │
  │   │                 │                                                              │
  │   │  ❌ 没有收到消息 │                                                              │
  │   └─────────────────┘                                                              │
  │                                                                                    │
  │ 结果：HTTP 更新无法同步到正在进行的协作会话！                                         │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ├─────────────── 不同实例 ─────────► ❌ 消息断链，链路终止
         │
         │ 同一实例
         ▼
  ✅ 消息已被订阅端成功接收
         │
         ▼
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ 【阶段 3：文档状态条件】 —— 决定是否处理接收到的消息                                 │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼ C3.1 检查
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ C3.1: 目标文档是否已在**当前实例**的内存中加载？                                    │
  │                                                                                    │
  │ 代码证据 1: server/collaboration/APIUpdateExtension.ts:125-137                    │
  │   private handleMessage = async (                                                   │
  │       _pattern: string,                                                             │
  │       channel: string,                                                              │
  │       message: string                                                               │
  │   ): Promise<void> => {                                                             │
  │       try {                                                                         │
  │           const documentId = channel.replace(`${CHANNEL_PREFIX}:`, "");            │
  │           const document = this.documents.get(documentId);  // 从 Map 中获取       │
  │                                                                                    │
  │           // ⚠️ 关键：如果文档不在内存中，直接忽略！                                  │
  │           if (!document) {                                                          │
  │               // Document not loaded in this instance, ignore                       │
  │               return;  // 直接返回，不做任何处理                                     │
  │           }                                                                          │
  │           // ... 后续处理                                                            │
  │       }                                                                              │
  │   };                                                                                 │
  │                                                                                    │
  │ 代码证据 2: server/collaboration/APIUpdateExtension.ts:92-98                        │
  │   // 文档何时被加载到内存？                                                          │
  │   async afterLoadDocument({                                                          │
  │       documentName,                                                                  │
  │       document,                                                                      │
  │   }: afterLoadDocumentPayload): Promise<void> {                                     │
  │       const [, documentId] = documentName.split(".");                               │
  │       // ⚠️ 关键：只有当 Hocuspocus 加载文档时才会加入 Map                           │
  │       // Hocuspocus 加载文档的条件是：有客户端通过协作 WebSocket 连接                │
  │       this.documents.set(documentId, document);                                      │
  │   }                                                                                  │
  │                                                                                    │
  │ 真实生效逻辑:                                                                        │
  │   - this.documents 是一个 Map<string, HocuspocusDocument>                          │
  │   - 只有当有客户端通过 `/collaboration` WebSocket 连接到某个文档时：                │
  │     → Hocuspocus 调用 afterLoadDocument()                                            │
  │     → 文档被加入 this.documents Map                                                  │
  │   - 当收到 Redis 消息时：                                                           │
  │     → 调用 this.documents.get(documentId)                                            │
  │     → 如果返回 undefined → 直接 return，不做任何处理                                 │
  │                                                                                    │
  │ ⚠️ 关键点：这是**实例级**的检查！                                                     │
  │   - 每个实例有自己独立的 this.documents Map                                          │
  │   - 文档只在**有活跃协作连接的实例**的内存中加载                                      │
  │   - 即使消息被所有实例的订阅端收到，也只在**文档已加载的实例**上被处理               │
  │                                                                                    │
  │ 【多实例场景示例】                                                                   │
  │                                                                                    │
  │   假设：                                                                            │
  │   - 有 3 个实例：Instance A, B, C                                                   │
  │   - 客户端 X 连接到 Instance A，协作编辑文档 D                                       │
  │   - HTTP 更新请求被负载均衡路由到 Instance B                                         │
  │                                                                                    │
  │   消息发布：                                                                        │
  │   - Instance B 调用 notifyUpdate()                                                  │
  │   - 消息发布到 Redis（假设链路正常）                                                 │
  │                                                                                    │
  │   消息接收：                                                                        │
  │   - Instance A: 收到消息 ✅                                                         │
  │   - Instance B: 收到消息 ✅                                                         │
  │   - Instance C: 收到消息 ✅                                                         │
  │                                                                                    │
  │   消息处理（关键！）：                                                               │
  │   - Instance A:                                                                     │
  │     → this.documents.get("D") → 找到 ✅（因为有客户端连接）                          │
  │     → 处理消息 ✅                                                                   │
  │     → 同步到客户端 X ✅                                                              │
  │                                                                                    │
  │   - Instance B:                                                                     │
  │     → this.documents.get("D") → 未找到 ❌（没有客户端连接）                         │
  │     → 直接 return ❌                                                                 │
  │     → 不做任何处理 ❌                                                                │
  │                                                                                    │
  │   - Instance C:                                                                     │
  │     → this.documents.get("D") → 未找到 ❌                                           │
  │     → 直接 return ❌                                                                 │
  │     → 不做任何处理 ❌                                                                │
  │                                                                                    │
  │   最终结果：只有 Instance A 上的客户端 X 能收到更新！                                 │
  │   这是设计上的正确行为——只有有活跃协作连接的实例才需要同步更新                        │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ├─────────────── 未加载 ───────────► ❌ 直接忽略，链路终止
         │
         │ 已加载
         ▼ C3.2 检查
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ C3.2: 文档是否尚未从内存中卸载？                                                     │
  │                                                                                    │
  │ 代码证据 1: server/collaboration/APIUpdateExtension.ts:112-119                    │
  │   // 文档何时被卸载？                                                                │
  │   async onDisconnect({                                                              │
  │       documentName,                                                                  │
  │       clientsCount,                                                                  │
  │   }: onDisconnectPayload): Promise<void> {                                          │
  │       // ⚠️ 关键：当所有客户端都断开时，从内存中删除                                 │
  │       if (clientsCount === 0) {                                                      │
  │           const [, documentId] = documentName.split(".");                           │
  │           this.documents.delete(documentId);  // 从 Map 中删除                      │
  │       }                                                                              │
  │   }                                                                                  │
  │                                                                                    │
  │ 代码证据 2: server/collaboration/APIUpdateExtension.ts:100-107                     │
  │   // 服务销毁时                                                                      │
  │   async onDestroy(_data: onDestroyPayload): Promise<void> {                        │
  │       if (this.subscriber) {                                                         │
  │           await this.subscriber.punsubscribe(`${CHANNEL_PREFIX}:*`);               │
  │           await this.subscriber.quit();                                              │
  │           this.subscriber = null;                                                    │
  │       }                                                                              │
  │       this.documents.clear();  // 清空所有文档                                       │
  │   }                                                                                  │
  │                                                                                    │
  │ 真实生效逻辑:                                                                        │
  │   文档从内存中卸载的条件（满足任一即可）：                                             │
  │                                                                                    │
  │   条件 1: 所有协作客户端断开连接                                                      │
  │     → onDisconnect() 被调用                                                         │
  │     → clientsCount === 0                                                             │
  │     → this.documents.delete(documentId)                                              │
  │                                                                                    │
  │   条件 2: 服务被销毁                                                                 │
  │     → onDestroy() 被调用                                                            │
  │     → this.documents.clear()                                                         │
  │                                                                                    │
  │ 【时序问题场景】                                                                     │
  │                                                                                    │
  │   假设：                                                                            │
  │   - Time T0: 最后一个客户端断开协作连接                                               │
  │     → onDisconnect() 被调用                                                         │
  │     → clientsCount === 0                                                             │
  │     → this.documents.delete("D") ✅                                                  │
  │                                                                                    │
  │   - Time T1（T1 > T0): HTTP API 更新文档 D                                           │
  │     → 消息发布到 Redis ✅                                                            │
  │     → 订阅端收到消息 ✅                                                               │
  │     → this.documents.get("D") → undefined ❌                                         │
  │     → 直接 return ❌                                                                 │
  │                                                                                    │
  │   结果：即使是刚刚断开连接的文档，也无法同步 API 更新                                  │
  │   这是预期行为——没有活跃协作连接，不需要同步                                          │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ├─────────────── 已卸载 ───────────► ❌ 直接忽略，链路终止
         │
         │ 未卸载
         ▼
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ 【阶段 4：数据库条件】 —— 决定能否成功应用更新                                       │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼ C4.1 检查
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ C4.1: 文档在数据库中是否存在且有有效的 state 字段？                                  │
  │                                                                                    │
  │ 代码证据: server/collaboration/APIUpdateExtension.ts:147-160                       │
  │   // 从数据库获取最新状态                                                            │
  │   const dbDocument = await Document.unscoped().findOne({                            │
  │       attributes: ["state", "content", "text"],                                     │
  │       where: { id: documentId },                                                     │
  │   });                                                                                │
  │                                                                                    │
  │   // 检查 1: 文档是否存在                                                            │
  │   if (!dbDocument) {                                                                 │
  │       Logger.warn(`Document ${documentId} not found in database`);                  │
  │       return;  // 直接返回                                                           │
  │   }                                                                                  │
  │                                                                                    │
  │   // 检查 2: state 字段是否存在                                                      │
  │   if (!dbDocument.state) {                                                          │
  │       Logger.warn(`Document ${documentId} has no state in database`);              │
  │       return;  // 直接返回                                                           │
  │   }                                                                                  │
  │                                                                                    │
  │ 真实生效逻辑:                                                                        │
  │   1. 使用 unscoped() 查询——忽略软删除标记                                             │
  │   2. 只查询必要字段：state, content, text                                            │
  │   3. 如果 dbDocument 为 null → 文档不存在 → return                                  │
  │   4. 如果 dbDocument.state 为 null/undefined → return                               │
  │                                                                                    │
  │ 可能导致失败的场景：                                                                 │
  │   - 文档已被硬删除                                                                   │
  │   - 文档已被软删除但 unscoped() 也找不到（实际不会发生）                              │
  │   - 文档存在但 state 字段为 null（可能是旧数据或导入失败）                            │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ├─────────────── 不存在/无 state ──► ❌ 不应用更新，链路终止
         │
         │ 存在且有 state
         ▼
  【阶段 5：应用更新】
  - 从数据库读取最新 state: dbDocument.state
  - 创建临时 Y.Doc: const dbYdoc = new Y.Doc(); Y.applyUpdate(dbYdoc, dbDocument.state)
  - 计算增量更新:
      const currentStateVector = Y.encodeStateVector(document);  // 当前内存状态
      const update = Y.encodeStateAsUpdate(dbYdoc, currentStateVector);  // 增量
  - 应用更新: if (update.length > 0) { Y.applyUpdate(document, update); }
  - 完成！✅

═══════════════════════════════════════════════════════════════════════════════════
【关键结论】
API 更新同步到协作会话需要同时满足以上 **11 个条件**（C1.1-C4.1）！
═══════════════════════════════════════════════════════════════════════════════════
```

### 3.2 条件矩阵详表（严格版）

| 阶段 | 条件编号 | 条件描述 | 真实生效逻辑（代码级） | 代码位置 | 不满足时的行为 | 调试方法 |
|------|----------|----------|------------------------|----------|----------------|----------|
| **阶段 1：发布端** | | | | | | |
| | C1.1 | 文档 `state` 字段发生变化 | `model.changed("state")` 必须返回 `true` | `server/models/Document.ts:587` | 不触发 `notifyUpdate()` | 检查更新是否实际修改了 state 字段；参考测试 `documentUpdater.test.ts` |
| | C1.2 | 请求上下文中有认证用户 | `ctx.auth?.user?.id` 必须存在（非 `undefined`） | `server/models/Document.ts:587` | 不触发 `notifyUpdate()` | 检查是否来自认证 API 请求；后台任务更新可能不满足 |
| | C1.3 | 数据库事务已提交（如有） | 有事务时等待 `transaction.afterCommit()`，无事务时立即执行 | `server/models/Document.ts:593-598` | 事务回滚则不执行 | 检查事务是否正常提交；注意 `void notify()` 不阻塞 |
| | C1.4 | 发布端 Redis 连接正常 | `RedisAdapter.defaultClient.publish()` 必须成功 | `server/collaboration/APIUpdateExtension.ts:203` | 消息未发布 | 检查 `REDIS_URL` 是否正确；检查 Redis 连接状态和日志 |
| **阶段 2：订阅端** | | | | | | |
| | C2.1 | 协作服务已启用 | `env.SERVICES.includes("collaboration")` 为 `true`，导致 `collaboration.init()` 被调用 | `server/index.ts:181-188` | `APIUpdateExtension` 未加载，无订阅 | 检查启动日志是否有 `"Starting collaboration service"`；检查 `SERVICES` 环境变量 |
| | C2.2 | 订阅端 Redis 连接正常且订阅成功 | `onConfigure()` 中 `psubscribe()` 成功，`this.subscriber` 不为 `null`，`this.configured` 为 `true` | `server/collaboration/APIUpdateExtension.ts:47-90` | 无法接收消息 | 检查日志是否有 `"Failed to configure APIUpdateExtension Redis subscriber"`；检查订阅端 Redis URL |
| | C2.3 | 发布与订阅使用同一 Redis 实例 | 发布端用 `REDIS_URL`，订阅端用 `REDIS_COLLABORATION_URL \|\| REDIS_URL`，两者必须指向同一实例 | 发布: `server/collaboration/APIUpdateExtension.ts:203`<br>订阅: `server/collaboration/APIUpdateExtension.ts:56` | **消息断链！** 发布在 Redis A，订阅在 Redis B | 检查 `REDIS_URL` 和 `REDIS_COLLABORATION_URL` 是否相同；如果配置了协作 Redis，必须确保两者一致或修复代码 |
| **阶段 3：文档状态** | | | | | | |
| | C3.1 | 目标文档已在当前实例内存加载 | `this.documents.get(documentId)` 返回非 `undefined` | `server/collaboration/APIUpdateExtension.ts:132-137` | 直接 `return`，不处理消息 | 检查该实例是否有活跃的协作连接；检查日志是否有 `"Received API update for document"` |
| | C3.2 | 文档尚未从内存卸载 | `clientsCount > 0` 且服务未销毁 | `server/collaboration/APIUpdateExtension.ts:112-119` | 文档已从 Map 中删除 | 检查是否所有客户端都已断开；检查服务状态 |
| **阶段 4：数据库** | | | | | | |
| | C4.1 | 文档在数据库中存在且有有效的 `state` 字段 | `Document.unscoped().findOne()` 找到记录且 `dbDocument.state` 存在（非 `null/undefined`） | `server/collaboration/APIUpdateExtension.ts:147-160` | 不应用更新 | 检查日志是否有 `"Document ${documentId} not found in database"` 或 `"has no state"` |

### 3.3 关键条件深度分析

#### 3.3.1 最容易断链的条件：C2.3（发布订阅 Redis 一致性）

**代码证据对比**：

```typescript
// ========== 发布端：硬编码使用 REDIS_URL ==========
// server/collaboration/APIUpdateExtension.ts:193-209
static async notifyUpdate(documentId: string, actorId: string): Promise<void> {
  const channel = `${CHANNEL_PREFIX}:${documentId}`;
  const message = JSON.stringify({ actorId, timestamp: Date.now() });
  
  // ⚠️ 关键：这里硬编码使用 RedisAdapter.defaultClient
  // defaultClient 的定义在 server/storage/redis.ts:129-136
  // 只使用 env.REDIS_URL，完全不考虑 REDIS_COLLABORATION_URL
  await RedisAdapter.defaultClient.publish(channel, message);
}

// ========== 订阅端：优先使用 REDIS_COLLABORATION_URL ==========
// server/collaboration/APIUpdateExtension.ts:47-61
async onConfigure(_data: onConfigurePayload): Promise<void> {
  if (this.configured) { return; }
  this.configured = true;

  try {
    this.subscriber = new RedisAdapter(
      // ⚠️ 关键：这里优先使用 REDIS_COLLABORATION_URL
      // 只有未配置时才使用 REDIS_URL
      env.REDIS_COLLABORATION_URL || env.REDIS_URL,
      {
        connectionNameSuffix: "collab-api-updates",
        maxRetriesPerRequest: null,
      }
    );
    // ... psubscribe ...
  } catch (error) {
    // 失败时标记
    this.subscriber = null;
    this.configured = false;
  }
}
```

**断链条件真值表**：

| 条件编号 | 条件 | 代码判断 | 断链时的值 |
|----------|------|----------|------------|
| DC1 | `SERVICES` 包含 "collaboration" | `env.SERVICES.includes("collaboration")` | `true` |
| DC2 | `REDIS_COLLABORATION_URL` 已配置 | `env.REDIS_COLLABORATION_URL !== undefined && env.REDIS_COLLABORATION_URL !== ""` | `true` |
| DC3 | `REDIS_COLLABORATION_URL ≠ REDIS_URL` | 两个 URL 指向不同 Redis 实例 | `true` |
| DC4 | `state` 字段变化 | `model.changed("state")` | `true` |

**必须同时满足 DC1-DC4 才会断链！**

#### 3.3.2 最容易被忽视的条件：C3.1（文档已在内存加载）

**代码证据**：

```typescript
// server/collaboration/APIUpdateExtension.ts:125-137
private handleMessage = async (
  _pattern: string,
  channel: string,
  message: string
): Promise<void> => {
  try {
    const documentId = channel.replace(`${CHANNEL_PREFIX}:`, "");
    const document = this.documents.get(documentId);

    // ⚠️ 关键：如果文档不在内存中，直接 return
    // 没有日志，没有错误提示，静默忽略！
    if (!document) {
      // Document not loaded in this instance, ignore
      return;
    }
    // ... 后续处理
  }
};
```

**多实例部署时的特殊行为**：

| 场景 | 实例 A（有协作连接） | 实例 B（无协作连接） | 实例 C（无协作连接） |
|------|---------------------|---------------------|---------------------|
| 收到 Redis 消息 | ✅ 是 | ✅ 是 | ✅ 是 |
| `documents.get()` | ✅ 找到 | ❌ `undefined` | ❌ `undefined` |
| 处理消息 | ✅ 是 | ❌ 直接 return | ❌ 直接 return |
| 同步到客户端 | ✅ 是 | ❌ 否 | ❌ 否 |

**这是设计上的正确行为**——只有有活跃协作连接的实例才需要同步更新。

---

## 4. 协作服务多实例部署的真实生效条件

### 4.1 条件详表（与真实代码逻辑逐条对应）

| 条件编号 | 条件描述 | 真实生效逻辑（代码级） | 代码位置 | 不满足时的行为 |
|----------|----------|------------------------|----------|----------------|
| CS1 | `SERVICES` 包含 `"collaboration"` | `env.SERVICES.includes("collaboration")` 必须为 `true`，导致循环中 `collaboration.init()` 被调用 | `server/index.ts:181-188` | 协作服务完全不启动，`APIUpdateExtension` 不加载 |
| CS2 | `REDIS_COLLABORATION_URL` 已配置 | `env.REDIS_COLLABORATION_URL` 必须非空（truthy），导致两个独立影响：<br>1. `@hocuspocus/extension-redis` 被加入 extensions<br>2. `webProcessCount` 不被强制设置为 1 | 1. `server/services/collaboration.ts:49-55`<br>2. `server/index.ts:38-46` | 1. Hocuspocus Redis 扩展不加载，协作状态无法跨实例同步<br>2. 进程数被强制限制为 1 |
| CS3（API 更新链路） | `REDIS_URL` 与 `REDIS_COLLABORATION_URL` 指向同一 Redis 实例 | 发布端硬编码用 `REDIS_URL`，订阅端优先用 `REDIS_COLLABORATION_URL`，两者必须一致 | 发布: `server/collaboration/APIUpdateExtension.ts:203`<br>订阅: `server/collaboration/APIUpdateExtension.ts:56` | API 更新消息断链，无法同步到协作会话 |

### 4.2 进程限制的真实生效逻辑

**代码证据**（`server/index.ts:34-47`）：

```typescript
// The number of processes to run, defaults to the number of CPU's available
// for the web service, and 1 for collaboration unless REDIS_COLLABORATION_URL is set.
let webProcessCount = env.WEB_CONCURRENCY;

// ⚠️ 真实生效逻辑：
// 只有当两个条件**同时满足**时，才会限制进程数：
//   1. env.SERVICES 包含 "collaboration"
//   2. env.REDIS_COLLABORATION_URL 未配置（falsy）
if (env.SERVICES.includes("collaboration") && !env.REDIS_COLLABORATION_URL) {
  if (webProcessCount !== 1) {
    Logger.info(
      "lifecycle",
      "Note: Restricting process count to 1 due to use of collaborative service without REDIS_COLLABORATION_URL"
    );
  }

  webProcessCount = 1;  // 强制设置为 1
}
```

**真值表（与代码逻辑完全对应）**：

| `SERVICES` 包含 "collaboration" | `REDIS_COLLABORATION_URL` 已配置 | `webProcessCount` 限制 | 实际行为 |
|----------------------------------|-----------------------------------|------------------------|----------|
| ❌ 否 | 任意 | ❌ 不限制 | 协作服务不启动，无需限制 |
| ✅ 是 | ❌ 否 | ✅ 强制为 1 | 输出日志 `"Restricting process count to 1..."` |
| ✅ 是 | ✅ 是 | ❌ 不限制 | 使用 `WEB_CONCURRENCY` 的值 |

### 4.3 Hocuspocus Redis 扩展的真实生效逻辑

**代码证据**（`server/services/collaboration.ts:48-55`）：

```typescript
extensions: [
  // ⚠️ 真实生效逻辑：
  // 使用展开运算符，只有当 env.REDIS_COLLABORATION_URL 为 truthy 时，
  // 才会将 [new Redis(...)] 展开到 extensions 数组中
  ...(env.REDIS_COLLABORATION_URL
    ? [
        new Redis({
          redis: RedisAdapter.collaborationClient,
        }),
      ]
    : []),  // falsy 时展开空数组
  // ... 其他扩展
]
```

**真值表**：

| `REDIS_COLLABORATION_URL` | 条件判断结果 | `@hocuspocus/extension-redis` | 协作状态同步方式 |
|---------------------------|--------------|-------------------------------|------------------|
| `undefined` | falsy | ❌ 不加载 | 仅内存存储，无法跨实例 |
| `""`（空字符串） | falsy | ❌ 不加载 | 仅内存存储，无法跨实例 |
| `"redis://..."`（非空） | truthy | ✅ 加载 | 通过 Redis pub/sub 跨实例同步 |

### 4.4 多实例部署的完整条件链路图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│              协作服务多实例部署的完整条件链路（严格版）                                 │
└─────────────────────────────────────────────────────────────────────────────────────┘

  想要多实例部署协作服务且 API 更新链路正常，必须同时满足：

  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ 条件 CS1: SERVICES 包含 "collaboration"                                            │
  │                                                                                    │
  │ 真实生效逻辑（代码级）:                                                             │
  │   server/index.ts:181-188                                                          │
  │                                                                                    │
  │   for (const name of env.SERVICES) {                                               │
  │       const init = services[name as keyof typeof services];                        │
  │       await init(app, server, env.SERVICES);  // 只有 SERVICES 包含时才调用        │
  │   }                                                                                  │
  │                                                                                    │
  │ 检查方法:                                                                          │
  │   - 查看启动日志是否有: "Starting collaboration service"                            │
  │   - 检查环境变量 SERVICES 是否包含 "collaboration"                                  │
  └──────────────────────────────────────────────────────────────────────────────────┘
           │
           ├─────────────── 不满足 ─────────────► ❌ 协作服务不启动
           │
           │ 满足
           ▼
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ 条件 CS2: 配置 REDIS_COLLABORATION_URL                                            │
  │                                                                                    │
  │ 真实生效逻辑（代码级）:                                                             │
  │   影响 A: 进程数不被限制                                                           │
  │   server/index.ts:38-46                                                            │
  │   if (env.SERVICES.includes("collaboration")                                       │
  │       && !env.REDIS_COLLABORATION_URL) {  // 未配置时才限制                        │
  │       webProcessCount = 1;                                                         │
  │   }                                                                                  │
  │                                                                                    │
  │   影响 B: Hocuspocus Redis 扩展被加载                                               │
  │   server/services/collaboration.ts:49-55                                           │
  │   extensions: [                                                                     │
  │       ...(env.REDIS_COLLABORATION_URL  // 配置了才加载                             │
  │           ? [new Redis({ redis: RedisAdapter.collaborationClient })]              │
  │           : []),                                                                    │
  │   ]                                                                                  │
  │                                                                                    │
  │ 检查方法:                                                                          │
  │   - 查看启动日志是否有: "Restricting process count to 1"（如果有说明未配置）        │
  │   - 检查环境变量 REDIS_COLLABORATION_URL 是否非空                                   │
  └──────────────────────────────────────────────────────────────────────────────────┘
           │
           ├─────────────── 不满足 ─────────────► ❌ 单进程限制 + 无 Redis 扩展
           │
           │ 满足
           ▼
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ 条件 CS3: REDIS_URL = REDIS_COLLABORATION_URL（或指向同一实例）                    │
  │                                                                                    │
  │ 真实生效逻辑（代码级）:                                                             │
  │                                                                                    │
  │   发布端（硬编码）:                                                                 │
  │   server/collaboration/APIUpdateExtension.ts:203                                   │
  │   await RedisAdapter.defaultClient.publish(channel, message);                      │
  │   → defaultClient 使用 env.REDIS_URL（server/storage/redis.ts:129-136）           │
  │                                                                                    │
  │   订阅端（优先）:                                                                   │
  │   server/collaboration/APIUpdateExtension.ts:56                                    │
  │   env.REDIS_COLLABORATION_URL || env.REDIS_URL                                     │
  │                                                                                    │
  │   一致性条件:                                                                      │
  │   发布端使用的 Redis 实例 === 订阅端使用的 Redis 实例                                │
  │                                                                                    │
  │ 检查方法:                                                                          │
  │   - 比较 REDIS_URL 和 REDIS_COLLABORATION_URL                                      │
  │   - 如果都配置了，必须指向同一 Redis 实例                                            │
  │   - 或者: 不配置 REDIS_COLLABORATION_URL，让订阅端自动使用 REDIS_URL                │
  └──────────────────────────────────────────────────────────────────────────────────┘
           │
           ├─────────────── 不满足 ─────────────► ❌ API 更新消息断链！
           │
           │ 满足
           ▼
  ✅ 协作服务支持多实例部署！
  ✅ API 更新链路正常！
```

---

## 5. WebSocket 事件服务实现

### 5.1 初始化与配置

WebSocket 事件服务使用 Socket.IO 库实现，核心配置在 `server/services/websockets.ts` 中：

```typescript
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

### 5.2 Redis 适配器配置

使用 `socket.io-redis` 适配器实现多实例消息广播：

```typescript
io.adapter(
  createAdapter({
    pubClient: Redis.defaultClient,
    subClient: Redis.defaultSubscriber,
  })
);
```

### 5.3 Redis 客户端管理

Redis 客户端管理在 `server/storage/redis.ts` 中实现，提供三种客户端：

| 客户端 | 用途 | 环境变量 | 代码位置 |
|--------|------|----------|----------|
| `defaultClient` | 通用操作、发布消息 | `REDIS_URL` | `server/storage/redis.ts:129-136` |
| `defaultSubscriber` | 订阅专用（Socket.IO） | `REDIS_URL` | `server/storage/redis.ts:140-148` |
| `collaborationClient` | 协作服务专用 | `REDIS_COLLABORATION_URL \|\| REDIS_URL` | `server/storage/redis.ts:150-159` |

---

## 6. 部署与配置建议（严格版）

### 6.1 环境变量配置与真实生效逻辑对应表

| 环境变量 | 真实生效逻辑 | 影响的条件 | 验证方法 |
|----------|--------------|------------|----------|
| `SERVICES` | 必须包含 `"collaboration"` 才会启动协作服务 | CS1 | 检查启动日志：`"Starting collaboration service"` |
| `REDIS_URL` | 用于：1) Socket.IO 适配器 2) API 更新**发布端**（硬编码） 3) 默认缓存 | C1.4, C2.3 | 检查 `RedisAdapter.defaultClient` 连接状态 |
| `REDIS_COLLABORATION_URL` | 配置后：1) 进程数不被限制 2) Hocuspocus Redis 扩展被加载 3) API 更新**订阅端**优先使用 | CS2, C2.3 | 检查启动日志是否有 `"Restricting process count to 1"` |
| `WEB_CONCURRENCY` | 只有当 `SERVICES` 包含 `"collaboration"` 且未配置 `REDIS_COLLABORATION_URL` 时，才会被强制设置为 1 | CS2 | 检查实际进程数；检查启动日志 |

### 6.2 配置最佳实践（与代码逻辑严格对应）

#### 推荐配置方式 A：单 Redis 实例（简单部署）

```bash
# 环境变量
SERVICES=web,collaboration
REDIS_URL=redis://redis:6379
# REDIS_COLLABORATION_URL 不设置（undefined）
```

**真实生效逻辑验证**：

| 条件 | 代码判断结果 | 状态 |
|------|-------------|------|
| CS1 | `env.SERVICES.includes("collaboration")` → `true` | ✅ 满足 |
| CS2 | `env.REDIS_COLLABORATION_URL` → `undefined` (falsy) | ❌ 不满足 |
| - 进程限制 | `webProcessCount` 被强制设置为 1 | ⚠️ 单进程 |
| - Hocuspocus Redis 扩展 | 不加载 | ⚠️ 无跨实例同步 |
| C2.3 (API 更新链路) | 发布端和订阅端都用 `REDIS_URL` | ✅ 正常 |

**适用场景**：单实例部署、开发环境

---

#### 推荐配置方式 B：单 Redis 实例 + 多进程协作（推荐）

```bash
# 环境变量
SERVICES=web,collaboration
REDIS_URL=redis://redis:6379
REDIS_COLLABORATION_URL=redis://redis:6379  # ⚠️ 与 REDIS_URL 相同！
WEB_CONCURRENCY=4
```

**真实生效逻辑验证**：

| 条件 | 代码判断结果 | 状态 |
|------|-------------|------|
| CS1 | `env.SERVICES.includes("collaboration")` → `true` | ✅ 满足 |
| CS2 | `env.REDIS_COLLABORATION_URL` → `truthy` | ✅ 满足 |
| - 进程限制 | `webProcessCount` 不被限制 → 使用 4 | ✅ 多进程 |
| - Hocuspocus Redis 扩展 | 加载 | ✅ 跨实例同步 |
| C2.3 (API 更新链路) | 发布端 `REDIS_URL`，订阅端 `REDIS_COLLABORATION_URL` → 同一实例 | ✅ 正常 |

**✅ 这是唯一推荐的多实例部署配置！**

---

#### 不推荐配置方式 C：双 Redis 实例（存在断链风险）

```bash
# 环境变量（不推荐）
SERVICES=web,collaboration
REDIS_URL=redis://redis-default:6379
REDIS_COLLABORATION_URL=redis://redis-collab:6379  # ⚠️ 与 REDIS_URL 不同！
```

**真实生效逻辑验证**：

| 条件 | 代码判断结果 | 状态 |
|------|-------------|------|
| CS1 | `env.SERVICES.includes("collaboration")` → `true` | ✅ 满足 |
| CS2 | `env.REDIS_COLLABORATION_URL` → `truthy` | ✅ 满足 |
| - 进程限制 | `webProcessCount` 不被限制 | ✅ 多进程 |
| - Hocuspocus Redis 扩展 | 加载 | ✅ 跨实例同步 |
| C2.3 (API 更新链路) | 发布端 `redis-default:6379`，订阅端 `redis-collab:6379` → **不同实例** | ❌ **断链！** |

**⚠️ 问题**：API 更新消息断链，HTTP 更新无法同步到协作会话！

**修复方案**：
1. 将 `REDIS_COLLABORATION_URL` 改为与 `REDIS_URL` 相同（推荐）
2. 或修改代码使发布端也使用 `REDIS_COLLABORATION_URL`

### 6.3 多实例部署前的检查清单

在部署多实例协作服务之前，请逐项验证：

| 检查项 | 验证命令/方法 | 预期结果 | 失败时的排查方向 |
|--------|---------------|----------|------------------|
| 1. `SERVICES` 包含 "collaboration" | 检查启动日志 | 出现 `"Starting collaboration service"` | 检查 `SERVICES` 环境变量 |
| 2. `REDIS_COLLABORATION_URL` 已配置 | 检查启动日志 | **不出现** `"Restricting process count to 1"` | 检查 `REDIS_COLLABORATION_URL` 环境变量 |
| 3. `REDIS_URL` = `REDIS_COLLABORATION_URL` | 比较两个环境变量 | 指向同一 Redis 实例 | 如果不同，修改为相同或修复代码 |
| 4. Hocuspocus Redis 扩展已加载 | 检查代码逻辑或调试日志 | `@hocuspocus/extension-redis` 被加入 extensions | 检查 `REDIS_COLLABORATION_URL` 是否非空 |
| 5. API 更新发布端连接正常 | 检查 Redis 连接日志 | `RedisAdapter.defaultClient` 连接成功 | 检查 `REDIS_URL` 和 Redis 状态 |
| 6. API 更新订阅端连接正常 | 检查日志 | **不出现** `"Failed to configure APIUpdateExtension Redis subscriber"` | 检查 `REDIS_COLLABORATION_URL` 和 Redis 状态 |

### 6.4 关键日志监控

在运行时监控以下日志，及时发现问题：

| 日志内容 | 含义 | 严重程度 |
|----------|------|----------|
| `"Starting collaboration service"` | 协作服务启动成功 | ℹ️ 信息 |
| `"Restricting process count to 1..."` | 未配置协作 Redis，进程被限制 | ⚠️ 警告 |
| `"Subscribed to collaboration:api-update:* for API updates"` | API 更新订阅成功 | ℹ️ 信息 |
| `"Failed to configure APIUpdateExtension Redis subscriber"` | API 更新订阅失败 | ❌ 错误 |
| `"Received API update for document"` | 收到并处理 API 更新 | ℹ️ 信息 |
| `"Applied API update to document"` | 成功应用 API 更新 | ℹ️ 信息 |
| `"Document ${documentId} not found in database"` | 文档不存在于数据库 | ⚠️ 警告 |
| `"Document ${documentId} has no state in database"` | 文档无 state 字段 | ⚠️ 警告 |

---

## 7. 总结

### 7.1 核心架构

Outline 的多实例 WebSocket 集群实现采用了成熟的技术方案：

1. **Socket.IO + socket.io-redis**：实现系统事件的跨实例广播
2. **Hocuspocus + @hocuspocus/extension-redis**：实现协作编辑的实时同步
3. **自定义 Redis pub/sub**：实现 API 更新到协作会话的同步
4. **多 Redis 客户端分离**：发布、订阅、协作使用独立连接，避免阻塞

### 7.2 关键发现（与真实代码逻辑逐条对应）

#### 发现 1：API 更新同步需要满足 11 个连续条件

| 阶段 | 条件编号 | 真实生效逻辑 | 代码位置 |
|------|----------|-------------|----------|
| 发布端 | C1.1 | `model.changed("state") === true` | `server/models/Document.ts:587` |
| | C1.2 | `ctx.auth?.user?.id !== undefined` | `server/models/Document.ts:587` |
| | C1.3 | 事务已提交（如有） | `server/models/Document.ts:593-598` |
| | C1.4 | `RedisAdapter.defaultClient.publish()` 成功 | `server/collaboration/APIUpdateExtension.ts:203` |
| 订阅端 | C2.1 | `env.SERVICES.includes("collaboration") === true` | `server/index.ts:181-188` |
| | C2.2 | `onConfigure()` 成功，`this.subscriber !== null` | `server/collaboration/APIUpdateExtension.ts:47-90` |
| | C2.3 | 发布端 Redis === 订阅端 Redis | 发布: `server/collaboration/APIUpdateExtension.ts:203`<br>订阅: `server/collaboration/APIUpdateExtension.ts:56` |
| 文档状态 | C3.1 | `this.documents.get(documentId) !== undefined` | `server/collaboration/APIUpdateExtension.ts:132-137` |
| | C3.2 | `clientsCount > 0` 且服务未销毁 | `server/collaboration/APIUpdateExtension.ts:112-119` |
| 数据库 | C4.1 | `dbDocument !== null && dbDocument.state !== null` | `server/collaboration/APIUpdateExtension.ts:147-160` |

**所有条件必须同时满足！**

#### 发现 2：协作服务多实例部署的强制条件链

| 条件编号 | 真实生效逻辑 | 代码位置 | 不满足时的影响 |
|----------|-------------|----------|----------------|
| CS1 | `env.SERVICES.includes("collaboration")` | `server/index.ts:181-188` | 协作服务不启动 |
| CS2 | `env.REDIS_COLLABORATION_URL` 已配置（truthy） | `server/index.ts:38-46` + `server/services/collaboration.ts:49-55` | 单进程限制 + 无 Redis 扩展 |
| CS3 | `REDIS_URL` 与 `REDIS_COLLABORATION_URL` 指向同一实例 | 发布: `server/collaboration/APIUpdateExtension.ts:203`<br>订阅: `server/collaboration/APIUpdateExtension.ts:56` | API 更新消息断链 |

#### 发现 3：发布订阅 Redis 选择的不对称性（设计缺陷）

| 组件 | Redis 选择逻辑 | 真实使用的环境变量 | 代码位置 |
|------|----------------|-------------------|----------|
| 发布端 | 硬编码 `RedisAdapter.defaultClient` | **始终 `REDIS_URL`** | `server/collaboration/APIUpdateExtension.ts:203` |
| 订阅端 | `env.REDIS_COLLABORATION_URL \|\| env.REDIS_URL` | **优先 `REDIS_COLLABORATION_URL`** | `server/collaboration/APIUpdateExtension.ts:56` |

**断链条件**（必须同时满足）：
1. `env.SERVICES.includes("collaboration")` → `true`
2. `env.REDIS_COLLABORATION_URL` 已配置（truthy）
3. `REDIS_URL` ≠ `REDIS_COLLABORATION_URL`（指向不同实例）
4. `model.changed("state")` → `true`

---

## 8. 参考代码位置速查表

| 功能模块 | 关键条件 | 代码位置 | 行号 |
|----------|----------|----------|------|
| **API 更新发布触发** | `state` 字段变化 + 认证用户 | `server/models/Document.ts` | 585-599 |
| **API 更新发布** | 硬编码使用 `REDIS_URL` | `server/collaboration/APIUpdateExtension.ts` | 193-209 |
| **API 更新订阅** | 优先使用 `REDIS_COLLABORATION_URL` | `server/collaboration/APIUpdateExtension.ts` | 47-90 |
| **文档加载检查** | 必须在内存 Map 中 | `server/collaboration/APIUpdateExtension.ts` | 125-137 |
| **文档卸载** | `clientsCount === 0` | `server/collaboration/APIUpdateExtension.ts` | 112-119 |
| **协作服务启动** | `SERVICES` 包含 "collaboration" | `server/index.ts` | 181-188 |
| **进程数限制** | 未配置协作 Redis 时强制为 1 | `server/index.ts` | 34-47 |
| **Hocuspocus Redis 扩展** | 配置协作 Redis 时才加载 | `server/services/collaboration.ts` | 48-55 |
| **Redis 客户端定义** | `defaultClient` 只用 `REDIS_URL` | `server/storage/redis.ts` | 129-15