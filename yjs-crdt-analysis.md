# Outline Yjs CRDT 实时协作编辑系统技术分析报告

## 1. 系统架构概览

Outline 的实时协作编辑系统采用了 **Hocuspocus + Yjs CRDT** 技术栈，实现了从前端编辑到后端持久化的完整协作链路。

### 1.1 核心技术栈

| 层级 | 技术方案 | 职责 |
|------|----------|------|
| 前端编辑器 | ProseMirror + y-prosemirror | 富文本编辑与 CRDT 绑定 |
| 前端状态同步 | Hocuspocus Provider + Yjs | 客户端 CRDT 管理与网络同步 |
| 本地持久化 | y-indexeddb | 浏览器端离线缓存 |
| 服务端同步 | Hocuspocus Server | WebSocket 连接管理与消息路由 |
| 服务端扩展 | Redis Extension | 多实例间状态同步 |
| 数据持久化 | PostgreSQL | CRDT 状态与文档快照存储 |

### 1.2 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端 (React App)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐     ┌─────────────────┐     ┌─────────────────────────┐   │
│  │ ProseMirror │────▶│ y-prosemirror   │────▶│      Y.Doc (CRDT)      │   │
│  │   Editor    │     │  (Sync Plugin)  │     │                         │   │
│  └─────────────┘     └─────────────────┘     └─────────────────────────┘   │
│                                                            │                  │
│                              ┌─────────────────────────────┘                  │
│                              ▼                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                     Hocuspocus Provider                                  │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐ │  │
│  │  │ WebSocket Client │  │ Awareness (光标) │  │ IndexeddbPersistence │ │  │
│  │  └──────────────────┘  └──────────────────┘  └──────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ WebSocket (/collaboration)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           服务端 (Node.js + Koa)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                        Hocuspocus Server                                  │ │
│  │                                                                           │ │
│  │  Extensions Chain:                                                        │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐              │ │
│  │  │ Throttle │ │Auth      │ │Editor    │ │Persistence   │              │ │
│  │  │          │ │Extension │ │Version    │ │Extension     │              │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────────┘              │ │
│  │  ┌──────────────┐ ┌──────────┐ ┌──────────┐                            │ │
│  │  │ APIUpdate    │ │Views     │ │Logger    │                            │ │
│  │  │ Extension    │ │Extension │ │Extension │                            │ │
│  │  └──────────────┘ └──────────┘ └──────────┘                            │ │
│  │                                                                           │ │
│  │  Optional: Redis Extension (多实例部署)                                   │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                          │
│                                      ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                   documentCollaborativeUpdater Command                    │ │
│  │  - 事务更新 PostgreSQL                                                      │ │
│  │  - 触发 documents.update 事件                                               │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PostgreSQL 数据库                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  documents 表:                                                                 │
│  ┌─────────────┬─────────────┬─────────────┬─────────────────────────────┐  │
│  │   state     │   content   │    text     │    collaboratorIds          │  │
│  │  (BLOB)     │  (JSONB)    │  (TEXT)     │      (UUID[])               │  │
│  │ Yjs二进制   │ ProseMirror │  Markdown   │  协作者用户ID列表            │  │
│  │  CRDT状态   │   快照      │   快照      │                             │  │
│  └─────────────┴─────────────┴─────────────┴─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 前端协作编辑初始化流程

### 2.1 核心组件：MultiplayerEditor

`app/scenes/Document/components/MultiplayerEditor.tsx` 是协作编辑的入口组件。

#### 2.1.1 初始化流程

```typescript
// 1. 创建 Yjs 文档实例
const [ydoc] = useState(() => new Y.Doc());

// 2. 配置本地持久化 (IndexedDB)
const localProvider =
  typeof indexedDB !== "undefined"
    ? new IndexeddbPersistence(name, ydoc)
    : undefined;

// 3. 配置远程同步 (Hocuspocus WebSocket)
const provider = new HocuspocusProvider({
  parameters: {
    editorVersion: EDITOR_VERSION,
  },
  url: `${env.COLLABORATION_URL}/collaboration`,
  name,  // document.${documentId}
  document: ydoc,
  token,   // JWT 协作令牌
});
```

**关键点说明**：
- **文档命名规范**：`document.${documentId}`，用于服务端路由
- **双持久化策略**：IndexedDB 本地缓存 + 远程服务端同步
- **惰性初始化**：使用 `useLayoutEffect` 而非 `useState` 避免 React StrictMode 下的重复连接问题

