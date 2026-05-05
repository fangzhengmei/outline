# Outline 文档版本历史与撤销协作机制分析

## 1. 概述

Outline 是一个基于 Y.js CRDT（Conflict-free Replicated Data Type）的实时协作知识库系统。本文档深入分析其版本历史记录机制、**用户界面触发的版本恢复主路径**、以及多人协作场景下撤销操作如何跨 CRDT 和版本快照两个机制进行协作。

**关键修正**：用户从版本历史界面点击"恢复"时，**不是**通过后端 API 直接修改数据库，而是通过前端编辑器的 `replaceSelection` 操作实现。这个操作会被 `yUndoPlugin` 捕获到撤销栈中，用户可以按 Ctrl+Z 撤销这个"恢复操作"！

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

### 3.2 版本历史界面

#### 界面结构

版本历史界面位于 `History.tsx`，包含：
- 版本列表（按时间倒序）
- "显示变更"开关
- 版本对比功能

**关键代码位置**：`app/scenes/Document/components/History/History.tsx`

#### 版本列表项

每个版本项包含：
- 时间戳
- 协作者头像
- "恢复"操作按钮
- 更多菜单（复制链接、下载、删除）

**关键代码位置**：`app/scenes/Document/components/History/RevisionListItem.tsx`

### 3.3 恢复操作的完整流程

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

