# Outline 文档版本历史与撤销协作机制分析

## 1. 概述

Outline 是一个基于 Y.js CRDT（Conflict-free Replicated Data Type）的实时协作知识库系统。本文档深入分析其版本历史记录机制、**用户界面触发的版本恢复主路径**、以及多人协作场景下撤销操作如何跨 CRDT 和版本快照两个机制进行协作。

**关键修正**：
1. 用户从版本历史界面点击"恢复"时，**不是**通过后端 API 直接修改数据库，而是通过前端编辑器的 `replaceSelection` 操作实现。这个操作会被 `yUndoPlugin` 捕获到撤销栈中，用户可以按 Ctrl+Z 撤销这个"恢复操作"！
2. **恢复后的状态清理**：`history.replace(document.url, history.location.state)` 实际上没有清除 `restore` 和 `revisionId` 状态，这是一个潜在问题。
3. **备用恢复路径的协作同步**：`restoreFromRevision` 只更新 `content` 字段，**不更新 `state` 字段**；而 `notifyCollaborationServer` 只在 `state` 改变时触发，所以备用路径**不会**同步到协作客户端！

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

#### 协作者追踪

系统使用 Redis 实时追踪当前编辑会话的协作者：

```typescript
// 键格式: collaborators:{documentId}
const key = Document.getCollaboratorKey(event.documentId);
const collaboratorIds = await Redis.defaultClient.smembers(key);
await Redis.defaultClient.del(key); // 读取后清空
```

**关键代码位置**：`server/queues/processors/RevisionsProcessor.ts:26-28`

## 3. 版本恢复的主路径（用户 UI 触发）

### 3.1 核心发现

**重要**：用户从版本历史界面点击"恢复"按钮时，系统**不是**通过后端 `documents.restore` API 直接修改数据库，而是通过**前端编辑器操作**实现的。这个操作会被 `yUndoPlugin` 捕获，用户可以按 Ctrl+Z 撤销这个"恢复操作"！

### 3.2 恢复操作的完整流程

#### 第一阶段：导航触发

当用户点击"恢复"按钮时：

```typescript
// restoreRevision action
export const restoreRevision = createAction({
  name: ({ t }) => t("Restore"),
  // ...
  perform: async ({ event, location, activeDocumentId }) => {
    event?.preventDefault();
    
    // 1. 获取当前选中的 revisionId
    const match = matchPath<{ revisionId: string }>(location.pathname, {
      path: matchDocumentHistory,
    });
    const revisionId = match?.params.revisionId;

    const document = stores.documents.get(activeDocumentId);
    if (!document) {
      return;
    }

    // 2. 导航到文档页面，在 location.state 中传递恢复参数
    history.push(document.url, {
      restore: true,
      revisionId,
    });
  },
});
```

**关键代码位置**：`app/actions/definitions/revisions.tsx:17-45`

#### 第二阶段：编辑器同步完成后执行

在 `DocumentScene` 中，当编辑器同步完成后（`onSynced` 回调）：

```typescript
const onSynced = useCallback(async () => {
  const restore = location.state?.restore;
  const revisionId = location.state?.revisionId;
  const editor = editorRef.current;

  if (!editor) {
    return;
  }

  if (!restore) {
    return;  // 不是恢复操作，直接返回
  }

  // 1. 调用 API 获取版本数据
  const response = await client.post("/revisions.info", {
    id: revisionId,
  });

  if (response) {
    // 2. 用版本数据替换当前文档内容
    await replaceSelection(
      response.data,
      new AllSelection(editor.view.state.doc)
    );
    toast.success(t("Document restored"));
    
    // 3. ⚠️ 问题：这里没有真正清除恢复参数！
    // history.replace(document.url, history.location.state);
    // 这行代码把原来的 state（包含 restore: true）又传回去了
    history.replace(document.url, history.location.state);
  }
}, [location, replaceSelection, t, history, document.url]);
```

**关键代码位置**：`app/scenes/Document/components/Document.tsx:108-140`

### 3.3 ⚠️ 问题1：恢复后的状态清理不正确

