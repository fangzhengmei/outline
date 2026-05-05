# Outline 文档版本历史与撤销协作机制分析

## 1. 概述

Outline 是一个基于 Y.js CRDT（Conflict-free Replicated Data Type）的实时协作知识库系统。本文档深入分析其版本历史记录机制以及多人协作场景下撤销操作如何跨 CRDT 和版本快照两个机制进行协作。

## 2. 版本历史记录机制

### 2.1 数据模型设计

#### Revision 模型

版本历史记录通过 `Revision` 模型存储，核心字段包括：

| 字段 | 类型 | 描述 |
|------|------|------|
| `version` | SMALLINT | 文档版本号 |
| `editorVersion` | VARCHAR | 编辑器版本 |
| `title` | VARCHAR | 文档标题快照 |
| `name` | VARCHAR | 版本名称（可选） |
| `content` | JSONB | ProseMirror 格式的内容快照 |
| `text` | TEXT | Markdown 格式内容（已弃用） |
| `icon` | VARCHAR | 文档图标 |
| `color` | VARCHAR | 图标颜色 |
| `documentId` | UUID | 关联文档 ID |
| `userId` | UUID | 创建者 ID |
| `collaboratorIds` | UUID[] | 协作者 ID 数组 |

**关键代码位置**：`server/models/Revision.ts:36-118`

#### Document 模型中的版本相关字段

Document 模型维护以下与版本相关的状态：

| 字段 | 类型 | 描述 |
|------|------|------|
| `version` | SMALLINT | 文档格式版本 |
| `revisionCount` | INTEGER | 历史版本数量 |
| `content` | JSONB | 当前内容快照（ProseMirror） |
| `state` | BLOB | Y.js CRDT 二进制状态 |
| `collaboratorIds` | UUID[] | 协作者列表 |

**关键代码位置**：`server/models/Document.ts:304-366`

### 2.2 版本创建流程

版本创建采用**异步队列处理**机制，避免阻塞主操作流程。

#### 触发时机

版本创建由以下事件触发：
- `documents.publish` - 文档发布
- `documents.update` - 文档更新（需 `done` 标志）
- `documents.update.debounced` - 防抖更新

**关键代码位置**：`server/queues/processors/RevisionsProcessor.ts:9-14`

#### 处理流程

```
1. 事件触发 → RevisionsProcessor.perform()
      ↓
2. 从 Redis 获取当前会话协作者 ID
      ↓
3. 获取最新文档和前一版本
      ↓
4. 比较 content 和 title（使用 fast-deep-equal）
      ↓
5. 内容不同 → 调用 revisionCreator 创建新版本
      ↓
6. 内容相同 → 跳过版本创建
```

**关键代码位置**：`server/queues/processors/RevisionsProcessor.ts:16-64`

#### 去重机制

在创建新版本前，系统会执行内容比对：

```typescript
const previous = await Revision.findLatest(document.id);

if (
  previous &&
  isEqual(document.content, previous.content) &&
  document.title === previous.title
) {
  return; // 内容相同，跳过
}
```

**关键代码位置**：`server/queues/processors/RevisionsProcessor.ts:38-44`

#### 协作者追踪

系统使用 Redis 实时追踪当前编辑会话的协作者：

```typescript
// 键格式: collaborators:{documentId}
const key = Document.getCollaboratorKey(event.documentId);
const collaboratorIds = await Redis.defaultClient.smembers(key);
await Redis.defaultClient.del(key); // 读取后清空
```

**关键代码位置**：`server/queues/processors/RevisionsProcessor.ts:26-28`

当用户在协作中产生变更时，`PersistenceExtension.onChange` 会将用户 ID 添加到 Redis 集合：

```typescript
async onChange({ context, documentName }: withContext<onChangePayload>) {
  const [, documentId] = documentName.split(".");
  if (context.user) {
    const key = Document.getCollaboratorKey(documentId);
    await Redis.defaultClient.sadd(key, context.user.id);
  }
}
```

**关键代码位置**：`server/collaboration/PersistenceExtension.ts:98-110`

### 2.3 版本快照构建

`Revision.createFromDocument` 方法从当前文档构建版本快照：

```typescript
static buildFromDocument(document: Document) {
  return this.build({
    title: document.title,
    icon: document.icon,
    color: document.color,
    content: document.content,  // 复制 content 快照
    userId: document.lastModifiedById,
    editorVersion: document.editorVersion,
    version: document.version,
    documentId: document.id,
    createdAt: document.updatedAt,  // 使用文档更新时间
  });
}
```