**关键点**：
- 这里没有直接调用 API
- 只是通过 `history.push` 导航，并在 `location.state` 中设置：
  - `restore: true` - 标记这是一个恢复操作
  - `revisionId` - 要恢复的版本 ID

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

  // ...

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
    
    // 3. 清除 location.state 中的恢复参数
    history.replace(document.url, history.location.state);
  }
}, [location, replaceSelection, t, history, document.url]);
```

**关键代码位置**：`app/scenes/Document/components/Document.tsx:108-140`

**关键点**：
- `onSynced` 回调在 **MultiplayerEditor** 同步完成后触发
- 检测到 `restore: true` 后执行恢复逻辑
- 调用 `/revisions.info` API 获取版本的完整数据
- 调用 `replaceSelection` 用版本数据**替换当前文档的全部内容**

#### 第三阶段：replaceSelection 执行编辑操作

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
    if (template instanceof Template) {
      document.templateId = template.id;
      document.fullWidth = template.fullWidth;
    }

    if (!titleRef.current) {
      const newTitle = TextHelper.replaceTemplateVariables(
        template.title,
        auth.user!
      );
      setTitle(newTitle);
      titleRef.current = newTitle;
      document.title = newTitle;
    }
    if (template.icon) {
      document.icon = template.icon;
    }
    if (template.color) {
      document.color = template.color;
    }

    // 5. 触发自动保存
    document.data = cloneDeep(template.data);
    updateIsDirtyRef.current();

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

### 3.4 流程时序图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    版本恢复的完整流程（用户 UI 触发）                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┐                                                        │
│  │  用户点击"恢复"   │                                                        │
│  │  (版本历史界面)   │                                                        │
│  └────────┬─────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌──────────────────────────────────────────────────────────┐             │
│  │ restoreRevision action 执行                                │             │
│  │                                                           │             │
│  │ history.push(document.url, {                              │             │
│  │   restore: true,                                          │             │
│  │   revisionId: "xxx"                                       │             │
│  │ })                                                         │             │
│  └───────────────────────────┬──────────────────────────────┘             │
│                              │                                              │
│                              ▼                                              │
│              ┌─────────────────────────────────────────┐                   │
│              │  导航到文档编辑页面                       │                   │
│              │  MultiplayerEditor 开始初始化             │                   │
│              └───────────────────┬─────────────────────┘                   │
│                                  │                                          │
│                                  ▼                                          │
│              ┌─────────────────────────────────────────┐                   │
│              │  HocuspocusProvider 连接协作服务器       │                   │
│              │  - 从 IndexedDB 加载本地状态             │                   │
│              │  - 从服务器同步最新状态                   │                   │
│              └───────────────────┬─────────────────────┘                   │
│                                  │                                          │
│                                  ▼                                          │
│              ┌─────────────────────────────────────────┐                   │
│              │  同步完成 → onSynced 回调触发            │                   │
│              │  检测到 location.state.restore === true  │                   │
│              └───────────────────┬─────────────────────┘                   │
│                                  │                                          │
│                                  ▼                                          │
│              ┌─────────────────────────────────────────┐                   │
│              │  调用 /revisions.info API                │                   │
│              │  获取版本的完整数据（content, title 等） │                   │
│              └───────────────────┬─────────────────────┘                   │
│                                  │                                          │
│                                  ▼                                          │
│              ┌─────────────────────────────────────────┐                   │
│              │  调用 replaceSelection()                 │                   │
│              │                                           │                   │
│              │  view.dispatch(                          │                   │
│              │    view.state.tr                         │                   │
│              │      .setSelection(sel)                  │                   │
│              │      .replaceSelectionWith(doc)          │                   │
│              │  )                                        │                   │
│              └───────────────────┬─────────────────────┘                   │
│                                  │                                          │
│                                  ▼                                          │
│              ┌─────────────────────────────────────────┐                   │
│              │  ProseMirror 事务分发                    │                   │
│              └───────────────────┬─────────────────────┘                   │
│                                  │                                          │
│                    ┌─────────────┴─────────────┐                         │
│                    ▼                           ▼                         │
│       ┌─────────────────────┐      ┌─────────────────────┐              │
│       │    ySyncPlugin      │      │    yUndoPlugin      │              │
│       │                     │      │                     │              │
│       │  同步到 Y.js 内存    │      │  捕获到撤销栈中      │              │
│       │  状态（可协作同步）  │      │  （可 Ctrl+Z 撤销） │              │
│       └──────────┬──────────┘      └──────────┬──────────┘              │
│                  │                              │                          │
│                  ▼                              │                          │
│       ┌─────────────────────┐                   │                          │
│       │ HocuspocusProvider  │                   │                          │
│       │ 通过 WebSocket 发送  │                   │                          │
│       │ 到协作服务器          │                   │                          │
│       └──────────┬──────────┘                   │                          │
│                  │                              │                          │
│                  ▼                              │                          │
│       ┌─────────────────────┐                   │                          │
│       │  其他协作者接收更新   │                   │                          │
│       │  （自动同步）        │                   │                          │
│       └─────────────────────┘                   │                          │
│                                                  │                          │
│                                                  ▼                          │
│                                       ┌─────────────────────┐             │
│                                       │  用户按 Ctrl+Z       │             │
│                                       │  撤销"恢复操作"      │             │
│                                       └──────────┬──────────┘             │
│                                                  │                          │
│                                                  ▼                          │
│                                       ┌─────────────────────┐             │
│                                       │  yUndoPlugin 执行   │             │
│                                       │  撤销（Y.js 原生）   │             │
│                                       └──────────┬──────────┘             │
│                                                  │                          │
│                                                  ▼                          │
│                                       ┌─────────────────────┐             │
│                                       │  同步到其他协作者     │             │
│                                       │  （"撤销恢复"同步）  │             │
│                                       └─────────────────────┘             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.5 /revisions.info API

这个 API 返回版本的完整数据，用于前端恢复操作：

```typescript
router.post(
  "revisions.info",
  auth(),
  validate(T.RevisionsInfoSchema),
  async (ctx: APIContext<T.RevisionsInfoReq>) => {
    const { id, documentId } = ctx.input.body;
    const { user } = ctx.state.auth;
    let revision: Revision;

    if (id) {
      revision = await Revision.findByPk(id, {
        rejectOnEmpty: true,
      });

      const document = await Document.findByPk(revision.documentId, {
        userId: user.id,
      });
      authorize(user, "listRevisions", document);
    } 
    // ...

    // presentRevision 会构建完整的版本数据
    ctx.body = {
      data: await presentRevision(revision),
      policies: presentPolicies(user, [revision]),
    };
  }
);
```

**关键代码位置**：`server/routes/api/revisions/revisions.ts:29-64`

#### presentRevision 构建版本数据

```typescript
async function presentRevision(revision: Revision) {
  const { emoji, strippedTitle } = parseTitle(revision.title);

  // 构建版本数据
  const [data, text, collaborators] = await Promise.all([
    DocumentHelper.toJSON(revision),      // ProseMirror 格式数据
    DocumentHelper.toMarkdown(revision),  // Markdown 格式
    revision.collaborators,                 // 协作者信息
  ]);

  return {
    id: revision.id,
    documentId: revision.documentId,
    title: strippedTitle,
    name: revision.name,
    data,           // ← 这个 data 字段用于 replaceSelection
    text,
    icon: revision.icon ?? emoji,
    color: revision.color,
    collaborators: collaborators.map((user) => presentUser(user)),
    createdAt: revision.createdAt,
    createdBy: presentUser(revision.user),
    createdById: revision.userId,
    deletedAt: revision.deletedAt,
  };
}
```

**关键代码位置**：`server/presenters/revision.ts:7-32`

## 4. 两种恢复路径对比

### 4.1 主路径 vs 备用路径

Outline 实际上有**两种**版本恢复路径：

| 特性 | 主路径（用户 UI 触发） | 备用路径（后端 API） |
|------|------------------------|----------------------|
| **触发方式** | 用户点击版本历史的"恢复"按钮 | 直接调用 `documents.restore` API |
| **实现方式** | `replaceSelection` + ProseMirror dispatch | `document.restoreFromRevision` |
| **执行位置** | 前端编辑器 | 后端数据库 |
| **可撤销** | ✅ 可撤销（yUndoPlugin） | ❌ 不可撤销 |
| **协作同步** | ✅ 通过 Y.js 实时同步 | ⚠️ 通过 APIUpdateExtension 同步 |
| **撤销栈影响** | 进入撤销栈，可 Ctrl+Z 撤销 | 不影响撤销栈 |
| **用户感知** | 自然的编辑操作体验 | 可能丢失未保存的编辑 |

### 4.2 备用路径：documents.restore API

虽然用户界面不使用这个路径，但代码中确实存在一个后端恢复路径：

```typescript
// 在 documents.restore 端点中
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

  // 直接复制快照字段到当前文档
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