**问题代码**：`app/scenes/Document/components/Document.tsx:138`

```typescript
// 当前代码（有问题）
history.replace(document.url, history.location.state);
```

**问题分析**：

| 问题 | 说明 |
|------|------|
| **状态没有被清除** | `history.location.state` 仍然包含 `restore: true` 和 `revisionId` |
| **重复执行风险** | 如果组件重新挂载或 `location` 因为其他原因改变，`onSynced` 可能再次执行 |
| **实际影响** | 虽然 `onSynced` 依赖于 `isLocalSynced` 和 `isRemoteSynced`，通常只会触发一次，但这是一个潜在的 bug |

**正确的做法应该是**：

```typescript
// 方案1：清除 restore 和 revisionId
history.replace(document.url, {
  ...history.location.state,
  restore: undefined,
  revisionId: undefined,
});

// 方案2：更彻底地清除所有状态
history.replace(document.url);
```

### 3.4 第三阶段：replaceSelection 执行编辑操作

`replaceSelection` 是核心函数，它执行一个标准的 ProseMirror 编辑操作：

```typescript
const replaceSelection = useCallback(
  (template: Template | Revision, selection?: Selection) => {
    const editor = editorRef.current;

    if (!editor) {
      return;
    }

    const { view, schema } = editor;
    const sel = selection ?? TextSelection.near(view.state.doc.resolve(0));
    
    // 1. 将版本数据转换为 ProseMirror Node
    const doc = Node.fromJSON(
      schema,
      ProsemirrorHelper.replaceTemplateVariables(template.data, auth.user!)
    );

    if (doc) {
      // 2. 执行 ProseMirror dispatch 操作
      // 这会触发：
      // - ySyncPlugin: 同步到 Y.js 内存状态
      // - yUndoPlugin: 捕获到撤销栈中
      view.dispatch(
        view.state.tr.setSelection(sel).replaceSelectionWith(doc)
      );
    }

    // 3. 标记编辑器为"脏"状态
    setIsEditorDirty(true);
    isEditorDirtyRef.current = true;

    // 4. 更新文档元数据（标题、图标、颜色）
    // ...

    // 5. 触发自动保存
    return onSaveRef.current({
      autosave: true,
      publish: false,
      done: false,
    });
  },
  [auth, document, editorRef]
);
```

**关键代码位置**：`app/scenes/Document/hooks/useDocumentSave.ts:194-249`

**关键点**：
- `view.dispatch()` 是标准的 ProseMirror 事务分发
- 这个事务会被 **ySyncPlugin** 同步到 Y.js 内存状态
- 同时会被 **yUndoPlugin** 捕获到撤销栈中！
- 用户可以按 **Ctrl+Z** 撤销这个"恢复操作"

### 3.5 主路径的协作同步

主路径的协作同步是自然的、正确的：

```
用户 A 执行恢复操作
      ↓
view.dispatch() 执行 ProseMirror 事务
      ↓
┌─────────────────────────────────────────┐
│  ySyncPlugin: 同步到 Y.js 内存状态     │
│  yUndoPlugin: 捕获到撤销栈中            │
└─────────────────────────────────────────┘
      ↓
HocuspocusProvider 通过 WebSocket 发送
      ↓
协作服务器接收更新
      ↓
其他协作者（用户 B、用户 C）自动同步
      ↓
结果：所有协作者都看到恢复后的文档
```

**关键优势**：
- ✅ 实时同步到所有协作者
- ✅ 恢复操作进入撤销栈，可 Ctrl+Z 撤销
- ✅ 撤销后也会同步到所有协作者
- ✅ `state` 和 `content` 都被正确更新（通过 `PersistenceExtension` 异步持久化）

## 4. 两种恢复路径对比

### 4.1 主路径 vs 备用路径

Outline 实际上有**两种**版本恢复路径，但它们的行为差异很大：