#### 2.1.2 事件处理机制

| 事件类型 | 处理逻辑 | 文件位置 |
|----------|----------|----------|
| `synced` | 标记远程同步完成，重置重试计数 | L176-180 |
| `close` | 处理连接关闭，检查错误码（如编辑器版本不兼容） | L182-195 |
| `status` | 更新 UI 状态（connecting/connected/disconnected） | L216-220 |
| `awarenessChange` | 更新协作者感知（光标、滚动位置） | L142-159 |
| `authenticationFailed` | 令牌刷新与重连逻辑 | L124-140 |

#### 2.1.3 智能连接管理

```typescript
// 空闲时自动断开连接以节省资源
useEffect(() => {
  if (isIdle && !isVisible && remoteProvider.status === WebSocketStatus.Connected) {
    void remoteProvider.disconnect();
  }
  
  if ((!isIdle || isVisible) && remoteProvider.status === WebSocketStatus.Disconnected) {
    void remoteProvider.connect();
  }
}, [remoteProvider, isIdle, isVisible]);
```

**设计亮点**：
- 页面不可见 + 用户空闲时自动断开
- 恢复可见性时自动重连
- 平衡实时性与资源消耗

### 2.2 编辑器集成：Multiplayer Extension

`app/editor/extensions/Multiplayer.ts` 负责将 Yjs 与 ProseMirror 编辑器绑定。

#### 2.2.1 核心插件配置

```typescript
return [
  ySyncPlugin(type),           // 内容同步
  yCursorPlugin(provider.awareness, {  // 光标同步
    awarenessStateFilter,
    selectionBuilder,
  }),
  yUndoPlugin(),               // 撤销/重做
  new Plugin({
    props: {
      handleScrollToSelection: (view) => isRemoteTransaction(view.state.tr),
    },
  }),
];
```

#### 2.2.2 用户-客户端ID映射

```typescript
// 仅在用户实际做出修改后才建立 userId <-> clientId 映射
const assignUser = (tr: Y.Transaction) => {
  const clientIds = Array.from(doc.store.clients.keys());
  
  if (tr.local && tr.changed.size > 0 && !clientIds.includes(doc.clientID)) {
    const permanentUserData = new Y.PermanentUserData(doc);
    permanentUserData.setUserMapping(doc, doc.clientID, user.id);
    doc.off("afterTransaction", assignUser);
  }
};

doc.on("afterTransaction", assignUser);
```

**设计意图**：
- 避免为只读用户创建不必要的映射
- `PermanentUserData` 用于追踪历史协作者
- 映射信息会被序列化到 CRDT 状态中

#### 2.2.3 感知状态优化

```typescript
// 10秒后自动隐藏远端用户的选择高亮
const selectionTimeout = 10 * Second.ms;

const selectionBuilder = (u: { id: string; color: string }) => {
  const cached = userAwarenessCache.get(u.id);
  const opacity =
    !cached || cached?.changedAt > new Date(Date.now() - selectionTimeout)
      ? selectionOpacity  // 70%
      : 0;  // 透明

  return {
    style: `background-color: ${u.color}${opacity}`,
    class: "ProseMirror-yjs-selection",
  };
};
```

---

## 3. CRDT 更新传播机制

### 3.1 Yjs 核心概念

Yjs 使用 **操作转换 (Operation Transformation)** 的替代方案 **CRDT (Conflict-free Replicated Data Type)**：

| 特性 | 说明 |
|------|------|
| **Y.Doc** | 共享文档根容器 |
| **Y.XmlFragment** | 用于存储富文本内容的类型 |
| **Y.PermanentUserData** | 持久化用户-客户端映射 |
| **State Vector** | 描述客户端已知状态的向量 |
| **Update** | 描述变更的二进制增量 |