#### 备用路径的同步机制

当使用备用路径时，通过 `APIUpdateExtension` 同步到协作客户端：

```
documents.restore API
      ↓
document.restoreFromRevision()
      ↓
document.saveWithCtx()
      ↓
Document.notifyCollaborationServer()
      ↓
Redis PUBLISH 到 collaboration:api-update:{docId}
      ↓
APIUpdateExtension.handleMessage()
      ↓
从数据库读取最新 state
      ↓
计算与内存 Y.Doc 的 diff
      ↓
Y.applyUpdate() 到内存文档
      ↓
Hocuspocus 广播到所有连接客户端
```

### 4.3 为什么主路径更好

主路径（前端编辑器操作）相比备用路径有以下优势：

1. **可撤销**：用户可以按 Ctrl+Z 撤销"恢复操作"，提供了安全网
2. **自然的协作体验**：通过 Y.js 实时同步，其他协作者看到的是一个"编辑操作"而不是突然的回滚
3. **保留撤销栈**：恢复操作本身作为一个编辑操作进入撤销栈，用户可以在"恢复前"和"恢复后"之间切换
4. **不丢失未保存编辑**：如果用户有未保存的编辑，恢复操作会进入撤销栈，用户可以撤销恢复回到之前的编辑状态

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

#### yUndoPlugin 的核心特性

`yUndoPlugin` 是 Y.js 提供的撤销插件，具有以下特性：