| 特性 | 主路径（用户 UI 触发） | 备用路径（后端 API - `revisionId` 分支） |
|------|------------------------|------------------------------------------|
| **触发方式** | 用户点击版本历史的"恢复"按钮 | 直接调用 `documents.restore` API，传递 `revisionId` |
| **实现方式** | `replaceSelection` + ProseMirror dispatch | `document.restoreFromRevision` |
| **执行位置** | 前端编辑器 | 后端数据库 |
| **更新的字段** | Y.js 内存状态 → 异步持久化到 `state` 和 `content` | 只更新 `content`、`title`、`icon`、`color` |
| **`state` 字段** | ✅ 更新（Y.js 同步） | ❌ **不更新** |
| **可撤销** | ✅ 可撤销（yUndoPlugin） | ❌ 不可撤销 |
| **协作同步** | ✅ 通过 Y.js 实时同步 | ❌ **不同步**（因为 `state` 没改变） |
| **撤销栈影响** | 进入撤销栈，可 Ctrl+Z 撤销 | 不影响撤销栈 |
| **用户界面使用** | ✅ 是 | ❌ 否（用户界面不使用这个分支） |

### 4.2 备用路径：documents.restore API

虽然用户界面不使用这个路径，但代码中确实存在一个后端恢复路径。让我们仔细分析它的行为：

#### 端点实现

```typescript
// 在 documents.restore 端点中
// 这个端点处理三种情况：
// 1. 恢复已删除的文档（document.deletedAt 存在）
// 2. 恢复已归档的文档（document.archivedAt 存在）
// 3. 恢复到特定版本（revisionId 存在）← 这是我们讨论的分支

else if (revisionId) {
  // restore a document to a specific revision
  authorize(user, "update", document);
  const revision = await Revision.findByPk(revisionId, { transaction });
  authorize(document, "restore", revision);

  // 直接更新数据库
  await document.restoreFromRevision(revision);
  await document.saveWithCtx(ctx, undefined, { name: "restore" });
}
```

**关键代码位置**：`server/routes/api/documents/documents.ts:1006-1013`

#### restoreFromRevision 方法

```typescript
restoreFromRevision = async (revision: Revision) => {
  if (revision.documentId !== this.id) {
    throw new Error("Revision does not belong to this document");
  }

  // 只更新了这些字段
  this.content = revision.content;
  this.text = await DocumentHelper.toMarkdown(revision, {
    includeTitle: false,
  });
  this.title = revision.title;
  this.icon = revision.icon;
  this.color = revision.color;
  
  // ⚠️ 没有更新 `state` 字段！
};
```

**关键代码位置**：`server/models/Document.ts:903-915`

### 4.3 ⚠️ 问题2：备用路径的协作同步问题

这是一个**严重的问题**：备用路径不会同步到协作客户端！

#### 问题分析：协作同步的触发条件

让我们梳理一下协作同步的触发条件：

1. **`notifyCollaborationServer` 只在 `state` 改变时触发**：
   ```typescript
   @AfterUpdate
   static notifyCollaborationServer(model: Document, ctx: HookContext) {
     // 注意：只有当 `state` 字段改变时才通知协作服务器！
     if (model.changed("state") && ctx.auth?.user?.id) {
       const actorId = ctx.auth.user.id;
       const notify = async () => {
         await APIUpdateExtension.notifyUpdate(model.id, actorId);
       };
       
       // ... 通知协作服务器
     }
   }
   ```
   **关键代码位置**：`server/models/Document.ts:586-599`

2. **`restoreFromRevision` 不更新 `state` 字段**：
   - 只更新 `content`、`text`、`title`、`icon`、`color`
   - **没有更新 `state`（Y.js CRDT 二进制状态）**

3. **结果**：
   - `model.changed("state")` 返回 `false`
   - `notifyCollaborationServer` 不触发
   - `APIUpdateExtension.notifyUpdate` 不被调用
   - **协作客户端不会收到同步通知！**

#### 问题后果

| 场景 | 结果 |
|------|------|
| **在线协作客户端** | 不会收到同步通知，继续使用旧的 Y.js 状态 |
| **新连接的客户端** | 从 `state` 字段加载（旧状态），而不是从 `content` 重建 |
| **持久化** | `content` 是新版本，但 `state` 是旧版本，数据不一致 |

