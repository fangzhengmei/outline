# Outline 评论、用户提及与通知 Fan-out 机制分析

## 1. 概述

Outline 的评论系统采用事件驱动架构，结合异步任务队列实现高效的通知分发（fan-out）机制。本文档详细分析评论创建、用户提及检测和通知批量下发的完整工作流程。

### 核心组件

| 组件 | 路径 | 职责 |
|------|------|------|
| 评论模型 | `server/models/Comment.ts` | 评论数据模型与基础方法 |
| 评论 API | `server/routes/api/comments/comments.ts` | 评论 CRUD 接口 |
| 通知处理器 | `server/queues/processors/NotificationsProcessor.ts` | 事件路由到对应任务 |
| 评论创建通知任务 | `server/queues/tasks/CommentCreatedNotificationsTask.ts` | 评论通知核心 fan-out 逻辑 |
| 群组提及通知任务 | `server/queues/tasks/GroupMentionedInCommentNotificationsTask.ts` | 群组成员批量通知 |
| 通知辅助类 | `server/models/helpers/NotificationHelper.ts` | 接收者选择与权限检查 |
| Prosemirror 辅助类 | `server/models/helpers/ProsemirrorHelper.tsx` | 提及检测与文档解析 |

---

## 2. 评论创建流程

### 2.1 API 层处理

**文件**: `server/routes/api/comments/comments.ts:30-74`

```typescript
router.post(
  "comments.create",
  rateLimiter(RateLimiterStrategy.TwentyFivePerMinute),
  auth(),
  feature(TeamPreference.Commenting),
  validate(T.CommentsCreateSchema),
  transaction(),
  async (ctx: APIContext<T.CommentsCreateReq>) => {
    const { id, documentId, parentCommentId } = ctx.input.body;
    const { user } = ctx.state.auth;
    const { transaction } = ctx.state;

    // 1. 权限校验
    const document = await Document.findByPk(documentId, {
      userId: user.id,
      transaction,
    });
    authorize(user, "comment", document);

    // 2. 文本处理（图片替换为附件）
    const text = ctx.input.body.text
      ? await TextHelper.replaceImagesWithAttachments(
          ctx,
          ctx.input.body.text,
          user
        )
      : undefined;
    const data = text
      ? commentParser.parse(text).toJSON()
      : ctx.input.body.data;

    // 3. 创建评论（触发事件）
    const comment = await Comment.createWithCtx(ctx, {
      id,
      data,
      createdById: user.id,
      documentId,
      parentCommentId,
    });

    ctx.body = {
      data: presentComment(comment),
      policies: presentPolicies(user, [comment]),
    };
  }
);
```

**关键步骤**：
1. **限流**：每分钟最多 25 条评论
2. **认证**：`auth()` 中间件验证 JWT
3. **特性开关**：检查团队是否启用评论功能
4. **权限校验**：`authorize(user, "comment", document)` 检查用户是否有评论权限
5. **图片处理**：将 base64/远程图片替换为本地附件
6. **创建评论**：使用 `createWithCtx` 触发事件钩子

### 2.2 模型层与事件触发

**文件**: `server/models/base/Model.ts:207-237`

评论创建后，通过 Sequelize 钩子触发事件：

```typescript
@AfterCreate
static async afterCreateEvent<T extends Model>(
  model: T,
  context: HookContext
) {
  await this.insertEvent("create", model, context);
}

// 事件构建与调度
protected static async insertEvent<T extends Model>(
  name: string,    // "create"
  model: T,
  context: HookContext
) {
  const namespace = this.eventNamespace ?? this.tableName; // "comments"
  
  const attrs = {
    name: `${namespace}.${context.event.name ?? name}`, // "comments.create"
    modelId: model.id,
    documentId: model.documentId,
    teamId: context.auth?.user.teamId,
    actorId: context.auth?.user?.id,
    authType: context.auth?.type,
    ip: context.ip,
    changes: model.previousChangeset,
    data: context.event.data,
  };

  // 事务提交后调度事件
  if (context.transaction) {
    (context.transaction.parent || context.transaction).afterCommit(() =>
      models.event.schedule(attrs)
    );
  }
}
```