1. **基于 Y.js Transaction**：撤销操作是 Y.js 原生的，不是 ProseMirror 的
2. **忽略远程事务**：只撤销本地用户的操作，不会撤销其他协作者的变更
3. **堆栈隔离**：每个客户端维护独立的撤销堆栈
4. **操作追踪**：通过 Y.Transaction 的 `origin` 标识区分操作来源

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
| 直接调用 `documents.restore` API | ❌ 不可撤销 | 后端直接修改数据库 |
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

### 5.5 撤销栈结构示意

假设用户执行了以下操作：

```
操作 1: 输入 "Hello"
操作 2: 输入 " World"
操作 3: 恢复到版本 A（replaceSelection）
操作 4: 输入 "More"
```

撤销栈结构：

```
┌─────────────────────────────────────────────────────────────────┐
│                        yUndoPlugin 撤销栈                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  栈顶（可撤销）                                          │  │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐          │  │
│  │  │ 操作 4    │  │ 操作 3    │  │ 操作 2    │  操作 1   │  │
│  │  │ "More"    │  │ 恢复版本A │  │ " World"  │  "Hello"  │  │
│  │  └───────────┘  └───────────┘  └───────────┘          │  │
│  │                              ↑                          │  │
│  │                              │                          │  │
│  │                    这是一个标准的编辑操作！               │  │
│  │                    可以被 Ctrl+Z 撤销                    │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  用户按 Ctrl+Z 一次 → 撤销"操作 4" → 文档内容 = 版本 A       │
│  用户按 Ctrl+Z 两次 → 撤销"操作 3" → 文档内容 = "Hello World"│
│  用户按 Ctrl+Z 三次 → 撤销"操作 2" → 文档内容 = "Hello"      │
│  用户按 Ctrl+Z 四次 → 撤销"操作 1" → 文档内容 = 空           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**关键点**：
- 版本恢复操作（操作 3）和普通编辑操作（操作 1、2、4）在撤销栈中**完全平等**
- 用户可以在"恢复前"和"恢复后"之间自由切换
- 所有撤销操作都会通过 Y.js 实时同步到其他协作者

## 6. CRDT (Y.js) 集成实现

### 6.1 技术选型

Outline 使用以下技术栈实现实时协作：

| 组件 | 库/框架 | 用途 |
|------|---------|------|
| CRDT 引擎 | Y.js | 无冲突数据类型核心 |
| 同步协议 | Hocuspocus | WebSocket 同步服务器 |
| ProseMirror 桥接 | y-prosemirror | Y.js ↔ ProseMirror 转换 |
| 本地持久化 | y-indexeddb | 浏览器本地缓存 |

### 6.2 前端集成

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

#### MultiplayerEditor 组件

`MultiplayerEditor` 管理协作连接和状态：

```typescript
function MultiplayerEditor(
  { onSynced, ...props }: Props,
  ref: ForwardedRef<SharedEditor>
) {
  const [ydoc] = useState(() => new Y.Doc());
  
  // ...

  useLayoutEffect(() => {
    // 1. 本地持久化（IndexedDB）
    const localProvider =
      typeof indexedDB !== "undefined"
        ? new IndexeddbPersistence(name, ydoc)
        : undefined;

    // 2. 远程连接（Hocuspocus WebSocket）
    const provider = new HocuspocusProvider({
      parameters: {
        editorVersion: EDITOR_VERSION,
      },
      url: `${env.COLLABORATION_URL}/collaboration`,
      name,
      document: ydoc,
      token,
    });

    // 3. 同步完成回调
    provider.on("synced", () => {
      presence.touch(documentId, currentUser.id, false);
      setRemoteSynced(true);
      retryCount.current = 0;
    });
  }, [...]);

  // 4. 当本地和远程都同步完成后，调用 onSynced
  useEffect(() => {
    if ((!hasLocalPersistence || isLocalSynced) && isRemoteSynced) {
      void onSynced?.();  // ← 版本恢复操作在这里触发！
    }
  }, [onSynced, hasLocalPersistence, isLocalSynced, isRemoteSynced]);

  // ...
}
```

**关键代码位置**：`app/scenes/Document/components/MultiplayerEditor.tsx:55-358`

### 6.3 后端服务架构

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

  // 1. 尝试从 state 字段加载（Y.js 二进制）
  const documentWithoutLock = await Document.unscoped().findOne({
    attributes: ["state"],
    where: { id: documentId },
  });

  if (documentWithoutLock.state) {
    const ydoc = new Y.Doc();
    Y.applyUpdate(ydoc, documentWithoutLock.state);
    return ydoc;
  }

  // 2. 回退：从 content 或 text 重建 Y.js 状态
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

  // 1. 从 Redis 获取当前会话的协作者
  const key = Document.getCollaboratorKey(documentId);
  const sessionCollaboratorIds = await Redis.defaultClient.smembers(key);

  if (!sessionCollaboratorIds || sessionCollaboratorIds.length === 0) {
    return;  // 无变更，跳过
  }

  // 2. 调用协作文档更新命令
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

### 6.4 双存储架构

Outline 采用**双存储**架构来兼顾协作效率和版本管理：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Document 模型                                   │
├─────────────────────────┬───────────────────────────────────────────┤
│      content (JSONB)    │         state (BLOB)                    │
├─────────────────────────┼───────────────────────────────────────────┤
│  ProseMirror 快照格式    │    Y.js CRDT 二进制状态                 │
│                         │                                           │
│  • 人类可读              │    • 协作编辑核心                       │
│  • 版本比对基础          │    • 实时同步必需                       │
│  • 历史恢复源            │    • 包含完整操作历史                   │
│  • 搜索引擎索引          │    • 可增量更新                         │
│                         │                                           │
│  更新时机:               │    更新时机:                            │
│   - 协作断开时           │    - 每次协作变更时                     │
│   - API 更新时           │    - 文档加载重建时                     │
└─────────────────────────┴───────────────────────────────────────────┘
```