**关键代码位置**：`server/models/Revision.ts:174-188`

### 2.4 版本创建时序图

```
┌─────────────┐     ┌─────────────────┐     ┌──────────────────────┐
│   Client    │     │  Collaboration  │     │  DocumentUpdater     │
│  (Editor)   │     │    Server       │     │   (Command)          │
└──────┬──────┘     └────────┬────────┘     └──────────┬───────────┘
       │                     │                          │
       │──编辑操作──────────>│                          │
       │                     │──onChange()─────────────>│
       │                     │   Redis.sadd(userId)    │
       │                     │                          │
       │──持续编辑──────────>│                          │
       │                     │                          │
       │──断开连接──────────>│                          │
       │                     │                          │
       │                     │──onStoreDocument()──────>│
       │                     │   documentCollaborativeUpdater()
       │                     │                          │
       │                     │                          │──更新 content, state
       │                     │                          │   触发 documents.update
       │                     │                          │
       │                     │     ┌────────────────────┴──────────┐
       │                     │     │      RevisionsProcessor         │
       │                     │     │  (异步队列处理)                 │
       │                     │     └────────────────────┬──────────┘
       │                     │                          │
       │                     │                          │──Redis.smembers(collaborators)
       │                     │                          │
       │                     │                          │──比较 content vs previous.content
       │                     │                          │
       │                     │                          │──revisionCreator()
       │                     │                          │   插入 Revision 记录
       │                     │                          │
```

## 3. CRDT (Y.js) 集成实现

### 3.1 技术选型

Outline 使用以下技术栈实现实时协作：

| 组件 | 库/框架 | 用途 |
|------|---------|------|
| CRDT 引擎 | Y.js | 无冲突数据类型核心 |
| 同步协议 | Hocuspocus | WebSocket 同步服务器 |
| ProseMirror 桥接 | y-prosemirror | Y.js ↔ ProseMirror 转换 |
| 本地持久化 | y-indexeddb | 浏览器本地缓存 |

### 3.2 前端集成

#### Multiplayer 编辑器扩展

`Multiplayer` 扩展是前端协作编辑的核心：

```typescript
get plugins() {
  const type = doc.get("default", Y.XmlFragment);
  return [
    ySyncPlugin(type),           // Y.js ↔ ProseMirror 同步
    yCursorPlugin(provider.awareness, { ... }),  // 光标同步
    yUndoPlugin(),               // 撤销/重做支持
    new Plugin({
      props: {
        // 远程变更时自动滚动到选区
        handleScrollToSelection: (view) => isRemoteTransaction(view.state.tr),
      },
    }),
  ];
}
```

**关键代码位置**：`app/editor/extensions/Multiplayer.ts:45-131`

#### 撤销/重做命令

协作模式下，撤销操作直接调用 Y.js 的 undo/redo：

```typescript
commands() {
  return {
    undo: () => undo,  // y-prosemirror 的 undo
    redo: () => redo,  // y-prosemirror 的 redo
  };
}
```

**关键代码位置**：`app/editor/extensions/Multiplayer.ts:133-138`

#### 非协作模式的 History 扩展

当不处于协作模式时，使用传统的 ProseMirror 历史：

```typescript
get plugins() {
  return [history()];  // prosemirror-history
}

keys() {
  return {
    "Mod-z": () => this.editor.commands.undo(),
    "Mod-y": () => this.editor.commands.redo(),
    "Shift-Mod-z": () => this.editor.commands.redo(),
  };
}
```

**关键代码位置**：`shared/editor/extensions/History.ts:19-26`

### 3.3 后端服务架构

#### Hocuspocus 扩展配置

协作服务器通过一系列扩展实现功能：

```typescript
const hocuspocus = Server.configure({
  debounce: 3000,        // 防抖时间
  timeout: 30000,        // 超时时间
  maxDebounce: 10000,    // 最大防抖
  extensions: [
    new Redis({ ... }),              // Redis 多实例同步
    new Throttle({ ... }),           // 限流
    new ConnectionLimitExtension(),  // 连接数限制
    new EditorVersionExtension(),    // 编辑器版本校验
    new AuthenticationExtension(),   // 认证
    new PersistenceExtension(),      // 持久化（核心）
    new APIUpdateExtension(),        // API 更新同步
    new ViewsExtension(),            // 视图更新
    new LoggerExtension(),           // 日志
    new MetricsExtension(),          // 指标
  ],
});
```