**事件格式**：`{tableName}.{action}` → `comments.create`

**关键设计**：
- 使用 `afterCommit` 确保事件只在事务成功后调度
- 事件包含完整上下文（actorId、teamId、ip 等）
- `changeset` 记录变更前后的数据

---

## 3. 用户提及检测机制

### 3.1 提及数据结构

提及在 Prosemirror 文档中是一种特殊节点类型：

**文件**: `server/models/helpers/ProsemirrorHelper.tsx:57-65`

```typescript
export type MentionAttrs = {
  type: MentionType;           // "user" | "group" | "document"
  label: string;               // 显示名称，如 "@张三"
  modelId: string;             // 被提及的用户/群组/文档 ID
  actorId: string | undefined; // 创建提及的用户 ID
  id: string;                  // 提及实例的唯一 UUID
  href?: string;               // 链接地址
  unfurl?: UnfurlResponse[keyof UnfurlResponse];
};
```

### 3.2 提及 URL 格式

**文件**: `shared/utils/parseMentionUrl.ts:10-28`

提及在 ProseMirror 中通过特殊 URL 格式标识：

```typescript
// 支持两种格式：
// 1. 三段式：mention://{id}/{type}/{modelId}
// 2. 两段式：mention://{type}/{modelId}

const parseMentionUrl = (url: string) => {
  const match3 = url.match(
    /^mention:\/\/([a-z0-9-]+)\/([a-z_]+)\/([a-z0-9-]+)$/
  );
  // 例: mention://abc123/user/def456
  // → { id: "abc123", mentionType: "user", modelId: "def456" }

  const match2 = url.match(/^mention:\/\/([a-z_]+)\/([a-z0-9-]+)$/);
  // 例: mention://user/def456
  // → { mentionType: "user", modelId: "def456" }
};
```

### 3.3 提及检测逻辑

**文件**: `server/models/helpers/ProsemirrorHelper.tsx:117-142`

```typescript
static parseMentions(doc: Node, options?: Partial<MentionAttrs>) {
  const mentions: MentionAttrs[] = [];
  const seenIds = new Set<string>(); // 用于去重

  doc.descendants((node: Node) => {
    // 检查节点类型是否为 "mention"
    if (node.type.name === "mention") {
      // 过滤条件检查
      if (
        !(options?.type && options.type !== node.attrs.type) &&
        !(options?.modelId && options.modelId !== node.attrs.modelId) &&
        !seenIds.has(node.attrs.id) // 去重
      ) {
        seenIds.add(node.attrs.id);
        mentions.push(node.attrs as MentionAttrs);
      }
      return false; // 不再遍历子节点
    }

    if (!node.content.size) {
      return false;
    }
    return true;
  });

  return mentions;
}
```

**检测流程**：
1. **遍历文档**：使用 `doc.descendants()` 深度遍历所有节点
2. **类型过滤**：仅处理 `type === "mention"` 的节点
3. **条件过滤**：
   - 按 `MentionType` 过滤（`user` / `group` / `document`）
   - 按 `modelId` 过滤（可选）
4. **去重**：使用 `seenIds` Set 确保同一提及不被重复处理

**使用示例**：
```typescript
// 仅检测用户提及
const userMentions = ProsemirrorHelper.parseMentions(docNode, {
  type: MentionType.User
});

// 仅检测群组提及
const groupMentions = ProsemirrorHelper.parseMentions(docNode, {
  type: MentionType.Group
});
```

---

## 4. 通知 Fan-out 机制

### 4.1 事件处理流程

```
┌─────────────────┐
│ 评论创建完成    │
│ (comments.create)│
└────────┬────────┘
         │
         ▼
┌──────────────────────────────┐
│  NotificationsProcessor      │
│  监听 events.create 队列     │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│ CommentCreatedNotificationsTask      │
│  (异步后台任务，优先级: Background)   │
└────────┬─────────────────────────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐  ┌────────────────────────────────┐
│直接处理│  │ GroupMentionedInCommentTask   │
│(用户)  │  │ (群组提及 - 独立异步任务)      │
└───┬───┘  └──────────────────┬─────────────┘
    │                         │
    ▼                         ▼
┌──────────────────────────────────────────┐
│           Notification.create()           │
│    (为每个接收者创建通知记录)              │
└────────────────────┬─────────────────────┘
                     │
                     ▼
            ┌────────────────┐
            │ 通知交付渠道    │
            │ (App / Email)  │
            └────────────────┘
```