#### 新连接客户端的加载流程

让我们看看新连接的客户端会发生什么：

```typescript
// PersistenceExtension.onLoadDocument
async onLoadDocument({ documentName, ...data }: withContext<onLoadDocumentPayload>) {
  const [, documentId] = documentName.split(".");
  const fieldName = "default";

  // 1. 首先尝试从 `state` 字段加载（Y.js 二进制）
  const documentWithoutLock = await Document.unscoped().findOne({
    attributes: ["state"],
    where: { id: documentId },
  });

  if (documentWithoutLock.state) {
    // ⚠️ 如果 `state` 存在，直接使用它！
    // 备用路径不会更新 `state`，所以这里加载的是旧状态！
    const ydoc = new Y.Doc();
    Y.applyUpdate(ydoc, documentWithoutLock.state);
    return ydoc;
  }

  // 2. 只有当 `state` 不存在时，才从 `content` 重建
  // ...
}
```

**关键代码位置**：`server/collaboration/PersistenceExtension.ts:19-96`

**问题**：
- 备用路径恢复版本后，`state` 字段仍然是旧的
- 新连接的客户端会从 `state` 加载，得到**旧版本**的文档！
- 这会导致严重的数据不一致

#### 问题图示

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    备用恢复路径的问题（协作不同步）                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  场景：用户 A 通过 API 调用 documents.restore 恢复到版本 A                   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  数据库状态                                                          │  │
│  │                                                                     │  │
│  │  Document:                                                          │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │  content:  版本 A 的内容（新）← restoreFromRevision 更新    │  │  │
│  │  │  state:    版本 B 的状态（旧）← 没有更新！                   │  │  │
│  │  │  title:    版本 A 的标题（新）← restoreFromRevision 更新    │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  notifyCollaborationServer 检查                                       │  │
│  │                                                                     │  │
│  │  if (model.changed("state") && ctx.auth?.user?.id) {              │  │
│  │    // state 没有改变！                                              │  │
│  │    // model.changed("state") 返回 false                             │  │
│  │    // 所以不会通知协作服务器！                                        │  │
│  │  }                                                                  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│                    ┌─────────┴─────────┐                                    │
│                    ▼                   ▼                                    │
│  ┌───────────────────────────┐  ┌───────────────────────────┐           │
│  │  在线协作客户端            │  │  新连接的客户端            │           │
│  │                           │  │                           │           │
│  │  - 没有收到同步通知        │  │  - 从 state 字段加载      │           │
│  │  - 继续使用旧的 Y.js 状态  │  │  - state 是旧版本！       │           │
│  │  - 显示旧版本内容          │  │  - 显示旧版本内容          │           │
│  │                           │  │                           │           │
│  │  ⚠️ 数据不一致！          │  │  ⚠️ 数据不一致！          │           │
│  └───────────────────────────┘  └───────────────────────────┘           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 备用路径的实际用途

虽然备用路径的 `revisionId` 分支存在协作同步问题，但它可能用于以下场景：

1. **恢复已删除的文档**：
   ```typescript
   if (document.deletedAt) {
     // restore a previously deleted document
     await document.restoreTo(ctx, { collectionId: destCollectionId! });
   }
   ```
   这种情况下，文档之前不在活跃状态，不需要同步 `state`。

2. **恢复已归档的文档**：
   ```typescript
   else if (document.archivedAt) {
     // restore a previously archived document
     await document.restoreTo(ctx, { collectionId: destCollectionId! });
   }
   ```
   同样，文档之前不在活跃状态。

3. **脚本或迁移任务**：
   - 通过 API 批量恢复版本
   - 不涉及实时协作

**关键发现**：用户界面不使用备用路径的 `revisionId` 分支，用户界面使用的是主路径（`replaceSelection` + Y.js 同步）。

## 5. 撤销操作机制

### 5.1 双轨撤销策略

Outline 采用**双轨撤销策略**，分别应对不同的协作场景：