**关键代码位置**：`server/services/collaboration.ts:44-71`

#### PersistenceExtension - 持久化核心

该扩展负责 Y.js 状态与数据库的双向同步：

##### 文档加载流程

```typescript
async onLoadDocument({ documentName, ...data }: withContext<onLoadDocumentPayload>) {
  const [, documentId] = documentName.split(".");
  const fieldName = "default";

  // 1. 检查内存中是否已存在
  if (!data.document.isEmpty(fieldName)) {
    return;
  }

  // 2. 尝试从 state 字段加载（Y.js 二进制）
  const documentWithoutLock = await Document.unscoped().findOne({
    attributes: ["state"],
    where: { id: documentId },
  });

  if (documentWithoutLock.state) {
    const ydoc = new Y.Doc();
    Y.applyUpdate(ydoc, documentWithoutLock.state);
    return ydoc;
  }

  // 3. 回退：从 content 或 text 重建 Y.js 状态
  return await sequelize.transaction(async (transaction) => {
    const document = await Document.unscoped().findOne({
      attributes: ["id", "state", "content", "text"],
      transaction,
      lock: transaction.LOCK.UPDATE,
      // ...
    });

    let ydoc;
    if (document.content) {
      ydoc = ProsemirrorHelper.toYDoc(document.content, fieldName);
    } else {
      ydoc = ProsemirrorHelper.toYDoc(document.text, fieldName);
    }

    // 保存重建的 state
    const state = ProsemirrorHelper.toState(ydoc);
    await document.update({ state }, { silent: true, hooks: false, transaction });
    return ydoc;
  });
}
```

**关键代码位置**：`server/collaboration/PersistenceExtension.ts:19-96`

##### 文档持久化流程

```typescript
async onStoreDocument({
  document, context, documentName, clientsCount, requestParameters,
}: onStoreDocumentPayload) {
  const [, documentId] = documentName.split(".");
  const clientVersion = requestParameters.get("editorVersion");

  const key = Document.getCollaboratorKey(documentId);
  const sessionCollaboratorIds = await Redis.defaultClient.smembers(key);

  if (!sessionCollaboratorIds || sessionCollaboratorIds.length === 0) {
    return;  // 无变更，跳过
  }

  await documentCollaborativeUpdater({
    documentId,
    ydoc: document,
    sessionCollaboratorIds,
    isLastConnection: clientsCount === 0,
    clientVersion,
  });
}
```

**关键代码位置**：`server/collaboration/PersistenceExtension.ts:112-143`

### 3.4 documentCollaborativeUpdater 命令

该命令将 Y.js 状态转换并持久化到数据库：

```typescript
export default async function documentCollaborativeUpdater({
  documentId, ydoc, sessionCollaboratorIds, isLastConnection, clientVersion,
}: Props) {
  return sequelize.transaction(async (transaction) => {
    const document = await Document.unscoped()
      .scope("withoutState")
      .findOne({
        where: { id: documentId },
        transaction,
        lock: { of: Document, level: transaction.LOCK.UPDATE },
        // ...
      });

    // 1. 编码 Y.js 状态
    const state = Y.encodeStateAsUpdate(ydoc);
    
    // 2. 转换为 ProseMirror 快照
    const content = yDocToProsemirrorJSON(ydoc, "default") as ProsemirrorData;
    
    // 3. 检查是否有实际变化
    const isUnchanged = isEqual(document.content, content);
    if (isUnchanged) {
      return;
    }

    // 4. 提取协作者（从 Y.PermanentUserData）
    const pud = new Y.PermanentUserData(ydoc);
    const pudIds = Array.from(pud.clients.values());
    const collaboratorIds = uniq([
      ...document.collaboratorIds,
      ...sessionCollaboratorIds,
      ...pudIds,
    ]);

    // 5. 更新文档
    await document.update(
      {
        content,
        state: Buffer.from(state),
        lastModifiedById,
        collaboratorIds,
        editorVersion,
      },
      {
        transaction,
        hooks: false,  // 避免触发 AfterUpdate 导致无限循环
      }
    );

    // 6. 触发更新事件
    await Event.schedule({
      name: "documents.update",
      documentId: document.id,
      data: {
        multiplayer: true,
        title: document.title,
        done: isLastConnection,  // 标记是否所有连接都已断开
      },
    });
  });
}
```