## 7. 完整数据流程总结

### 7.1 版本恢复的完整数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    版本恢复的完整数据流（用户 UI 触发）                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  前端层                                                                      │
│  ───────                                                                    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  1. 用户点击版本历史的"恢复"按钮                                      │  │
│  │                                                                     │  │
│  │     restoreRevision action:                                         │  │
│  │     history.push(document.url, {                                    │  │
│  │       restore: true,                                                │  │
│  │       revisionId: "rev-123"                                        │  │
│  │     })                                                              │  │
│  └───────────────────────────┬─────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  2. 导航到文档编辑页面                                                │  │
│  │                                                                     │  │
│  │     MultiplayerEditor 初始化:                                        │  │
│  │     - 创建 Y.Doc() 实例                                              │  │
│  │     - IndexeddbPersistence 加载本地状态                              │  │
│  │     - HocuspocusProvider 连接协作服务器                              │  │
│  │     - 同步本地和远程状态                                             │  │
│  └───────────────────────────┬─────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  3. 同步完成 → onSynced 回调触发                                      │  │
│  │                                                                     │  │
│  │     useEffect(() => {                                                │  │
│  │       if ((!hasLocalPersistence || isLocalSynced) && isRemoteSynced) {│
│  │         void onSynced?.();  // ← 检测 restore: true                │  │
│  │       }                                                              │  │
│  │     }, [...]);                                                       │  │
│  └───────────────────────────┬─────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  4. onSynced 执行恢复逻辑                                             │  │
│  │                                                                     │  │
│  │     const onSynced = useCallback(async () => {                      │  │
│  │       const restore = location.state?.restore;                       │  │
│  │       const revisionId = location.state?.revisionId;                 │  │
│  │                                                                     │  │
│  │       if (!restore) return;                                          │  │
│  │                                                                     │  │
│  │       // 调用 API 获取版本数据                                       │  │
│  │       const response = await client.post("/revisions.info", {       │  │
│  │         id: revisionId,                                              │  │
│  │       });                                                           │  │
│  │                                                                     │  │
│  │       // 替换文档内容                                               │  │
│  │       await replaceSelection(                                        │  │
│  │         response.data,                                               │  │
│  │         new AllSelection(editor.view.state.doc)                     │  │
│  │       );                                                            │  │
│  │     }, [...]);                                                       │  │
│  └───────────────────────────┬─────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  5. replaceSelection 执行编辑操作                                     │  │
│  │                                                                     │  │
│  │     const replaceSelection = useCallback(                            │  │
│  │       (template: Template | Revision, selection?: Selection) => {   │  │
│  │         const { view, schema } = editor;                             │  │
│  │                                                                     │  │
│  │         // 转换为 ProseMirror Node                                   │  │
│  │         const doc = Node.fromJSON(                                    │  │
│  │           schema,                                                     │  │
│  │           ProsemirrorHelper.replaceTemplateVariables(template.data,  │  │
│  │             auth.user!)                                               │  │
│  │         );                                                           │  │
│  │                                                                     │  │
│  │         // 执行 ProseMirror dispatch ← 关键！                        │  │
│  │         view.dispatch(                                                │  │
│  │           view.state.tr                                               │  │
│  │             .setSelection(sel)                                        │  │
│  │             .replaceSelectionWith(doc)                                │  │
│  │         );                                                           │  │
│  │       },                                                               │  │
│  │       [auth, document, editorRef]                                     │  │
│  │     );                                                               │  │
│  └───────────────────────────┬─────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  6. ProseMirror 事务分发                                              │  │
│  └───────────────────────────┬─────────────────────────────────────────┘  │
│                              │                                              │
│                    ┌─────────┴─────────┐                                  │
│                    ▼                   ▼                                  │
│       ┌─────────────────────┐  ┌─────────────────────┐                   │
│       │    ySyncPlugin      │  │    yUndoPlugin      │                   │
│       │                     │  │                     │                   │
│       │  同步到 Y.js 内存    │  │  捕获到撤销栈中      │                   │
│       │  状态               │  │                     │                   │
│       └──────────┬──────────┘  └──────────┬──────────┘                   │
│                  │                          │                               │
│                  ▼                          │                               │
│       ┌─────────────────────┐               │                               │
│       │ HocuspocusProvider  │               │                               │
│       │ 通过 WebSocket 发送  │               │                               │
│       │ 到协作服务器          │               │                               │
│       └──────────┬──────────┘               │                               │
│                  │                          │                               │
│                  ▼                          │                               │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  7. 用户可以按 Ctrl+Z 撤销这个"恢复操作"                              │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  后端层                                                                      │
│  ───────                                                                    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  8. 协作服务器接收更新                                                │  │
│  │                                                                     │  │
│  │     PersistenceExtension.onChange():                                 │  │
│  │     - Redis.sadd(collaborators:{docId}, userId)                    │  │
│  └───────────────────────────┬─────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  9. 防抖后持久化到数据库                                              │  │
│  │                                                                     │  │
│  │     PersistenceExtension.onStoreDocument():                          │  │
│  │     - 从 Redis 获取 sessionCollaboratorIds                           │  │
│  │     - 调用 documentCollaborativeUpdater()                            │  │
│  └───────────────────────────┬─────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  10. 更新数据库                                                       │  │
│  │                                                                     │  │
│  │     documentCollaborativeUpdater:                                    │  │
│  │     - 编码 Y.js state: Y.encodeStateAsUpdate(ydoc)                  │  │
│  │     - 转换为 ProseMirror: yDocToProsemirrorJSON(ydoc, "default")   │  │
│  │     - 更新 Document.state 和 Document.content                         │  │
│  │     - 触发 documents.update 事件                                      │  │
│  └───────────────────────────┬─────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  11. 异步创建版本快照                                                 │  │
│  │                                                                     │  │
│  │     RevisionsProcessor:                                               │  │
│  │     - 比较当前 content 与前一 Revision.content                        │  │
│  │     - 不同则创建新的 Revision 记录                                     │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 协作编辑 → 版本创建 完整流程

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
│                              └─────────────────┘                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 8. 关键技术点总结