| 撤销类型 | 实现组件 | 适用场景 | 协作感知 |
|----------|----------|----------|----------|
| **本地撤销** | `yUndoPlugin` (Y.js) | 实时协作编辑中的本地操作 | ✅ 感知远程变更 |
| **版本恢复** | `replaceSelection` + `yUndoPlugin` | 恢复到历史版本 | ✅ 可撤销，可同步 |
| **非协作撤销** | `prosemirror-history` | 非协作模式（单用户） | ❌ 不涉及协作 |

### 5.2 yUndoPlugin 工作原理

在协作模式下，`Multiplayer` 扩展注册 `yUndoPlugin`：

```typescript
get plugins() {
  const type = doc.get("default", Y.XmlFragment);
  return [
    ySyncPlugin(type),           // Y.js ↔ ProseMirror 同步
    yCursorPlugin(provider.awareness, { ... }),  // 光标同步
    yUndoPlugin(),               // 撤销/重做支持（核心）
    // ...
  ];
}

commands() {
  return {
    undo: () => undo,  // y-prosemirror 的 undo
    redo: () => redo,  // y-prosemirror 的 redo
  };
}
```

**关键代码位置**：`app/editor/extensions/Multiplayer.ts:118-138`

### 5.3 撤销边界

#### 可撤销的操作

| 操作类型 | 是否可撤销 | 说明 |
|----------|------------|------|
| 本地用户的编辑操作 | ✅ 可撤销 | 被 `yUndoPlugin` 捕获 |
| **版本恢复操作** | ✅ 可撤销 | 通过 `replaceSelection` 实现，进入撤销栈 |
| 模板插入操作 | ✅ 可撤销 | 同样使用 `replaceSelection` |
| 输入文本、删除文本 | ✅ 可撤销 | 标准编辑操作 |

#### 不可撤销的操作

| 操作类型 | 是否可撤销 | 说明 |
|----------|------------|------|
| 远程协作者的操作 | ❌ 不可撤销 | 通过 `isRemoteTransaction` 识别 |
| 直接调用 `documents.restore` API | ❌ 不可撤销 | 后端直接修改数据库，且不同步 |
| 文档发布/归档 | ❌ 不可撤销 | 状态变更操作 |

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

### 5.4 版本恢复与撤销的衔接

这是整个机制中**最关键**的部分：

#### 衔接流程

```
1. 用户点击"恢复"按钮
      ↓
2. history.push 导航，携带 restore: true, revisionId
      ↓
3. 编辑器同步完成 → onSynced 回调
      ↓
4. 调用 /revisions.info 获取版本数据
      ↓
5. 调用 replaceSelection(versionData, AllSelection)
      ↓
6. view.dispatch(...) 执行 ProseMirror 事务
      ↓
7. ySyncPlugin 同步到 Y.js 内存状态
      ↓
8. yUndoPlugin 捕获到撤销栈中 ← 关键！
      ↓
9. 用户可以按 Ctrl+Z 撤销这个"恢复操作"
```

#### 关键洞察

**版本恢复不是"回滚"，而是"前滚"**：

| 传统理解（错误） | 实际实现（正确） |
|------------------|------------------|
| 从数据库回滚到旧版本 | 将旧版本内容作为**新的编辑操作**应用 |
| 无法撤销恢复操作 | 恢复操作进入撤销栈，可 Ctrl+Z 撤销 |
| 其他协作者看到"突然回滚" | 其他协作者看到一个"编辑操作" |

#### 撤销"恢复操作"后的同步

当用户按 Ctrl+Z 撤销"恢复操作"时：

```
用户按 Ctrl+Z
      ↓
yUndoPlugin 执行撤销（Y.js 原生）
      ↓
Y.js Transaction 被创建（origin: yUndoPluginKey）
      ↓
ySyncPlugin 同步到 Y.js 内存状态
      ↓
HocuspocusProvider 通过 WebSocket 发送到协作服务器
      ↓
其他协作者接收更新
      ↓
其他协作者的 Y.js 状态自动同步
      ↓
结果：所有协作者都回到"恢复前"的状态
```