**关键代码位置**：`server/commands/documentCollaborativeUpdater.ts:25-119`

## 4. 版本快照机制详解

### 4.1 双存储架构

Outline 采用**双存储**架构来兼顾协作效率和版本管理：

```
┌─────────────────────────────────────────────────────────────┐
│                      Document 模型                           │
├─────────────────────────┬───────────────────────────────────┤
│      content (JSONB)    │         state (BLOB)              │
├─────────────────────────┼───────────────────────────────────┤
│  ProseMirror 快照格式    │    Y.js CRDT 二进制状态           │
│                         │                                   │
│  • 人类可读              │    • 协作编辑核心                │
│  • 版本比对基础          │    • 实时同步必需                │
│  • 历史恢复源            │    • 包含完整操作历史            │
│  • 搜索引擎索引          │    • 可增量更新                  │
│                         │                                   │
│  来源:                   │    来源:                         │
│   - yDocToProsemirrorJSON │   - Y.encodeStateAsUpdate()    │
│                         │                                   │
│  更新时机:               │    更新时机:                     │
│   - 协作断开时           │    - 每次协作变更时              │
│   - API 更新时           │    - 文档加载重建时              │
└─────────────────────────┴───────────────────────────────────┘
```

### 4.2 快照转换流程

#### Y.js → ProseMirror

```typescript
// 在 documentCollaborativeUpdater 中
const content = yDocToProsemirrorJSON(ydoc, "default") as ProsemirrorData;
```

**关键代码位置**：`server/commands/documentCollaborativeUpdater.ts:53`

#### ProseMirror → Y.js（重建）

```typescript
// 在 PersistenceExtension.onLoadDocument 中
if (document.content) {
  ydoc = ProsemirrorHelper.toYDoc(document.content, fieldName);
} else {
  ydoc = ProsemirrorHelper.toYDoc(document.text, fieldName);
}
```

**关键代码位置**：`server/collaboration/PersistenceExtension.ts:70-82`

### 4.3 版本快照与 CRDT 状态的关系

| 场景 | content (快照) | state (CRDT) | 说明 |
|------|----------------|--------------|------|
| 新文档创建 | ✅ 初始值 | ❌ 空 | state 从 content 重建 |
| 协作编辑中 | ⚠️ 可能过期 | ✅ 最新 | state 是真相源 |
| 协作断开后 | ✅ 同步更新 | ✅ 持久化 | 两者保持一致 |
| 版本恢复时 | ✅ 从 Revision 复制 | ⚠️ 需重新同步 | APIUpdateExtension 处理 |

## 5. 撤销操作协作机制

Outline 支持**两种撤销模式**，分别应对不同的协作场景：

### 5.1 撤销模式概览

| 模式 | 实现 | 适用场景 | 协作感知 |
|------|------|----------|----------|
| **本地撤销** | yUndoPlugin (Y.js) | 实时协作编辑 | ✅ 感知远程变更 |
| **版本恢复** | Revision 快照恢复 | 恢复到历史版本 | ⚠️ 需同步到协作客户端 |

### 5.2 本地撤销/重做机制

#### yUndoPlugin 工作原理

`yUndoPlugin` 是 y-prosemirror 提供的撤销插件，核心特性：

1. **忽略远程事务**：只撤销本地用户的操作，不会撤销其他协作者的变更
2. **堆栈隔离**：每个客户端维护独立的撤销堆栈
3. **操作追踪**：通过 Y.Transaction 的 origin 标识区分操作来源

#### 远程事务识别

系统通过 `isRemoteTransaction` 函数判断事务来源：

```typescript
export function isRemoteTransaction(tr: Transaction): boolean {
  const meta = tr.getMeta(ySyncPluginKey);
  // 注意：逻辑看似翻转但实际正确
  // isChangeOrigin 为 true 表示变更来自远程同步
  return !!meta?.isChangeOrigin;
}
```

**关键代码位置**：`shared/editor/lib/multiplayer.ts:12-17`

#### 远程事务的处理策略

在多个扩展中，远程事务会触发特殊处理：

1. **ToggleBlock 节点**：忽略远程变更的状态更新
   ```typescript
   if (isRemoteTransaction(tr)) {
     return;  // 远程变更不触发本地状态更新
   }
   ```
   **关键代码位置**：`shared/editor/nodes/ToggleBlock.ts:175`