### 8.1 双轨恢复路径

| 路径 | 触发方式 | 实现方式 | 可撤销 | 同步方式 |
|------|----------|----------|--------|----------|
| **主路径（UI）** | 用户点击"恢复"按钮 | `replaceSelection` + ProseMirror dispatch | ✅ 可撤销 | Y.js 实时同步 |
| **备用路径（API）** | 直接调用 `documents.restore` | `restoreFromRevision` 直接修改数据库 | ❌ 不可撤销 | APIUpdateExtension 同步 |

### 8.2 版本恢复与撤销的衔接

| 步骤 | 操作 | 关键代码位置 |
|------|------|--------------|
| 1 | 用户点击"恢复"按钮 | `RevisionListItem.tsx` |
| 2 | `restoreRevision` action 触发导航 | `app/actions/definitions/revisions.tsx:17-45` |
| 3 | `location.state` 携带 `restore: true` 和 `revisionId` | `DocumentScene.onSynced` |
| 4 | 编辑器同步完成后执行恢复 | `app/scenes/Document/components/Document.tsx:108-140` |
| 5 | 调用 `/revisions.info` 获取版本数据 | `server/routes/api/revisions/revisions.ts:29-64` |
| 6 | `replaceSelection` 执行编辑操作 | `app/scenes/Document/hooks/useDocumentSave.ts:194-249` |
| 7 | `view.dispatch()` 触发 ProseMirror 事务 | `useDocumentSave.ts:210-212` |
| 8 | `ySyncPlugin` 同步到 Y.js | `Multiplayer.ts:119` |
| 9 | `yUndoPlugin` 捕获到撤销栈 | `Multiplayer.ts:124` |
| 10 | 用户可按 Ctrl+Z 撤销"恢复操作" | `yUndoPlugin` 原生支持 |