### 4.2 评论创建通知任务核心逻辑

**文件**: `server/queues/tasks/CommentCreatedNotificationsTask.ts:17-143`

```typescript
export default class CommentCreatedNotificationsTask extends BaseTask<CommentEvent> {
  public async perform(event: CommentEvent) {
    // 1. 加载关联数据
    const [document, comment] = await Promise.all([
      Document.scope("withCollection").findOne({
        where: { id: event.documentId },
      }),
      Comment.findByPk(event.modelId),
    ]);
    if (!document || !comment) {
      return; // 防御性检查
    }

    // 2. 自动订阅：评论者自动订阅文档更新
    await sequelize.transaction(async (transaction) => {
      await subscriptionCreator({
        ctx: createContext({
          user: comment.createdBy,
          authType: event.authType,
          ip: event.ip,
          transaction,
        }),
        documentId: document.id,
        event: SubscriptionType.Document,
        resubscribe: false, // 不强制重新订阅
      });
    });

    // 3. 处理用户提及
    const userMentions = ProsemirrorHelper.parseMentions(
      ProsemirrorHelper.toProsemirror(comment.data),
      { type: MentionType.User }
    );
    const userIdsMentioned: string[] = [];

    for (const mention of userMentions) {
      if (userIdsMentioned.includes(mention.modelId)) {
        continue; // 去重
      }

      const recipient = await User.findByPk(mention.modelId);

      // 权限与订阅检查
      if (
        mention.actorId &&
        recipient &&
        recipient.id !== mention.actorId && // 不通知自己
        recipient.subscribedToEventType(
          NotificationEventType.MentionedInComment
        ) &&
        (await canUserAccessDocument(recipient, document.id)) // 有文档访问权限
      ) {
        await Notification.create({
          event: NotificationEventType.MentionedInComment,
          userId: recipient.id,
          actorId: mention.actorId,
          teamId: document.teamId,
          commentId: comment.id,
          documentId: document.id,
        });
        userIdsMentioned.push(recipient.id);
      }
    }

    // 4. 处理群组提及（异步任务）
    const groupMentions = ProsemirrorHelper.parseMentions(
      ProsemirrorHelper.toProsemirror(comment.data),
      { type: MentionType.Group }
    );

    const mentionedGroup: string[] = [];
    for (const group of groupMentions) {
      if (mentionedGroup.includes(group.modelId)) {
        continue;
      }

      // 检查群组是否禁用提及
      const groupModel = await Group.findByPk(group.modelId);
      if (groupModel?.disableMentions) {
        continue;
      }

      // 调度独立任务处理群组成员
      await new GroupMentionedInCommentNotificationsTask().schedule({
        ...event,
        data: {
          groupId: group.modelId,
          actorId: group.actorId ?? event.actorId,
        },
      });

      mentionedGroup.push(group.modelId);
    }

    // 5. 处理订阅者通知（排除已提及用户）
    const recipients = (
      await NotificationHelper.getCommentNotificationRecipients(
        document,
        comment,
        comment.createdById
      )
    ).filter((recipient) => !userIdsMentioned.includes(recipient.id)); // 去重

    // 批量创建通知
    await sequelize.transaction(async (transaction) => {
      for (const recipient of recipients) {
        await Notification.create(
          {
            event: NotificationEventType.CreateComment,
            userId: recipient.id,
            actorId: comment.createdById,
            teamId: document.teamId,
            commentId: comment.id,
            documentId: document.id,
          },
          { transaction }
        );
      }
    });
  }
}
```

**任务配置**：
```typescript
public get options() {
  return {
    priority: TaskPriority.Background, // 后台低优先级
  };
}
```