2. **Mermaid 扩展**：远程变更跳过重新渲染
   ```typescript
   if (isRemoteTransaction(transaction)) {
     // 远程变更直接使用新状态
     this.map = map;
     return;
   }
   ```
   **关键代码位置**：`shared/editor/extensions/Mermaid.ts:442`

3. **自动滚动控制**：
   ```typescript
   handleScrollToSelection: (view) => isRemoteTransaction(view.state.tr),
   ```
   远程变更时自动滚动到选区位置
   **关键代码位置**：`app/editor/extensions/Multiplayer.ts:127`

### 5.3 版本恢复机制（跨协作同步）

当用户需要恢复到历史版本时，系统通过 `documents.restore` API 端点处理。

#### 恢复流程

```
1. 用户选择历史版本 → 调用 restoreRevision action
      ↓
2. 前端导航到文档页面，携带 restore 和 revisionId 参数
      ↓
3. 后端处理: documents.restore 端点
      ↓
4. 调用 document.restoreFromRevision(revision)
      ↓
5. 更新文档 content, title, icon, color
      ↓
6. 保存并触发 "restore" 事件
      ↓
7. APIUpdateExtension 广播变更到所有协作客户端
```

#### 核心实现代码

##### 前端触发

```typescript
export const restoreRevision = createAction({
  name: ({ t }) => t("Restore"),
  perform: async ({ event, location, activeDocumentId }) => {
    event?.preventDefault();
    
    const match = matchPath<{ revisionId: string }>(location.pathname, {
      path: matchDocumentHistory,
    });
    const revisionId = match?.params.revisionId;
    const document = stores.documents.get(activeDocumentId);
    
    // 导航时传递恢复参数
    history.push(document.url, {
      restore: true,
      revisionId,
    });
  },
});
```

**关键代码位置**：`app/actions/definitions/revisions.tsx:17-45`

##### 后端恢复

```typescript
// documents.restore 端点中的版本恢复逻辑
else if (revisionId) {
  // restore a document to a specific revision
  authorize(user, "update", document);
  const revision = await Revision.findByPk(revisionId, { transaction });
  authorize(document, "restore", revision);

  await document.restoreFromRevision(revision);
  await document.saveWithCtx(ctx, undefined, { name: "restore" });
}
```

**关键代码位置**：`server/routes/api/documents/documents.ts:1006-1013`

##### restoreFromRevision 方法

```typescript
restoreFromRevision = async (revision: Revision) => {
  if (revision.documentId !== this.id) {
    throw new Error("Revision does not belong to this document");
  }

  // 直接复制快照字段
  this.content = revision.content;
  this.text = await DocumentHelper.toMarkdown(revision, {
    includeTitle: false,
  });
  this.title = revision.title;
  this.icon = revision.icon;
  this.color = revision.color;
};
```

**关键代码位置**：`server/models/Document.ts:903-915`

#### 协作同步机制（APIUpdateExtension）

当通过 API 更新文档（包括版本恢复）时，需要将变更同步到所有协作客户端。`APIUpdateExtension` 负责此功能。

##### 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Update Flow                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────┐      ┌──────────────────┐      ┌──────────────┐  │
│  │ API Call │ ───> │ Document.update  │ ───> │ Redis Pub/Sub│  │
│  │ (Restore)│      │  (saveWithCtx)   │      │  Publish     │  │
│  └──────────┘      └──────────────────┘      └──────┬───────┘  │
│                                                        │          │
│                             ┌──────────────────────────┘          │
│                             ▼                                     │
│              ┌──────────────────────────────┐                    │
│              │  APIUpdateExtension (Server) │                    │
│              │  - Redis Subscriber           │                    │
│              │  - 监听 channel:update 消息   │                    │
│              └───────────────┬──────────────┘                    │
│                              │                                    │
│                              ▼                                    │
│              ┌──────────────────────────────┐                    │
│              │  1. 从数据库读取最新 state    │                    │
│              │  2. 计算与内存 state 的 diff   │                    │
│              │  3. Y.applyUpdate() 到内存    │                    │
│              │  4. Hocuspocus 广播到所有客户端│                    │
│              └──────────────────────────────┘                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

##### 核心实现

###### 订阅与监听