## 6. 关键问题总结

### 6.1 问题1：恢复后的状态清理不正确

| 项目 | 说明 |
|------|------|
| **问题代码位置** | `app/scenes/Document/components/Document.tsx:138` |
| **问题代码** | `history.replace(document.url, history.location.state);` |
| **问题描述** | 这行代码把原来的 `history.location.state`（包含 `restore: true` 和 `revisionId`）又传回去了，没有真正清除恢复相关的状态 |
| **潜在风险** | 如果组件重新挂载或 `location` 因为其他原因改变，`onSynced` 可能再次执行 |
| **实际影响** | 虽然 `onSynced` 依赖于 `isLocalSynced` 和 `isRemoteSynced`，通常只会触发一次，但这是一个潜在的 bug |
| **修复建议** | 清除 `restore` 和 `revisionId`，或完全清除 `location.state` |

### 6.2 问题2：备用恢复路径的协作同步问题

| 项目 | 说明 |
|------|------|
| **问题代码位置** | `server/models/Document.ts:903-915` 和 `server/models/Document.ts:586-599` |
| **问题描述** | `restoreFromRevision` 只更新 `content` 字段，**不更新 `state` 字段**；而 `notifyCollaborationServer` 只在 `state` 改变时触发 |
| **影响范围** | 备用路径（`documents.restore` API 的 `revisionId` 分支） |
| **用户界面是否使用** | ❌ 否（用户界面使用主路径） |
| **后果** | 在线协作客户端不会收到同步通知；新连接的客户端从 `state` 加载旧版本；`content` 和 `state` 数据不一致 |
| **实际用途** | 恢复已删除/归档的文档、脚本或迁移任务（不涉及实时协作） |

### 6.3 两种恢复路径的对比

| 特性 | 主路径（UI 触发） | 备用路径（API 触发 - `revisionId` 分支） |
|------|-------------------|------------------------------------------|
| **触发方式** | 用户点击"恢复"按钮 | 直接调用 API |
| **实现方式** | `replaceSelection` + ProseMirror dispatch | `restoreFromRevision` |
| **`state` 字段** | ✅ 通过 Y.js 同步更新 | ❌ 不更新 |
| **协作同步** | ✅ Y.js 实时同步 | ❌ 不同步 |
| **可撤销** | ✅ 可撤销（yUndoPlugin） | ❌ 不可撤销 |
| **数据一致性** | ✅ 一致 | ❌ 可能不一致 |
| **用户界面使用** | ✅ 是 | ❌ 否 |

## 7. 核心机制总结