### 4.3 通知接收者选择逻辑

**文件**: `server/models/helpers/NotificationHelper.ts:53-147`

```typescript
public static getCommentNotificationRecipients = async (
  document: Document,
  comment: Comment,
  actorId: string
): Promise<User[]> => {
  let recipients: User[];

  // 区分回复评论和顶级评论
  if (comment.parentCommentId) {
    // ========== 回复评论：通知线程参与者 ==========
    const contextComments = await Comment.findAll({
      attributes: ["createdById", "data"],
      where: {
        [Op.or]: [
          { id: comment.parentCommentId },      // 父评论
          { parentCommentId: comment.parentCommentId }, // 同线程其他回复
        ],
      },
    });

    // 收集线程中的用户
    const createdUserIdsInThread = contextComments.map((c) => c.createdById);
    const mentionedUserIdsInThread = contextComments
      .flatMap((c) =>
        ProsemirrorHelper.parseMentions(
          ProsemirrorHelper.toProsemirror(c.data),
          { type: MentionType.User }
        )
      )
      .map((mention) => mention.modelId);

    // 合并去重 + 排除动作执行者
    const userIdsInThread = uniq([
      ...createdUserIdsInThread,
      ...mentionedUserIdsInThread,
    ]).filter((userId) => userId !== actorId);

    recipients = await User.findAll({
      where: {
        id: { [Op.in]: userIdsInThread },
        teamId: document.teamId,
      },
    });

    // 过滤订阅了该类型通知的用户
    recipients = recipients.filter((recipient) =>
      recipient.subscribedToEventType(NotificationEventType.CreateComment)
    );
  } else {
    // ========== 顶级评论：通知文档订阅者 ==========
    recipients = await this.getDocumentNotificationRecipients({
      document,
      notificationType: NotificationEventType.CreateComment,
      actorId,
      disableAccessCheck: true, // 后续统一检查
    });
  }

  // ========== 统一过滤条件 ==========
  const filtered: User[] = [];

  for (const recipient of recipients) {
    // 1. 排除被暂停的用户
    if (recipient.isSuspended) {
      continue;
    }

    // 2. 智能抑制：如果用户已查看文档，则不通知
    const view = await View.findOne({
      where: {
        userId: recipient.id,
        documentId: document.id,
        updatedAt: {
          [Op.gt]: comment.createdAt, // 查看时间晚于评论创建时间
        },
      },
    });

    if (view) {
      Logger.info(
        "processor",
        `suppressing notification to ${recipient.id} because doc viewed`
      );
      continue;
    }

    // 3. 权限检查：确保有文档访问权限
    if (await canUserAccessDocument(recipient, document.id)) {
      filtered.push(recipient);
    }
  }

  return filtered;
};
```

**文档订阅者获取逻辑**：
```typescript
static getDocumentNotificationRecipients = async ({
  document,
  notificationType,
  actorId,
  disableAccessCheck = false,
}) => {
  // 集合级订阅 + 文档级订阅
  const [collectionSubs, documentSubs] = await Promise.all([
    document.collectionId
      ? Subscription.findAll({
          where: {
            userId: { [Op.ne]: actorId },
            event: SubscriptionType.Document,
            collectionId: document.collectionId,
          },
          include: [{ association: "user", required: true }],
        })
      : [],
    Subscription.findAll({
      where: {
        userId: { [Op.ne]: actorId },
        event: SubscriptionType.Document,
        documentId: document.id,
      },
      include: [{ association: "user", required: true }],
    }),
  ]);

  // 去重
  return uniqBy(
    [...collectionSubs, ...documentSubs].map((s) => s.user),
    (user) => user.id
  );
};
```

### 4.4 群组提及批量处理

**文件**: `server/queues/tasks/GroupMentionedInCommentNotificationsTask.ts:15-84`

群组可能包含大量成员，因此采用独立异步任务 + 批量处理模式：