### 3.2 数据流：编辑操作到 CRDT 更新

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         用户编辑操作流程                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. 用户在 ProseMirror 中输入文字                                          │
│         │                                                                 │
│         ▼                                                                 │
│  2. ProseMirror Transaction 创建                                          │
│         │                                                                 │
│         ▼                                                                 │
│  3. y-prosemirror ySyncPlugin 拦截                                        │
│         │                                                                 │
│         ▼                                                                 │
│  4. 转换为 Yjs Operation (insert/delete)                                  │
│         │                                                                 │
│         ▼                                                                 │
│  5. Y.Doc 本地应用变更，生成 Update                                        │
│         │                                                                 │
│         ├──────────────┬──────────────┐                                  │
│         ▼              ▼              ▼                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐                          │
│  │ IndexedDB│  │ Awareness │  │ Hocuspocus   │                          │
│  │  本地保存 │  │  广播光标 │  │  Provider    │                          │
│  └──────────┘  └──────────┘  │  同步到服务端 │                          │
│                               └──────────────┘                          │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Hocuspocus Provider 同步机制

#### 3.3.1 同步协议

Hocuspocus 基于 Yjs 的标准同步协议：

```
客户端 (Client)                              服务端 (Server)
    │                                              │
    │  Step 1: 发送 State Vector                   │
    │  (描述本地已知状态)                           │
    │ ───────────────────────────────────────────▶ │
    │                                              │
    │  Step 2: 服务端计算差异                       │
    │  (基于双方 State Vector)                     │
    │                                              │
    │  Step 3: 返回 Update                         │
    │  (客户端缺失的所有变更)                       │
    │ ◀─────────────────────────────────────────── │
    │                                              │
    │  Step 4: 客户端应用 Update                   │
    │  (本地状态与服务端对齐)                       │
    │                                              │
    │  实时更新流:                                  │
    │  任何本地变更 → 即时发送 Update               │
    │  ─────────────────────────────────────────▶ │
    │                                              │
    │  服务端广播其他客户端的 Update                │
    │  ◀────────────────────────────────────────── │
    │                                              │
```

#### 3.3.2 消息格式

```typescript
// Hocuspocus 使用二进制消息格式，基于 Yjs 的编码
// 消息类型包含:
// - Sync Step 1/2
// - Awareness 更新
// - Auth 请求/响应
```

---

## 4. 服务端同步与持久化机制

### 4.1 Hocuspocus Server 配置

`server/services/collaboration.ts` 配置服务端协作服务。

#### 4.1.1 扩展链配置

```typescript
const hocuspocus = Server.configure({
  debounce: 3000,      // 持久化防抖 3秒
  timeout: 30000,      // 连接超时 30秒
  maxDebounce: 10000,  // 最大防抖 10秒
  extensions: [
    // 1. Redis 扩展（多实例部署时）
    ...(env.REDIS_COLLABORATION_URL
      ? [new Redis({ redis: RedisAdapter.collaborationClient })]
      : []),
    
    // 2. 限流扩展
    new Throttle({
      throttle: env.RATE_LIMITER_COLLABORATION_REQUESTS,
      banTime: 5,  // 封禁时间（分钟）
    }),
    
    // 3. 核心业务扩展
    new ConnectionLimitExtension(),
    new EditorVersionExtension(),
    new AuthenticationExtension(),
    new PersistenceExtension(),      // 核心持久化
    new APIUpdateExtension(),        // API 变更同步
    new ViewsExtension(),
    new LoggerExtension(),
    new MetricsExtension(),
  ],
});
```

#### 4.1.2 WebSocket 路由

```typescript
// 协作编辑使用 /collaboration 路径
// 实时事件使用 /realtime 路径 (Socket.io)

server.on("upgrade", function (req, socket, head) {
  if (req.url?.startsWith("/collaboration")) {
    // Hocuspocus 处理协作编辑
    wss.handleUpgrade(req, socket, head, (client) => {
      hocuspocus.handleConnection(client, req, documentId);
    });
  }
  
  if (req.url?.startsWith("/realtime")) {
    // Socket.io 处理普通实时事件
    // 如: documents.update, collections.update 等
  }
});
```

### 4.2 认证扩展：AuthenticationExtension

`server/collaboration/AuthenticationExtension.ts`

```typescript
async onAuthenticate({ connection, token, documentName }) {
  const [, documentId] = documentName.split(".");
  
  // 1. 验证 JWT 令牌
  const { user } = await getUserForJWT(token, ["session", "collaboration"]);
  
  // 2. 加载文档并验证权限
  const document = await Document.findByPk(documentId, { userId: user.id });
  
  // 3. 读权限检查
  if (!can(user, "read", document)) {
    throw AuthenticationError("Authorization required");
  }
  
  // 4. 写权限检查 → 设置只读模式
  if (!can(user, "update", document)) {
    connection.readOnly = true;
  }
  
  return { user };  // 用户信息注入 context
}
```