```typescript
// 配置时建立 Redis 订阅
async onConfigure(_data: onConfigurePayload): Promise<void> {
  this.subscriber = new RedisAdapter(env.REDIS_COLLABORATION_URL || env.REDIS_URL, {
    connectionNameSuffix: "collab-api-updates",
  });

  // 订阅频道模式: collaboration:api-update:*
  await this.subscriber.psubscribe(`${CHANNEL_PREFIX}:*`, (err) => {
    // ...
  });

  // 处理消息
  this.subscriber.on("pmessage", this.handleMessage);
}
```

**关键代码位置**：`server/collaboration/APIUpdateExtension.ts:47-90`

###### 消息处理与同步

```typescript
private handleMessage = async (
  _pattern: string,
  channel: string,
  message: string
): Promise<void> => {
  const documentId = channel.replace(`${CHANNEL_PREFIX}:`, "");
  const document = this.documents.get(documentId);

  if (!document) {
    return;  // 该实例未加载此文档
  }

  const data = JSON.parse(message);

  // 1. 从数据库获取最新状态
  const dbDocument = await Document.unscoped().findOne({
    attributes: ["state", "content", "text"],
    where: { id: documentId },
  });

  if (!dbDocument.state) {
    return;
  }

  // 2. 重建数据库中的 Y.Doc
  const dbYdoc = new Y.Doc();
  Y.applyUpdate(dbYdoc, dbDocument.state);

  // 3. 计算增量更新
  const currentStateVector = Y.encodeStateVector(document);
  const update = Y.encodeStateAsUpdate(dbYdoc, currentStateVector);

  // 4. 应用到内存中的协作文档
  if (update.length > 0) {
    Y.applyUpdate(document, update);
    // Hocuspocus 会自动广播到所有连接的客户端
  }

  dbYdoc.destroy();
};
```

**关键代码位置**：`server/collaboration/APIUpdateExtension.ts:125-184`

###### 触发通知点

在 `Document.saveWithCtx` 后，如果 `state` 字段改变，会触发通知：

```typescript
// Document 模型的 AfterUpdate hook
static notifyCollaborationServer(model: Document, ctx: HookContext) {
  if (model.changed("state") && ctx.auth?.user?.id) {
    const actorId = ctx.auth.user.id;
    const notify = async () => {
      await APIUpdateExtension.notifyUpdate(model.id, actorId);
    };

    if (ctx.transaction) {
      const transaction = ctx.transaction.parent || ctx.transaction;
      transaction.afterCommit(notify);
    } else {
      void notify();
    }
  }
}
```

**关键代码位置**：`server/models/Document.ts:586-600`