```typescript
export default class GroupMentionedInCommentNotificationsTask extends BaseTask<GroupMentionEvent> {
  public async perform(event: GroupMentionEvent) {
    const { groupId, actorId } = event.data;

    // 双重检查：群组是否禁用提及
    const groupModel = await Group.findByPk(groupId);
    if (groupModel?.disableMentions) {
      return;
    }

    // ========== 批量处理 ==========
    await GroupUser.findAllInBatches<GroupUser>(
      {
        where: {
          groupId,
          userId: { [Op.ne]: actorId }, // 排除动作执行者
        },
        order: [["permission", "ASC"]],
        batchLimit: 10, // 每批 10 个用户
      },
      async (groupUsers) => {
        // 1. 批量加载用户数据（减少数据库查询）
        const userIds = groupUsers.map((gu) => gu.userId);
        const users = await User.findAll({
          where: { id: userIds },
        });
        const userMap = new Map(users.map((u) => [u.id, u]));

        // 2. 并行处理当前批次
        await Promise.all(
          groupUsers.map(async (groupUser) => {
            const recipient = userMap.get(groupUser.userId);
            if (
              recipient &&
              recipient.subscribedToEventType(
                NotificationEventType.GroupMentionedInComment
              ) &&
              (await canUserAccessDocument(recipient, event.documentId))
            ) {
              await Notification.create({
                event: NotificationEventType.GroupMentionedInComment,
                groupId,
                userId: recipient.id,
                actorId,
                teamId: event.teamId,
                documentId: event.documentId,
                commentId: event.modelId,
              });
            }
          })
        );
      }
    );
  }
}
```

**批量处理优势**：
1. **内存优化**：不会一次性加载所有群组成员到内存
2. **数据库保护**：每批 10 个查询，避免连接池耗尽
3. **并行处理**：批次内使用 `Promise.all` 提高效率
4. **失败隔离**：某批次失败不影响其他批次

### 4.5 评论更新时的提及处理

**文件**: `server/queues/tasks/CommentUpdatedNotificationsTask.ts:11-174`

评论更新时只处理**新增**的提及，避免重复通知：

```typescript
// API 层计算新增提及
// server/routes/api/comments/comments.ts:246-275
if (data !== undefined) {
  // 计算差集
  const existingMentionIds = ProsemirrorHelper.parseMentions(
    ProsemirrorHelper.toProsemirror(comment.data),
    { type: MentionType.User }
  ).map((mention) => mention.id);
  
  const updatedMentionIds = ProsemirrorHelper.parseMentions(
    ProsemirrorHelper.toProsemirror(data),
    { type: MentionType.User }
  ).map((mention) => mention.id);

  // 新增的提及 ID
  newMentionIds = difference(updatedMentionIds, existingMentionIds);
  
  comment.data = data;
}

// 保存时传递新增提及
await comment.saveWithCtx(ctx, undefined, { data: { newMentionIds } });
```

**任务层处理**：
```typescript
private async handleMentionedComment(event: CommentUpdateEvent) {
  const newMentionIds = event.data?.newMentionIds;
  if (!newMentionIds) {
    return; // 没有新增提及，不处理
  }

  // 仅过滤新增的提及
  const mentions = ProsemirrorHelper.parseMentions(
    ProsemirrorHelper.toProsemirror(comment.data),
    { type: MentionType.User }
  ).filter((mention) => newMentionIds.includes(mention.id));
  
  // ... 后续处理与创建时类似
}
```

---

## 5. 通知模型与事件类型

### 5.1 通知数据模型

**文件**: `server/models/Notification.ts:100-291`

```typescript
@Table({ tableName: "notifications", modelName: "notification" })
class Notification extends Model<...> {
  @PrimaryKey
  @Column(DataType.UUID)
  id: string;

  // 状态字段
  @Column
  emailedAt?: Date | null;   // 邮件发送时间
  @Column
  viewedAt: Date | null;     // 查看时间
  @Column
  archivedAt: Date | null;   // 归档时间
  
  // 事件类型
  @Column(DataType.STRING)
  event: NotificationEventType;
  
  // 关联字段
  @Column(DataType.UUID)
  userId: string;        // 接收者
  @Column(DataType.UUID)
  actorId: string;       // 执行者
  @Column(DataType.UUID)
  teamId: string;
  @Column(DataType.UUID)
  commentId: string;
  @Column(DataType.UUID)
  documentId: string;
  @Column(DataType.UUID)
  groupId: string;       // 群组提及专用
}
```