### 4.3 持久化扩展：PersistenceExtension

`server/collaboration/PersistenceExtension.ts` 是核心持久化逻辑。

#### 4.3.1 文档加载流程 (`onLoadDocument`)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        文档加载时序图                                        │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  客户端连接 → Hocuspocus → onLoadDocument 钩子                             │
│                                     │                                      │
│                                     ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 1. 检查内存中是否已有该文档 (Y.Doc)                                    │ │
│  │    - 若有且字段非空 → 直接返回                                          │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                     │                                      │
│                                     ▼ (无内存副本)                          │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 2. 快速检查数据库是否有 state 字段                                       │ │
│  │    - 无锁查询 state 字段                                               │ │
│  │    - 若存在 → 直接反序列化为 Y.Doc 返回                                  │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                     │                                      │
│                                     ▼ (无 state)                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 3. 事务内初始化 (带行锁)                                                │ │
│  │    - 双重检查: 再次确认 state 是否为空 (防止竞态)                        │ │
│  │    - 若仍为空:                                                          │ │
│  │      a. 优先从 content (JSONB) 转换                                     │ │
│  │      b. 回退到 text (Markdown) 转换                                     │ │
│  │      c. 生成初始 state 并写入数据库                                      │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

**关键代码**：

```typescript
async onLoadDocument({ documentName, ...data }) {
  const [, documentId] = documentName.split(".");
  const fieldName = "default";
  
  // 1. 内存中已有该字段，无需加载
  if (!data.document.isEmpty(fieldName)) {
    return;
  }
  
  // 2. 快速检查: 无锁查询 state
  const documentWithoutLock = await Document.unscoped().findOne({
    attributes: ["state"],
    where: { id: documentId },
  });
  
  if (documentWithoutLock.state) {
    const ydoc = new Y.Doc();
    Y.applyUpdate(ydoc, documentWithoutLock.state);
    return ydoc;
  }
  
  // 3. 事务内初始化 (带行锁防止竞态)
  return await sequelize.transaction(async (transaction) => {
    const document = await Document.unscoped().findOne({
      attributes: ["id", "state", "content", "text"],
      transaction,
      lock: transaction.LOCK.UPDATE,  // 行级锁
      where: { id: documentId },
    });
    
    // 双重检查
    if (document.state) {
      const ydoc = new Y.Doc();
      Y.applyUpdate(ydoc, document.state);
      return ydoc;
    }
    
    // 从 content 或 text 初始化
    let ydoc;
    if (document.content) {
      ydoc = ProsemirrorHelper.toYDoc(document.content, fieldName);
    } else {
      ydoc = ProsemirrorHelper.toYDoc(document.text, fieldName);
    }
    
    // 写入初始 state
    const state = ProsemirrorHelper.toState(ydoc);
    await document.update({ state }, { transaction, hooks: false });
    
    return ydoc;
  });
}
```

#### 4.3.2 变更追踪 (`onChange`)

```typescript
async onChange({ context, documentName }) {
  const [, documentId] = documentName.split(".");
  
  if (context.user) {
    // 记录当前会话的协作者到 Redis Set
    const key = Document.getCollaboratorKey(documentId);
    await Redis.defaultClient.sadd(key, context.user.id);
  }
}
```

#### 4.3.3 文档持久化 (`onStoreDocument`)

```typescript
async onStoreDocument({ document, context, documentName, clientsCount, requestParameters }) {
  const [, documentId] = documentName.split(".");
  const clientVersion = requestParameters.get("editorVersion");
  
  // 1. 获取本次会话的协作者列表
  const key = Document.getCollaboratorKey(documentId);
  const sessionCollaboratorIds = await Redis.defaultClient.smembers(key);
  
  if (!sessionCollaboratorIds || sessionCollaboratorIds.length === 0) {
    return;  // 无变更，跳过
  }
  
  // 2. 执行持久化命令
  try {
    await documentCollaborativeUpdater({
      documentId,
      ydoc: document,
      sessionCollaboratorIds,
      isLastConnection: clientsCount === 0,  // 是否最后一个连接
      clientVersion,
    });
  } catch (err) {
    Logger.error("Unable to persist document", err);
  }
}
```

**触发时机**：
- `debounce: 3000` - 变更后 3 秒防抖
- 最后一个客户端断开时立即触发
- `maxDebounce: 10000` - 最大延迟 10 秒