###### 通知发布

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

  await RedisAdapter.defaultClient.publish(channel, message);
}
```

**关键代码位置**：`server/collaboration/APIUpdateExtension.ts:193-209`

### 5.4 版本恢复的潜在问题与处理

#### 问题：content 更新但 state 未同步

在 `restoreFromRevision` 方法中，只更新了 `content` 字段，没有直接更新 `state` 字段：

```typescript
restoreFromRevision = async (revision: Revision) => {
  // ...
  this.content = revision.content;  // ✅ 更新 content
  // this.state 没有直接更新 ⚠️
  // ...
};
```

**关键代码位置**：`server/models/Document.ts:903-915`

#### 解决方案

系统通过以下机制确保协作客户端同步：

1. **saveWithCtx 触发事件**：
   ```typescript
   await document.saveWithCtx(ctx, undefined, { name: "restore" });
   ```
   **关键代码位置**：`server/routes/api/documents/documents.ts:1013`

2. **Document 模型的 BeforeUpdate hook 会 backfill content**：
   ```typescript
   @BeforeUpdate
   static async processUpdate(model: Document) {
     // ...
     // backfill content if it's missing
     if (!model.content) {
       model.content = await DocumentHelper.toJSON(model);
     }
     // ...
   }
   ```
   **关键代码位置**：`server/models/Document.ts:532-534`

3. **APIUpdateExtension 的增量同步**：
   - 虽然 `state` 没有直接从 `Revision` 恢复
   - 但 `content` 更新后，协作客户端会收到增量更新
   - 新连接的客户端会从 `content` 重建 `state`

#### 注意事项

版本恢复操作会**覆盖**所有协作者的当前编辑状态。这是设计上的选择——版本恢复被视为"权威"操作，应该将所有客户端同步到指定的历史版本。

## 6. 完整数据流程总结

### 6.1 协作编辑 → 版本创建 完整流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        协作编辑数据流转                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐                                                        │
│  │   Client A      │                                                        │
│  │  (Y.Doc #123)   │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           │──本地编辑 (origin: local)                                        │
│           │                                                                 │
│           │──WebSocket──>┌──────────────────────┐                           │
│           │              │   Hocuspocus Server │                           │
│           │              │  (内存 Y.Doc 副本)  │                           │
│           │              └──────────┬───────────┘                           │
│  ┌────────┴────────┐                │                                        │
│  │   Client B      │                │                                        │
│  │  (Y.Doc #123)   │<──实时同步─────┤                                        │
│  └─────────────────┘                │                                        │
│                                     │                                        │
│                                     │──debounce (3s)──>                     │
│                                     │                                        │
│                              ┌──────┴──────┐                               │
│                              │ onStoreDocument│                             │
│                              │ PersistenceExt│                             │
│                              └──────┬──────┘                               │
│                                     │                                        │
│                                     ▼                                        │
│                        ┌──────────────────────────┐                        │
│                        │ documentCollaborativeUpdater│                      │
│                        │                          │                        │
│                        │ 1. Y.encodeStateAsUpdate()│                       │
│                        │    → Document.state (BLOB)│                       │
│                        │                          │                        │
│                        │ 2. yDocToProsemirrorJSON()│                       │
│                        │    → Document.content (JSONB)│                     │
│                        │                          │                        │
│                        │ 3. Event.schedule(        │                       │
│                        │    "documents.update"     │                       │
│                        │    {done: isLastConnection})│                     │
│                        └─────────────┬────────────┘                        │
│                                      │                                     │
│                                      ▼                                     │
│                             ┌──────────────────┐                           │
│                             │ RevisionsProcessor│                           │
│                             │   (异步队列)      │                           │
│                             │                  │                           │
│                             │ 1. Redis.smembers │                           │
│                             │    collaboratorIds │                          │
│                             │                  │                           │
│                             │ 2. 比较 content   │                           │
│                             │    vs previous.content│                       │
│                             │                  │                           │
│                             │ 3. 不同则创建    │                           │
│                             │    Revision 记录  │                           │
│                             └────────┬─────────┘                           │
│                                      │                                     │
│                                      ▼                                     │
│                              ┌─────────────────┐                           │
│                              │  revisions 表   │                           │
│                              │  (版本历史快照) │                           │
│                              │  - content      │                           │
│                              │  - title        │                           │
│                              │  - collaboratorIds │                         │
│                              └─────────────────┘                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 版本恢复 → 协作同步 完整流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        版本恢复数据流转                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐                                                        │
│  │ 用户选择历史版本 │                                                        │
│  │ (Revision #456) │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌──────────────────────────────────────────────────────────┐             │
│  │ POST /api/documents.restore                               │             │
│  │ { id: docId, revisionId: revId }                          │             │
│  └───────────────────────────┬──────────────────────────────┘             │
│                              │                                              │
│                              ▼                                              │
│              ┌─────────────────────────────────────┐                       │
│              │ 1. document.restoreFromRevision()   │                       │
│              │    - this.content = revision.content │                      │
│              │    - this.title = revision.title     │                      │
│              │    - this.icon = revision.icon       │                      │
│              │    - this.color = revision.color     │                      │
│              └───────────────────┬─────────────────┘                       │
│                                  │                                          │
│                                  ▼                                          │
│              ┌─────────────────────────────────────┐                       │
│              │ 2. document.saveWithCtx()            │                       │
│              │    - 触发 BeforeUpdate hook           │                       │
│              │    - revisionCount += 1               │                       │
│              │    - 触发 AfterUpdate hook            │                       │
│              └───────────────────┬─────────────────┘                       │
│                                  │                                          │
│                                  ▼                                          │
│              ┌─────────────────────────────────────┐                       │
│              │ 3. notifyCollaborationServer()       │                       │
│              │    (Document.AfterUpdate hook)        │                       │
│              │                                       │                       │
│              │    if (model.changed("state")) {      │                       │
│              │      APIUpdateExtension.notifyUpdate() │                      │
│              │    }                                  │                       │
│              └───────────────────┬─────────────────┘                       │
│                                  │                                          │
│                                  ▼                                          │
│              ┌─────────────────────────────────────┐                       │
│              │ 4. Redis PUBLISH                     │                       │
│              │    channel: collaboration:api-update:{docId} │             │
│              │    message: { actorId, timestamp }  │                       │
│              └───────────────────┬─────────────────┘                       │
│                                  │                                          │
│                                  ▼                                          │
│              ┌─────────────────────────────────────────────┐               │
│              │ 5. APIUpdateExtension.handleMessage()        │               │
│              │    (协作服务器端)                             │               │
│              │                                               │               │
│              │    a. 从数据库读取 Document.state            │               │
│              │    b. 重建 dbYdoc = new Y.Doc()            │               │
│              │    c. Y.applyUpdate(dbYdoc, dbState)       │               │
│              │    d. 计算增量: encodeStateAsUpdate         │               │
│              │       (dbYdoc, encodeStateVector(memDoc))   │               │
│              │    e. Y.applyUpdate(memDoc, update)         │               │
│              │    f. Hocuspocus 自动广播到所有连接客户端   │               │
│              └───────────────────────────┬─────────────────┘               │
│                                          │                                  │
│                                          ▼                                  │
│                              ┌───────────────────────┐                      │
│                              │  所有在线协作客户端    │                      │
│                              │  自动接收同步更新      │                      │
│                              │  (通过 WebSocket)      │                      │
│                              └───────────────────────┘                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 7. 关键技术点总结

### 7.1 双轨撤销策略

| 撤销类型 | 技术实现 | 协作范围 | 使用场景 |
|----------|----------|----------|----------|
| **本地撤销** | yUndoPlugin | 单用户操作栈 | 编辑中的撤销 (Ctrl+Z) |
| **版本恢复** | Revision 快照 + APIUpdateExtension | 全客户端同步 | 恢复到历史版本 |

### 7.2 核心机制对照表

| 机制 | 组件 | 关键函数/方法 | 文件位置 |
|------|------|---------------|----------|
| CRDT 同步 | y-prosemirror | ySyncPlugin, yUndoPlugin | `app/editor/extensions/Multiplayer.ts` |
| 协作服务 | Hocuspocus | Server.configure | `server/services/collaboration.ts` |
| 状态持久化 | PersistenceExtension | onLoadDocument, onStoreDocument | `server/collaboration/PersistenceExtension.ts` |
| Y.js ↔ ProseMirror | ProsemirrorHelper | toYDoc, yDocToProsemirrorJSON | `server/models/helpers/ProsemirrorHelper.tsx` |
| 版本创建 | RevisionsProcessor | perform | `server/queues/processors/RevisionsProcessor.ts` |
| 版本恢复 | Document | restoreFromRevision | `server/models/Document.ts:903` |
| 跨协作同步 | APIUpdateExtension | notifyUpdate, handleMessage | `server/collaboration/APIUpdateExtension.ts` |
| 远程事务识别 | multiplayer utils | isRemoteTransaction | `shared/editor/lib/multiplayer.ts:12` |

### 7.3 数据一致性保障

1. **协作编辑时**：Y.js CRDT 保证最终一致性
2. **持久化时**：`documentCollaborativeUpdater` 确保 content 和 state 同步更新
3. **版本创建时**：去重检查避免相同内容创建多个版本
4. **版本恢复时**：`APIUpdateExtension` 通过 Redis Pub/Sub + Y.js 增量更新确保所有客户端同步

### 7.4 性能优化策略

1. **异步版本创建**：使用 Bull 队列处理版本创建，不阻塞主流程
2. **防抖持久化**：Hocuspocus debounce (3s) 减少频繁写入
3. **增量同步**：`APIUpdateExtension` 使用 `encodeStateAsUpdate` 只传输差异
4. **本地缓存**：y-indexeddb 提供离线支持和快速恢复
5. **内容去重**：`fast-deep-equal` 比较 content，避免无意义的版本创建

## 8. 参考代码位置

| 功能模块 | 文件路径 |
|----------|----------|
| 版本模型 | `server/models/Revision.ts` |
| 文档模型 | `server/models/Document.ts` |
| 版本处理器 | `server/queues/processors/RevisionsProcessor.ts` |
| 协作文档更新 | `server/commands/documentCollaborativeUpdater.ts` |
| 持久化扩展 | `server/collaboration/PersistenceExtension.ts` |
| API 更新同步 | `server/collaboration/APIUpdateExtension.ts` |
| 前端协作扩展 | `app/editor/extensions/Multiplayer.ts` |
| 远程事务识别 | `shared/editor/lib/multiplayer.ts` |
| 版本恢复操作 | `app/actions/definitions/revisions.tsx` |
| 文档恢复端点 | `server/routes/api/documents/documents.ts:1006-1013` |