### 8.3 撤销边界

#### 可撤销的操作

| 操作类型 | 撤销机制 | 协作同步 |
|----------|----------|----------|
| 本地编辑操作 | `yUndoPlugin` | ✅ 同步到其他协作者 |
| **版本恢复操作** | `yUndoPlugin`（通过 `replaceSelection`） | ✅ 同步到其他协作者 |
| 模板插入操作 | `yUndoPlugin`（通过 `replaceSelection`） | ✅ 同步到其他协作者 |

#### 不可撤销的操作

| 操作类型 | 原因 |
|----------|------|
| 远程协作者的操作 | 通过 `isRemoteTransaction` 识别，`yUndoPlugin` 忽略 |
| 直接调用 `documents.restore` API | 后端直接修改数据库，不经过编辑器撤销栈 |
| 文档发布/归档 | 状态变更操作，不是编辑操作 |

### 8.4 核心机制对照表

| 机制 | 组件 | 关键函数/方法 | 文件位置 |
|------|------|---------------|----------|
| CRDT 同步 | y-prosemirror | `ySyncPlugin`, `yUndoPlugin` | `app/editor/extensions/Multiplayer.ts` |
| 版本恢复触发 | restoreRevision action | `perform` | `app/actions/definitions/revisions.tsx:17-45` |
| 恢复执行 | DocumentScene | `onSynced` | `app/scenes/Document/components/Document.tsx:108-140` |
| 内容替换 | useDocumentSave | `replaceSelection` | `app/scenes/Document/hooks/useDocumentSave.ts:194-249` |
| 远程事务识别 | multiplayer utils | `isRemoteTransaction` | `shared/editor/lib/multiplayer.ts:12` |
| 状态持久化 | PersistenceExtension | `onLoadDocument`, `onStoreDocument` | `server/collaboration/PersistenceExtension.ts` |
| 版本创建 | RevisionsProcessor | `perform` | `server/queues/processors/RevisionsProcessor.ts` |
| API 更新同步 | APIUpdateExtension | `notifyUpdate`, `handleMessage` | `server/collaboration/APIUpdateExtension.ts` |

### 8.5 性能优化策略