### 4.4 协作文档更新命令：documentCollaborativeUpdater

`server/commands/documentCollaborativeUpdater.ts` 处理最终的数据库写入。

#### 4.4.1 事务更新流程

```typescript
export default async function documentCollaborativeUpdater({
  documentId, ydoc, sessionCollaboratorIds, isLastConnection, clientVersion
}) {
  return sequelize.transaction(async (transaction) => {
    // 1. 设置锁超时
    await sequelize.query(`SET LOCAL lock_timeout = '15s';`, { transaction });
    
    // 2. 带行锁查询文档
    const document = await Document.unscoped()
      .scope("withoutState")
      .findOne({
        where: { id: documentId },
        transaction,
        lock: { of: Document, level: transaction.LOCK.UPDATE },
      });
    
    // 3. 编码 Yjs 状态
    const state = Y.encodeStateAsUpdate(ydoc);
    
    // 4. 转换为 ProseMirror JSON
    const content = yDocToProsemirrorJSON(ydoc, "default") as ProsemirrorData;
    
    // 5. 检查是否有实际变更
    const isUnchanged = isEqual(document.content, content);
    if (isUnchanged) {
      return;  // 无变更，跳过
    }
    
    // 6. 提取协作者信息
    const pud = new Y.PermanentUserData(ydoc);
    const pudIds = Array.from(pud.clients.values());
    const collaboratorIds = uniq([
      ...document.collaboratorIds,
      ...sessionCollaboratorIds,
      ...pudIds,
    ]);
    
    // 7. 确定编辑器版本
    const editorVersion = // 取客户端和服务端版本中较新的
    
    // 8. 更新数据库
    await document.update(
      {
        content,                    // ProseMirror 快照
        state: Buffer.from(state),  // Yjs 二进制状态
        lastModifiedById,
        collaboratorIds,
        editorVersion,
      },
      {
        transaction,
        hooks: false,  // 禁用钩子防止无限循环
      }
    );
    
    // 9. 触发更新事件
    await Event.schedule({
      name: "documents.update",
      documentId: document.id,
      collectionId: document.collectionId,
      teamId: document.teamId,
      actorId: lastModifiedById,
      authType: AuthenticationType.APP,
      data: {
        multiplayer: true,
        title: document.title,
        done: isLastConnection,  // 标记会话结束
      },
    });
  });
}
```

### 4.5 API 变更同步：APIUpdateExtension

`server/collaboration/APIUpdateExtension.ts` 处理非协作方式的文档更新。

#### 4.5.1 问题场景

当通过 REST API 更新文档时（如 `POST /api/documents.update`），协作服务内存中的 Y.Doc 状态需要同步：

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    API 更新与协作状态同步                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  场景: 用户 A 通过 REST API 更新文档，用户 B 正在通过协作编辑同一文档          │
│                                                                           │
│  传统问题:                                                                 │
│  - API 直接更新数据库                                                       │
│  - 协作服务内存中的 Y.Doc 仍为旧版本                                        │
│  - 用户 B 的编辑可能覆盖 API 变更                                           │
│                                                                           │
│  Outline 解决方案:                                                         │
│  - API 更新后发布 Redis 消息                                                │
│  - 协作服务订阅并同步内存状态                                                │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 4.5.2 实现机制

```typescript
// 1. API 更新时发布通知
// Document model 或 Command 中调用:
await APIUpdateExtension.notifyUpdate(documentId, actorId);

// 2. 协作服务订阅 Redis 频道
async onConfigure() {
  this.subscriber = new RedisAdapter(/* ... */);
  await this.subscriber.psubscribe(`${CHANNEL_PREFIX}:*`, ...);
  this.subscriber.on("pmessage", this.handleMessage);
}

// 3. 处理更新消息
private handleMessage = async (pattern, channel, message) => {
  const documentId = channel.replace(`${CHANNEL_PREFIX}:`, "");
  const document = this.documents.get(documentId);  // 内存中的 Y.Doc
  
  if (!document) return;  // 该实例无此文档，忽略
  
  // 从数据库加载最新状态
  const dbDocument = await Document.unscoped().findOne({
    attributes: ["state", "content", "text"],
    where: { id: documentId },
  });
  
  // 计算差异并应用
  const dbYdoc = new Y.Doc();
  Y.applyUpdate(dbYdoc, dbDocument.state);
  
  const currentStateVector = Y.encodeStateVector(document);
  const update = Y.encodeStateAsUpdate(dbYdoc, currentStateVector);
  
  if (update.length > 0) {
    Y.applyUpdate(document, update);  // 应用到内存中的文档
    // Hocuspocus 会自动广播到所有连接的客户端
  }
};
```