### 5.2 通知事件类型

**文件**: `shared/types.ts:465-483`

```typescript
export enum NotificationEventType {
  // 文档相关
  PublishDocument = "documents.publish",
  UpdateDocument = "documents.update",
  AddUserToDocument = "documents.add_user",
  
  // 评论相关
  CreateComment = "comments.create",           // 新评论通知
  ResolveComment = "comments.resolve",         // 评论解决通知
  
  // 提及相关
  MentionedInComment = "comments.mentioned",           // 用户被@
  GroupMentionedInComment = "comments.group_mentioned", // 群组被@
  
  // 其他...
}
```

### 5.3 通知创建后触发事件

**文件**: `server/models/Notification.ts:199-222`

通知记录创建后会触发 `notifications.create` 事件，用于推送通知到前端：

```typescript
@AfterCreate
static async createEvent(
  model: Notification,
  options: SaveOptions<...>
) {
  const params = {
    name: "notifications.create",
    userId: model.userId,
    modelId: model.id,
    teamId: model.teamId,
    // ... 其他关联字段
  };

  if (options.transaction) {
    options.transaction.afterCommit(() => void Event.schedule(params));
    return;
  }
  await Event.schedule(params);
}
```

---

## 6. 关键设计模式与优化策略

### 6.1 Fan-out 模式总结

| 场景 | Fan-out 策略 | 处理位置 |
|------|-------------|----------|
| 用户提及 | 直接遍历，一对一创建 | `CommentCreatedNotificationsTask` |
| 群组提及 | 异步任务 + 批量处理 (10/批) | `GroupMentionedInCommentTask` |
| 订阅者通知 | 回复→线程参与者 / 顶级→文档订阅者 | `NotificationHelper` |

### 6.2 去重机制

系统在多个层面防止重复通知：

1. **用户提及去重**：
   ```typescript
   const userIdsMentioned: string[] = [];
   for (const mention of mentions) {
     if (userIdsMentioned.includes(mention.modelId)) continue;
     // 创建通知...
     userIdsMentioned.push(recipient.id);
   }
   ```

2. **订阅者与提及去重**：
   ```typescript
   const recipients = (await NotificationHelper.getCommentNotificationRecipients(...))
     .filter((recipient) => !userIdsMentioned.includes(recipient.id));
   ```

3. **Prosemirror 解析去重**：
   ```typescript
   const seenIds = new Set<string>();
   if (!seenIds.has(node.attrs.id)) {
     seenIds.add(node.attrs.id);
     mentions.push(node.attrs);
   }
   ```

### 6.3 权限检查链路

每个通知接收者都经过多层检查：

```
┌────────────────────────────────────────────────────┐
│              权限检查流程                           │
├────────────────────────────────────────────────────┤
│  1. 不是动作执行者                                  │
│     recipient.id !== actorId                        │
├────────────────────────────────────────────────────┤
│  2. 用户未被暂停                                    │
│     !recipient.isSuspended                          │
├────────────────────────────────────────────────────┤
│  3. 订阅了该通知类型                                │
│     recipient.subscribedToEventType(eventType)      │
├────────────────────────────────────────────────────┤
│  4. 有文档访问权限                                  │
│     canUserAccessDocument(recipient, documentId)    │
├────────────────────────────────────────────────────┤
│  5. 评论创建后未查看文档（智能抑制）                │
│     !View.findOne({ updatedAt > comment.createdAt })│
└────────────────────────────────────────────────────┘
```

### 6.4 异步任务与事务一致性

**关键设计**：
```typescript
// 事务提交后才调度事件
if (context.transaction) {
  context.transaction.afterCommit(() =>
    models.event.schedule(attrs)
  );
}
```

**优势**：
- 防止回滚后仍发送通知
- 确保数据一致性
- 符合"先持久化，后通知"原则

### 6.5 性能优化点