1. **异步版本创建**：使用 Bull 队列处理版本创建，不阻塞主流程
2. **防抖持久化**：Hocuspocus debounce (3s) 减少频繁写入
3. **增量同步**：`APIUpdateExtension` 使用 `encodeStateAsUpdate` 只传输差异
4. **本地缓存**：y-indexeddb 提供离线支持和快速恢复
5. **内容去重**：`fast-deep-equal` 比较 content，避免无意义的版本创建

## 9. 关键洞察

### 9.1 版本恢复不是"回滚"，而是"前滚"

这是最重要的洞察：

**传统理解**：版本恢复是从数据库回滚到旧版本，是一个"破坏性"操作，无法撤销。

**实际实现**：版本恢复是将旧版本的内容作为一个**新的编辑操作**应用到当前文档。这个操作：
- 进入 `yUndoPlugin` 的撤销栈
- 用户可以按 Ctrl+Z 撤销
- 其他协作者看到的是一个"编辑操作"而不是突然的回滚

### 9.2 撤销栈的"时间旅行"能力

由于版本恢复操作和普通编辑操作在撤销栈中完全平等，用户实际上拥有了"时间旅行"能力：

```
场景：
- 版本 A (昨天)
- 用户编辑 → 版本 B (今天)
- 用户恢复到版本 A → 这是一个编辑操作"操作 C"
- 用户撤销"操作 C" → 回到版本 B
- 用户重做"操作 C" → 回到版本 A
```

用户可以在"恢复前"和"恢复后"之间自由切换，提供了极高的灵活性。

### 9.3 协作友好的设计

传统的版本恢复（直接修改数据库）在协作场景下有问题：
- 其他协作者可能正在编辑
- 突然的回滚会覆盖他们的编辑
- 没有撤销机制

Outline 的设计解决了这些问题：
- 版本恢复作为一个编辑操作，通过 Y.js 实时同步
- 其他协作者的编辑不会被直接覆盖（CRDT 保证无冲突合并）
- 发起恢复的用户可以撤销恢复操作
- 所有变更都通过 Y.js 的 CRDT 机制保证一致性

### 9.4 双轨设计的权衡

Outline 实际上有两种恢复路径，这是设计上的权衡：

**主路径（UI 触发）**：
- ✅ 可撤销
- ✅ 协作友好
- ✅ 用户体验自然
- ⚠️ 需要经过编辑器和 Y.js 同步

**备用路径（API 触发）**：
- ✅ 直接、快速
- ⚠️ 不可撤销
- ⚠️ 可能覆盖未同步的编辑
- ⚠️ 协作体验不自然

用户界面只使用主路径，备用路径可能用于脚本、迁移等场景。

## 10. 参考代码位置

| 功能模块 | 文件路径 |
|----------|----------|
| 恢复操作触发 | `app/actions/definitions/revisions.tsx:17-45` |
| 恢复执行逻辑 | `app/scenes/Document/components/Document.tsx:108-140` |
| 内容替换核心 | `app/scenes/Document/hooks/useDocumentSave.ts:194-249` |
| 协作编辑器 | `app/scenes/Document/components/MultiplayerEditor.tsx` |
| 协作扩展 | `app/editor/extensions/Multiplayer.ts` |
| 版本历史界面 | `app/scenes/Document/components/History/History.tsx` |
| 版本列表项 | `app/scenes/Document/components/History/RevisionListItem.tsx` |
| 远程事务识别 | `shared/editor/lib/multiplayer.ts:12` |
| 版本模型 | `server/models/Revision.ts` |
| 文档模型 | `server/models/Document.ts` |
| 版本处理器 | `server/queues/processors/RevisionsProcessor.ts` |
| 协作文档更新 | `server/commands/documentCollaborativeUpdater.ts` |
| 持久化扩展 | `server/collaboration/PersistenceExtension.ts` |
| API 更新同步 | `server/collaboration/APIUpdateExtension.ts` |
| 版本信息 API | `server/routes/api/revisions/revisions.ts:29-64` |
| 版本呈现器 | `server/presenters/revision.ts` |