### 7.1 主路径：版本恢复的完整数据流（正确）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    主路径：版本恢复的完整数据流（用户 UI 触发）              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  前端层                                                                      │
│  ───────                                                                    │
│                                                                              │
│  1. 用户点击版本历史的"恢复"按钮                                            │
│       ↓                                                                     │
│  2. restoreRevision action 执行                                             │
│     history.push(document.url, { restore: true, revisionId: "xxx" })      │
│       ↓                                                                     │
│  3. 导航到文档编辑页面                                                       │
│     MultiplayerEditor 初始化、同步本地和远程状态                            │
│       ↓                                                                     │
│  4. 同步完成 → onSynced 回调触发                                            │
│     检测到 location.state.restore === true                                  │
│       ↓                                                                     │
│  5. 调用 /revisions.info API 获取版本数据                                   │
│       ↓                                                                     │
│  6. 调用 replaceSelection()                                                 │
│     view.dispatch(view.state.tr.setSelection(sel).replaceSelectionWith(doc))│
│       ↓                                                                     │
│  7. ProseMirror 事务分发                                                    │
│       ↓                                                                     │
│     ┌─────────────────────────────────────────────────────────────────────┐│
│     │  ySyncPlugin: 同步到 Y.js 内存状态 → HocuspocusProvider 同步      ││
│     │  yUndoPlugin: 捕获到撤销栈中 → 可 Ctrl+Z 撤销                      ││
│     └─────────────────────────────────────────────────────────────────────┘│
│       ↓                                                                     │
│  8. ⚠️ 问题：history.replace 没有真正清除状态                               │
│     history.replace(document.url, history.location.state)                  │
│       ↓                                                                     │
│  9. 其他协作者自动同步更新                                                   │
│                                                                              │
│  后端层                                                                      │
│  ───────                                                                    │
│                                                                              │
│  10. 协作服务器接收更新                                                      │
│      PersistenceExtension.onChange()                                        │
│      Redis.sadd(collaborators:{docId}, userId)                             │
│       ↓                                                                     │
│  11. 防抖后持久化                                                            │
│      PersistenceExtension.onStoreDocument()                                 │
│      调用 documentCollaborativeUpdater()                                    │
│       ↓                                                                     │
│  12. 更新数据库                                                              │
│      ✅ 更新 Document.state（Y.encodeStateAsUpdate）                        │
│      ✅ 更新 Document.content（yDocToProsemirrorJSON）                      │
│       ↓                                                                     │
│  13. 异步创建版本快照                                                        │
│      RevisionsProcessor 比较 content，不同则创建 Revision 记录              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 备用路径：版本恢复的数据流（有问题）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    备用路径：版本恢复的数据流（API 触发，有问题）           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  前端层                                                                      │
│  ───────                                                                    │
│                                                                              │
│  1. 调用 documents.restore API                                               │
│     POST /api/documents.restore { id: docId, revisionId: revId }          │
│       ↓                                                                     │
│  后端层                                                                      │
│  ───────                                                                    │
│                                                                              │
│  2. 端点处理                                                                 │
│     else if (revisionId) {                                                  │
│       await document.restoreFromRevision(revision);                        │
│       await document.saveWithCtx(ctx, ...);                                │
│     }                                                                       │
│       ↓                                                                     │
│  3. restoreFromRevision 执行                                                │
│     this.content = revision.content;  // ✅ 更新                            │
│     this.title = revision.title;      // ✅ 更新                            │
│     this.icon = revision.icon;        // ✅ 更新                            │
│     this.color = revision.color;      // ✅ 更新                            │
│     // ⚠️ 没有更新 this.state！       // ❌ 不更新                          │
│       ↓                                                                     │
│  4. saveWithCtx 触发 AfterUpdate hooks                                      │
│       ↓                                                                     │
│  5. notifyCollaborationServer 检查                                          │
│     if (model.changed("state") && ctx.auth?.user?.id) {                   │
│       // state 没有改变！                                                  │
│       // model.changed("state") 返回 false                                │
│       // ❌ 不会通知协作服务器！                                            │
│     }                                                                       │
│       ↓                                                                     │
│  后果：                                                                     │
│  • 在线协作客户端：没有收到同步通知，继续使用旧状态                          │
│  • 新连接的客户端：从 state 字段加载旧版本                                   │
│  • 数据库：content 是新版本，state 是旧版本，数据不一致                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 8. 参考代码位置

| 功能模块 | 文件路径 |
|----------|----------|
| 恢复操作触发 | `app/actions/definitions/revisions.tsx:17-45` |
| 恢复执行逻辑 | `app/scenes/Document/components/Document.tsx:108-140` |
| **问题1：状态清理** | `app/scenes/Document/components/Document.tsx:138` |
| 内容替换核心 | `app/scenes/Document/hooks/useDocumentSave.ts:194-249` |
| 协作扩展 | `app/editor/extensions/Multiplayer.ts` |
| 恢复版本方法 | `server/models/Document.ts:903-915` |
| **问题2：协作同步触发** | `server/models/Document.ts:586-599` |
| 版本模型 | `server/models/Revision.ts` |
| 版本处理器 | `server/queues/processors/RevisionsProcessor.ts` |
| 持久化扩展 | `server/collaboration/PersistenceExtension.ts` |
| 版本信息 API | `server/routes/api/revisions/revisions.ts` |
| 文档恢复端点 | `server/routes/api/documents/documents.ts:1006-1013` |