---

## 5. 数据库存储模型

### 5.1 Document 表核心字段

`server/models/Document.ts`

```typescript
@Table({ tableName: "documents" })
class Document extends ArchivableModel {
  
  /**
   * Yjs 协作状态 (二进制 BLOB)
   * 这是 CRDT 操作的权威来源
   */
  @Column(DataType.BLOB)
  state?: Uint8Array | null;
  
  /**
   * ProseMirror 内容快照 (JSONB)
   * 用于快速读取、导出、搜索索引等场景
   * 由 state 派生，可能稍滞后
   */
  @Column(DataType.JSONB)
  content: ProsemirrorData | null;
  
  /**
   * Markdown 文本 (已废弃，但仍兼容)
   * @deprecated Use content 或 DocumentHelper.toMarkdown
   */
  @Column(DataType.TEXT)
  text: string;
  
  /**
   * 协作者用户 ID 列表
   * 合并自: 会话协作者 + PermanentUserData
   */
  @Column(DataType.ARRAY(DataType.UUID))
  collaboratorIds: string[] = [];
  
  /**
   * 最后编辑该文档的编辑器版本
   * 用于版本兼容性检查
   */
  @Column
  editorVersion: string | null;
  
  // ... 其他字段
}
```

### 5.2 三种存储格式的关系

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      数据格式层级关系                                       │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│                         ┌─────────────────────┐                          │
│                         │    state (BLOB)     │                          │
│                         │   Yjs CRDT 二进制   │                          │
│                         │    (权威数据源)      │                          │
│                         └──────────┬──────────┘                          │
│                                    │                                        │
│                    持久化时同步派生 │                                        │
│                                    ▼                                        │
│                         ┌─────────────────────┐                          │
│                         │   content (JSONB)   │                          │
│                         │  ProseMirror 快照   │                          │
│                         │   (用于快速读取)      │                          │
│                         └──────────┬──────────┘                          │
│                                    │                                        │
│                         可导出为   │                                        │
│                                    ▼                                        │
│                         ┌─────────────────────┐                          │
│                         │     text (TEXT)     │                          │
│                         │   Markdown 格式      │                          │
│                         │    (已废弃)          │                          │
│                         └─────────────────────┘                          │
│                                                                           │
│  读取优先级:                                                               │
│  1. 优先使用 content (JSONB) - 最快                                        │
│  2. 回退到 state (BLOB) 反序列化 - 最权威                                  │
│  3. 最后回退到 text (Markdown) - 兼容旧文档                                │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 5.3 DocumentHelper 数据转换

`server/models/helpers/DocumentHelper.tsx` 提供统一的读取接口。

```typescript
static toProsemirror(document: Document | Revision | Collection | ProsemirrorData) {
  // 1. 直接是 ProseMirrorData
  if ("type" in document && document.type === "doc") {
    return Node.fromJSON(schema, document);
  }
  
  // 2. 优先使用 content (JSONB)
  if ("content" in document && document.content) {
    return Node.fromJSON(schema, document.content);
  }
  
  // 3. 回退到 state (BLOB) 反序列化
  if ("state" in document && document.state) {
    const ydoc = new Y.Doc();
    Y.applyUpdate(ydoc, document.state);
    return Node.fromJSON(schema, yDocToProsemirrorJSON(ydoc, "default"));
  }
  
  // 4. 最后回退到 text (Markdown)
  const text = document instanceof Collection ? document.description : document.text;
  return parser.parse(text ?? "") || Node.fromJSON(schema, {});
}
```

---

## 6. 多实例部署与 Redis 同步