1. **批量查询**：
   - 群组处理时批量加载用户：`User.findAll({ where: { id: userIds } })`
   - 避免 N+1 查询问题

2. **智能抑制**：
   - 通过 `View` 记录检查用户是否已查看文档
   - 避免发送"过时"通知

3. **异步解耦**：
   - 群组提及使用独立任务
   - 不阻塞主通知流程

4. **数据库索引**：
   - `Subscriptions` 表的查询条件都有索引
   - `Views` 表按 `(userId, documentId)` 查询

---

## 7. 数据流图

### 7.1 完整评论创建与通知分发流程

```
┌──────────────┐
│   前端用户    │
└──────┬───────┘
       │ POST /comments.create
       ▼
┌──────────────────────────────────────────────────────┐
│                    API 层                              │
│  server/routes/api/comments/comments.ts              │
├──────────────────────────────────────────────────────┤
│  1. auth() - JWT 认证                                 │
│  2. authorize() - 评论权限检查                        │
│  3. TextHelper.replaceImagesWithAttachments()        │
│  4. Comment.createWithCtx()                           │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│                   模型层                               │
│  server/models/base/Model.ts                         │
├──────────────────────────────────────────────────────┤
│  @AfterCreate → afterCreateEvent()                   │
│    → insertEvent("create", model, context)           │
│      → Event.schedule({ name: "comments.create" })   │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼ (Redis 队列)
┌──────────────────────────────────────────────────────┐
│              事件处理器                                │
│  server/queues/processors/NotificationsProcessor.ts  │
├──────────────────────────────────────────────────────┤
│  case "comments.create":                              │
│    → new CommentCreatedNotificationsTask().schedule() │
└──────────────────────┬───────────────────────────────┘
                       │ (异步后台任务)
                       ▼
┌──────────────────────────────────────────────────────┐
│         CommentCreatedNotificationsTask               │
│  server/queues/tasks/CommentCreatedNotificationsTask.ts│
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────┐ │
│  │ 用户提及处理  │ → │ 群组提及处理  │ → │订阅者通知│ │
│  └──────────────┘   └──────┬───────┘   └──────────┘ │
│                            │                          │
│                            ▼                          │
│              ┌─────────────────────────┐              │
│              │ GroupMentionedInComment │              │
│              │    NotificationsTask    │              │
│              │   (独立异步任务)         │              │
│              └────────────┬────────────┘              │
└───────────────────────────┼───────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │   Notification.create() │
              │  (为每个接收者创建记录)  │
              └────────────┬────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │  @AfterCreate 钩子      │
              │  → "notifications.create"│
              │  → 推送至 WebSocket      │
              └─────────────────────────┘
```

### 7.2 提及检测数据流