### 6.1 架构说明

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        多实例部署架构                                       │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│                         ┌──────────────────┐                              │
│                         │   Load Balancer  │                              │
│                         └────────┬─────────┘                              │
│                                  │                                         │
│              ┌───────────────────┼───────────────────┐                    │
│              │                   │                   │                    │
│              ▼                   ▼                   ▼                    │
│     ┌────────────────┐  ┌────────────────┐  ┌────────────────┐          │
│     │  Hocuspocus    │  │  Hocuspocus    │  │  Hocuspocus    │          │
│     │   Instance A   │  │   Instance B   │  │   Instance C   │          │
│     │                │  │                │  │                │          │
│     │ RedisExtension │  │ RedisExtension │  │ RedisExtension │          │
│     └───────┬────────┘  └───────┬────────┘  └───────┬────────┘          │
│             │                    │                    │                    │
│             └────────────────────┼────────────────────┘                    │
│                                  │                                         │
│                                  ▼                                         │
│                     ┌──────────────────────┐                               │
│                     │  Redis Cluster       │                               │
│                     │  (Pub/Sub + 状态)    │                               │
│                     └──────────────────────┘                               │
│                                                                           │
│  工作原理:                                                                 │
│  - RedisExtension 使用 Redis Pub/Sub 广播更新                              │
│  - 各实例订阅相关频道，同步内存中的 Y.Doc 状态                               │
│  - 客户端可连接到任意实例，获得一致的协作体验                                  │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 6.2 配置代码

```typescript
// server/services/collaboration.ts

extensions: [
  ...(env.REDIS_COLLABORATION_URL
    ? [
        new Redis({
          redis: RedisAdapter.collaborationClient,
        }),
      ]
    : []),
  // ... 其他扩展
]
```

---

## 7. 两种 WebSocket 服务的区别

Outline 实现了两套独立的 WebSocket 服务：

| 特性 | 协作编辑服务 | 实时事件服务 |
|------|-------------|-------------|
| **路径** | `/collaboration` | `/realtime` |
| **技术** | Hocuspocus (原生 WebSocket) | Socket.io |
| **用途** | Yjs CRDT 文档同步 | 实体变更通知 |
| **数据格式** | Yjs 二进制更新 | JSON 事件 |
| **示例事件** | 文档内容变更、光标位置 | documents.update, collections.create |
| **扩展机制** | Hocuspocus Extensions | Socket.io Event Handlers |

### 7.1 实时事件服务 (Websockets)

`server/services/websockets.ts` 处理非协作的实时通知：

```typescript
// 订阅的房间:
// - team-${teamId}        团队级事件
// - user-${userId}        用户级事件
// - collection-${colId}   集合级事件
// - group-${groupId}      用户组级事件

// 处理的事件类型:
// - documents.update / archive / delete
// - collections.create / update / delete
// - comments.create / update / delete
// - users.update / demote
// - notifications.create
// - pins.create / delete
// - stars.create / delete
// - fileOperations.update
// - imports.update
```

---

## 8. 关键设计要点与最佳实践

### 8.1 并发控制

| 场景 | 解决方案 | 代码位置 |
|------|----------|----------|
| 文档初始化竞态 | 行级锁 + 双重检查 | PersistenceExtension.onLoadDocument |
| 持久化写入竞态 | 行级锁 + 15秒超时 | documentCollaborativeUpdater |
| 多实例状态同步 | Redis Extension (Pub/Sub) | collaboration.ts |

### 8.2 性能优化

| 优化点 | 策略 | 说明 |
|--------|------|------|
| 读取性能 | 三级缓存 | IndexedDB → 内存 Y.Doc → PostgreSQL |
| 写入性能 | 防抖 + 批量 | debounce 3s, maxDebounce 10s |
| 连接管理 | 空闲断开 | 页面不可见 + 空闲时自动断开 |
| 数据库查询 | 按需加载 | 默认不加载 state 大字段 |

### 8.3 容错处理

| 故障场景 | 处理机制 |
|----------|----------|
| 网络中断 | Hocuspocus Provider 自动重连 + 指数退避 |
| 令牌过期 | authenticationFailed 事件触发刷新 |
| 编辑器版本不兼容 | EditorVersionExtension 检测并断开 |
| 数据库锁超时 | 15秒 lock_timeout 防止死锁 |
| Yjs 编码错误 | 全局错误监听，提示用户刷新 |

### 8.4 数据一致性保证

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      一致性保证机制                                         │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. CRDT 层面:                                                             │
│     - Yjs 内置保证: 操作可交换、可结合、可幂等                               │
│     - 任意顺序应用最终一致                                                  │
│                                                                           │
│  2. 存储层面:                                                              │
│     - 单文档写入使用行级锁                                                 │
│     - 事务内原子更新 state + content                                        │
│     - hooks: false 防止触发后处理导致循环                                  │
│                                                                           │
│  3. API 与协作同步:                                                        │
│     - APIUpdateExtension 监听 Redis 消息                                   │
│     - 使用 Y.encodeStateAsUpdate(ydoc, stateVector) 计算增量              │
│     - 安全合并到内存中的协作状态                                            │
│                                                                           │
│  4. 多实例同步:                                                            │
│     - RedisExtension 基于 Redis Pub/Sub                                    │
│     - 各实例间广播 Yjs 更新                                                 │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 9. 代码引用速查

### 9.1 前端关键文件

| 文件路径 | 职责 |
|----------|------|
| `app/scenes/Document/components/MultiplayerEditor.tsx` | 协作编辑器入口，Provider 初始化 |
| `app/editor/extensions/Multiplayer.ts` | Yjs 与 ProseMirror 集成 |
| `app/components/WebsocketProvider.tsx` | Socket.io 实时事件连接 (非协作) |

### 9.2 服务端关键文件

| 文件路径 | 职责 |
|----------|------|
| `server/services/collaboration.ts` | Hocuspocus 服务配置与启动 |
| `server/collaboration/PersistenceExtension.ts` | 文档加载与持久化核心 |
| `server/collaboration/AuthenticationExtension.ts` | 连接认证与权限检查 |
| `server/collaboration/APIUpdateExtension.ts` | API 变更同步到协作状态 |
| `server/commands/documentCollaborativeUpdater.ts` | 数据库事务更新 |
| `server/services/websockets.ts` | Socket.io 实时事件服务 |

### 9.3 数据模型与辅助类

| 文件路径 | 职责 |
|----------|------|
| `server/models/Document.ts` | 文档数据模型 |
| `server/models/helpers/DocumentHelper.tsx` | 文档格式转换工具 |
| `server/collaboration/types.ts` | 协作相关类型定义 |

---

## 10. 总结

Outline 的 Yjs CRDT 实时协作系统是一个设计精良的架构：

### 10.1 架构优势

1. **分层清晰**：前端编辑器 → Yjs CRDT → Hocuspocus 同步 → 服务端扩展 → 数据库持久化
2. **双重存储**：Yjs 二进制 state (权威) + ProseMirror JSON content (快速读取)
3. **离线优先**：IndexedDB 本地缓存 + 自动重连机制
4. **灵活扩展**：Hocuspocus Extension 机制实现认证、限流、日志等横切关注点

### 10.2 关键技术决策

| 决策点 | 选择 | 理由 |
|--------|------|------|
| CRDT 库 | Yjs | 成熟稳定，生态完善 (y-prosemirror, y-indexeddb) |
| 同步框架 | Hocuspocus | 专为 Yjs 设计，提供完善的服务端能力 |
| 数据库 | PostgreSQL | 支持 JSONB (content) + BLOB (state) |
| 多实例 | Redis Extension | 利用 Redis Pub/Sub 实现状态同步 |

### 10.3 数据流总结

```
用户编辑
    │
    ▼
ProseMirror Transaction
    │
    ▼
y-prosemirror 转换为 Yjs Operation
    │
    ▼
Y.Doc 生成 Update (本地应用)
    │
    ├─────────────────┬─────────────────┐
    ▼                 ▼                 ▼
IndexedDB 保存   Awareness 广播   Hocuspocus Provider 发送
    │                 光标位置          │
    │                                   ▼
    │                            WebSocket 到服务端
    │                                   │
    │                                   ▼
    │                            Hocuspocus Server 接收
    │                                   │
    │                                   ▼
    │                            内存 Y.Doc 应用更新
    │                                   │
    │                                   ▼
    │                            广播到其他连接客户端
    │                                   │
    │              ┌────────────────────┘
    │              ▼
    │         debounce 3秒后
    │              │
    │              ▼
    │         PersistenceExtension.onStoreDocument
    │              │
    │              ▼
    │         documentCollaborativeUpdater Command
    │              │
    │              ▼
    │         PostgreSQL 事务更新
    │              │
    │              ├── state (BLOB) - Yjs 二进制
    │              ├── content (JSONB) - ProseMirror 快照
    │              └── collaboratorIds - 协作者列表
    │
    ▼
Event.schedule → documents.update 事件
    │
    ▼
Websockets 服务广播 → 其他客户端 UI 更新
```

---

*报告生成时间: 2026-05-05*
*分析基于: Outline 代码库当前版本*