```
┌─────────────────────────────────────────────────────────┐
│                   评论内容 (Prosemirror JSON)            │
│  {                                                        │
│    type: "doc",                                          │
│    content: [                                             │
│      { type: "paragraph", content: [                     │
│        { type: "text", text: "请查看 " },                │
│        {                                                   │
│          type: "mention",                                 │
│          attrs: {                                         │
│            type: "user",                                  │
│            modelId: "user-123",                          │
│            actorId: "user-456",                          │
│            label: "@张三",                                │
│            id: "mention-abc"                              │
│          }                                                 │
│        },                                                  │
│        { type: "text", text: " 的建议" }                  │
│      ]}                                                    │
│    ]                                                       │
│  }                                                         │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼ ProsemirrorHelper.toProsemirror()
┌─────────────────────────────────────────────────────────┐
│              ProseMirror Node (内存结构)                 │
│  doc → paragraph → [text, mention, text]                │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼ ProsemirrorHelper.parseMentions()
┌─────────────────────────────────────────────────────────┐
│                   检测逻辑                                │
│  doc.descendants((node) => {                            │
│    if (node.type.name === "mention") {                  │
│      // 按类型过滤: MentionType.User / Group            │
│      // 去重: seenIds Set                                │
│      mentions.push(node.attrs);                          │
│    }                                                       │
│  })                                                        │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                    输出结果                               │
│  [                                                        │
│    {                                                      │
│      type: "user",                                       │
│      modelId: "user-123",  // 被提及用户 ID             │
│      actorId: "user-456",  // 评论作者 ID               │
│      id: "mention-abc",     // 提及实例 ID              │
│      label: "@张三"                                      │
│    }                                                       │
│  ]                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 8. 测试验证

### 8.1 群组提及批量处理测试

**文件**: `server/queues/tasks/GroupMentionedInCommentNotificationsTask.test.ts:224-269`

```typescript
it("should handle large groups with batching", async () => {
  const spy = jest.spyOn(Notification, "create");
  const actor = await buildUser();
  const document = await buildDocument({ teamId: actor.teamId });
  const comment = await buildComment({
    userId: actor.id,
    documentId: document.id,
  });

  const group = await buildGroup({ teamId: actor.teamId });

  // 创建 25 个成员（批大小为 10，需 3 批）
  const members = [];
  for (let i = 0; i < 25; i++) {
    const member = await buildUser({ teamId: actor.teamId });
    await buildGroupUser({
      groupId: group.id,
      userId: member.id,
    });
    member.setNotificationEventType(
      NotificationEventType.GroupMentionedInComment
    );
    await member.save();
    members.push(member);
  }

  const task = new GroupMentionedInCommentNotificationsTask();
  await task.perform({
    name: "comments.create",
    modelId: comment.id,
    documentId: document.id,
    teamId: actor.teamId,
    actorId: actor.id,
    ip: "127.0.0.1",
    data: {
      groupId: group.id,
      actorId: actor.id,
    },
  });

  expect(spy).toHaveBeenCalledTimes(25); // 所有成员都收到通知
});
```

### 8.2 其他关键测试场景

| 测试场景 | 预期行为 |
|---------|---------|
| 群组禁用提及 | 不发送任何通知 |
| 用户未订阅通知类型 | 不发送通知 |
| 动作执行者也是群组成员 | 不给自己发通知 |
| 用户无文档访问权限 | 不发送通知 |

---

## 9. 总结

### 9.1 核心设计原则

1. **事件驱动架构**：通过 `comments.create` 事件解耦评论创建与通知分发
2. **异步任务处理**：使用 Bull 队列处理通知，不阻塞主请求
3. **批量处理优化**：群组提及采用 10 个/批的处理方式，保护数据库
4. **多层去重机制**：在提及检测、任务处理、接收者选择等层面去重
5. **细粒度权限控制**：每用户级别的权限与订阅检查
6. **智能通知抑制**：基于 `View` 记录避免发送过时通知

### 9.2 关键代码位置索引

| 功能模块 | 文件路径 | 关键函数/方法 |
|---------|---------|--------------|
| 评论创建 API | `server/routes/api/comments/comments.ts` | `router.post("comments.create", ...)` |
| 事件触发 | `server/models/base/Model.ts` | `insertEvent()`, `afterCreateEvent()` |
| 通知处理器 | `server/queues/processors/NotificationsProcessor.ts` | `commentCreated()` |
| 评论通知任务 | `server/queues/tasks/CommentCreatedNotificationsTask.ts` | `perform()` |
| 群组提及任务 | `server/queues/tasks/GroupMentionedInCommentNotificationsTask.ts` | `perform()` |
| 提及检测 | `server/models/helpers/ProsemirrorHelper.tsx` | `parseMentions()` |
| 接收者选择 | `server/models/helpers/NotificationHelper.ts` | `getCommentNotificationRecipients()` |
| 通知模型 | `server/models/Notification.ts` | `@AfterCreate createEvent()` |

### 9.3 扩展建议

1. **监控指标**：
   - 通知 fan-out 延迟
   - 群组提及批处理耗时
   - 通知送达率（emailedAt/viewedAt 比率）

2. **潜在优化**：
   - `Notification.create()` 可考虑批量插入（当前为循环单条）
   - 大群组可考虑进一步分片处理
   - 缓存用户订阅状态减少数据库查询

3. **故障处理**：
   - 任务失败重试机制（Bull 内置）
   - 死信队列处理
   - 通知积压告警

---

*文档生成日期: 2026-05-05*
