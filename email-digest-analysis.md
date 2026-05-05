# Outline 邮件摘要和通知系统分析报告

## 1. 系统架构概览

Outline 的邮件通知系统采用**事件驱动 + 异步队列**的架构设计，主要由以下核心组件构成：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              业务层 (API/Routes)                               │
│  - 用户操作 → 触发事件 (Event.createFromContext)                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          事件存储层 (PostgreSQL)                                │
│  - events 表：记录所有事件日志                                                  │
│  - notifications 表：存储待发送的通知                                           │
│  - share_subscriptions 表：公开分享的文档订阅                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          队列层 (Bull + Redis)                                 │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐          │
│  │ globalEventQueue │  │processorEventQue │  │   taskQueue      │          │
│  │   (事件分发)      │  │   ue (处理器)    │  │   (任务执行)     │          │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          工作进程层 (Worker)                                    │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ NotificationsProcessor: 事件 → 通知 (Notification.create)                │ │
│  │ EmailsProcessor: 通知 → 邮件模板调度                                        │ │
│  │ EmailTask: 模板渲染 → SMTP 发送                                            │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          邮件投递层 (Nodemailer + SMTP)                        │
│  - SMTP 服务器配置 (SMTP_HOST, SMTP_PORT, SMTP_USERNAME 等)                   │
│  - 支持知名邮件服务 (SMTP_SERVICE: Gmail, Outlook 等)                          │
│  - 开发环境: ethereal.email 测试服务                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 通知生成机制

### 2.1 事件类型定义

通知生成始于系统事件，Outline 支持多种事件类型，在 `NotificationEventType` 中定义：

```typescript
// 通知事件类型
PublishDocument           // 文档发布
UpdateDocument            // 文档更新
AddUserToDocument         // 添加用户到文档
AddUserToCollection       // 添加用户到集合
CreateComment             // 创建评论
UpdateComment             // 更新评论
ResolveComment            // 解决评论
MentionedInDocument       // 文档中提及用户
MentionedInComment        // 评论中提及用户
GroupMentionedInDocument  // 文档中提及群组
GroupMentionedInComment   // 评论中提及群组
CreateCollection          // 创建集合
```

### 2.2 事件触发流程

#### 2.2.1 事件创建

事件通过 `Event.createFromContext` 方法创建，例如在文档发布时：

**文件**: `server/models/Event.ts:172-194`

```typescript
static createFromContext(
  ctx: Context | APIContext,
  attributes: Omit<Partial<Event>, "ip" | "teamId" | "actorId"> = {},
  defaultAttributes: Pick<Partial<Event>, "ip" | "teamId" | "actorId"> = {},
  options?: CreateOptions<InferAttributes<Event>>
) {
  const user = ctx.state.auth?.user;
  const authType = ctx.state.auth?.type;

  return this.create(
    {
      ...attributes,
      actorId: user?.id || defaultAttributes.actorId,
      teamId: user?.teamId || defaultAttributes.teamId,
      ip: ctx.request?.ip || defaultAttributes.ip,
      authType,
    },
    {
      transaction: ctx.state.transaction,
      ...options,
    }
  );
}
```

#### 2.2.2 事件入队

事件保存后通过 `@AfterSave` 钩子自动入队：

**文件**: `server/models/Event.ts:84-99`

```typescript
@AfterSave
static async enqueue(
  model: Event,
  options: SaveOptions<InferAttributes<Event>>
) {
  if (options.transaction) {
    // 事务提交后入队，确保数据一致性
    (options.transaction.parent || options.transaction).afterCommit(
      () => void globalEventQueue().add(model)
    );
    return;
  }
  void globalEventQueue().add(model);
}
```

### 2.3 通知处理器

`NotificationsProcessor` 负责将事件转换为通知任务：

**文件**: `server/queues/processors/NotificationsProcessor.ts:25-63`

```typescript
export default class NotificationsProcessor extends BaseProcessor {
  // 订阅的事件类型
  static applicableEvents: Event["name"][] = [
    "documents.publish",
    "documents.add_user",
    "documents.add_group",
    "revisions.create",
    "collections.create",
    "collections.add_user",
    "comments.create",
    "comments.update",
    "comments.add_reaction",
    "comments.remove_reaction",
  ];

  async perform(event: Event) {
    switch (event.name) {
      case "documents.publish":
        return this.documentPublished(event);
      case "documents.add_user":
        return this.documentAddUser(event);
      // ... 其他事件处理
    }
  }
}
```

### 2.4 通知任务示例：文档发布

以 `DocumentPublishedNotificationsTask` 为例，展示通知生成逻辑：

**文件**: `server/queues/tasks/DocumentPublishedNotificationsTask.ts:10-131`

```typescript
public async perform(event: DocumentEvent) {
  const document = await Document.findByPk(event.documentId, {
    includeState: true,
  });
  if (!document) {
    return;
  }

  await createSubscriptionsForDocument(document, event);

  // 1. 处理文档中的用户提及
  const mentions = DocumentHelper.parseMentions(document, {
    type: MentionType.User,
  });
  
  for (const mention of mentions) {
    const recipient = await User.findByPk(mention.modelId);
    if (
      recipient &&
      recipient.id !== mention.actorId &&
      recipient.subscribedToEventType(
        NotificationEventType.MentionedInDocument
      ) &&
      (await canUserAccessDocument(recipient, document.id))
    ) {
      await Notification.create({
        event: NotificationEventType.MentionedInDocument,
        userId: recipient.id,
        actorId: mention.actorId,
        teamId: document.teamId,
        documentId: document.id,
      });
    }
  }

  // 2. 处理群组提及
  const groupMentions = DocumentHelper.parseMentions(document, {
    type: MentionType.Group,
  });
  // ... 类似逻辑处理群组提及

  // 3. 获取文档订阅者并发送发布通知
  const recipients = (
    await NotificationHelper.getDocumentNotificationRecipients({
      document,
      notificationType: NotificationEventType.PublishDocument,
      actorId: document.lastModifiedById,
    })
  ).filter((recipient) => !userIdsMentioned.includes(recipient.id));

  for (const recipient of recipients) {
    await Notification.create({
      event: NotificationEventType.PublishDocument,
      userId: recipient.id,
      actorId: document.updatedBy.id,
      teamId: document.teamId,
      documentId: document.id,
    });
  }
}
```

### 2.5 通知数据模型

**文件**: `server/models/Notification.ts:95-198`

```typescript
@Table({
  tableName: "notifications",
  modelName: "notification",
  updatedAt: false,
})
class Notification extends Model {
  @IsUUID(4)
  @PrimaryKey
  @Default(DataType.UUIDV4)
  @Column(DataType.UUID)
  id: string;

  @AllowNull
  @Column
  emailedAt?: Date | null;  // 邮件发送时间

  @AllowNull
  @Column
  viewedAt: Date | null;    // 通知查看时间

  @AllowNull
  @Column
  archivedAt: Date | null;  // 通知归档时间

  @CreatedAt
  createdAt: Date;

  @Column(DataType.JSONB)
  data: NotificationData | null;  // 通知元数据

  @Column(DataType.STRING)
  event: NotificationEventType;    // 通知事件类型

  // 外键关联
  @ForeignKey(() => User)
  @Column(DataType.UUID)
  userId: string;         // 接收用户

  @ForeignKey(() => User)
  @Column(DataType.UUID)
  actorId: string;        // 触发用户

  @ForeignKey(() => Document)
  @Column(DataType.UUID)
  documentId: string;     // 关联文档

  @ForeignKey(() => Comment)
  @Column(DataType.UUID)
  commentId: string;      // 关联评论

  @ForeignKey(() => Collection)
  @Column(DataType.UUID)
  collectionId: string;   // 关联集合

  @ForeignKey(() => Team)
  @Column(DataType.UUID)
  teamId: string;         // 团队ID
}
```

### 2.6 用户通知设置

用户可以通过 `notificationSettings` 字段配置接收哪些类型的通知：

**文件**: `server/models/User.ts:334-358`

```typescript
/**
 * 设置用户的通知偏好
 */
public setNotificationEventType = (
  type: NotificationEventType,
  value = true
) => {
  this.notificationSettings = {
    ...this.notificationSettings,
    [type]: value,
  };
};

/**
 * 检查用户是否订阅了某类通知
 * 考虑系统默认值
 */
public subscribedToEventType = (type: NotificationEventType) =>
  this.notificationSettings[type] ?? NotificationEventDefaults[type] ?? false;
```

---

## 3. 调度机制

### 3.1 队列系统架构

Outline 使用 **Bull** 队列库，基于 **Redis** 实现异步任务调度。系统包含四个独立队列：

**文件**: `server/queues/index.ts:1-54`

```typescript
// 全局事件队列 - 事件分发入口
export const globalEventQueue = () => {
  if (!cachedGlobalEventQueue) {
    cachedGlobalEventQueue = createQueue("globalEvents", {
      attempts: 5,
      backoff: {
        type: "exponential",
        delay: Second.ms,  // 1秒
      },
    });
  }
  return cachedGlobalEventQueue;
};

// 处理器事件队列 - 处理器执行
export const processorEventQueue = () => {
  if (!cachedProcessorEventQueue) {
    cachedProcessorEventQueue = createQueue("processorEvents", {
      attempts: 5,
      backoff: {
        type: "exponential",
        delay: 10 * Second.ms,  // 10秒
      },
    });
  }
  return cachedProcessorEventQueue;
};

// WebSocket 队列 - 实时通知
export const websocketQueue = () => {
  if (!cachedWebsocketQueue) {
    cachedWebsocketQueue = createQueue("websockets", {
      timeout: 10 * Second.ms,
    });
  }
  return cachedWebsocketQueue;
};

// 任务队列 - 异步任务执行
export const taskQueue = () => {
  if (!cachedTaskQueue) {
    cachedTaskQueue = createQueue("tasks", {
      attempts: 5,
      backoff: {
        type: "exponential",
        delay: 10 * Second.ms,
      },
    });
  }
  return cachedTaskQueue;
};
```

### 3.2 队列实现

**文件**: `server/queues/queue.ts:10-68`

```typescript
export function createQueue(
  name: string,
  defaultJobOptions?: Partial<Queue.JobOptions>
) {
  const prefix = `queue.${snakeCase(name)}`;

  const queue = new Queue(name, {
    // 复用 Redis 连接
    createClient(type) {
      switch (type) {
        case "client":
          return Redis.defaultClient;
        case "subscriber":
          return Redis.defaultSubscriber;
        case "bclient":
          return new Redis(env.REDIS_URL, {
            maxRetriesPerRequest: null,
            connectionNameSuffix: "bull",
          });
        default:
          throw new Error(`Unexpected connection type: ${String(type)}`);
      }
    },
    defaultJobOptions: {
      removeOnComplete: true,  // 完成后自动清理
      removeOnFail: true,      // 失败后自动清理
      ...defaultJobOptions,
    },
  });

  // 监控指标
  queue.on("stalled", () => {
    Metrics.increment(`${prefix}.jobs.stalled`);
  });
  queue.on("completed", () => {
    Metrics.increment(`${prefix}.jobs.completed`);
  });
  queue.on("error", () => {
    Metrics.increment(`${prefix}.jobs.errored`);
  });
  queue.on("failed", () => {
    Metrics.increment(`${prefix}.jobs.failed`);
  });

  return queue;
}
```

### 3.3 工作进程

**文件**: `server/services/worker.ts:17-179`

工作进程负责消费队列中的任务：

```typescript
export default async function init() {
  // 1. 全局事件队列处理 - 事件分发
  globalEventQueue()
    .process(
      env.WORKER_CONCURRENCY_EVENTS,  // 并发数
      traceFunction({ ... })(async function (job) {
        const event = job.data as Event;
        
        // 遍历所有处理器，分发事件
        for (const name in processors) {
          const ProcessorClass = processors[name];
          
          if (name === "WebsocketsProcessor") {
            // WebSocket 特殊处理
            await websocketQueue().add(job.data);
          } else if (
            ProcessorClass.applicableEvents.includes(event.name) ||
            ProcessorClass.applicableEvents.includes("*")
          ) {
            // 其他处理器加入处理器队列
            await processorEventQueue().add({ event, name });
          }
        }
      })
    );

  // 2. 处理器事件队列处理
  processorEventQueue()
    .process(
      env.WORKER_CONCURRENCY_EVENTS,
      traceFunction({ ... })(async function (job) {
        const { event, name } = job.data;
        const ProcessorClass = processors[name];
        
        // @ts-expect-error
        const processor = new ProcessorClass();
        
        if (processor.perform) {
          try {
            await processor.perform(event);
          } catch (err) {
            // 最后一次尝试失败时调用 onFailed
            if (job.attemptsMade + 1 >= (job.opts.attempts || 1)) {
              await processor.onFailed(event).catch();
            }
            throw err;
          }
        }
      })
    );

  // 3. 任务队列处理
  taskQueue()
    .process(
      env.WORKER_CONCURRENCY_TASKS,
      traceFunction({ ... })(async function (job) {
        const { name, props } = job.data;
        const TaskClass = tasks[name];
        
        // @ts-expect-error
        const task = new TaskClass();
        
        try {
          return await task.perform(props);
        } catch (err) {
          if (job.attemptsMade + 1 >= (job.opts.attempts || 1)) {
            await task.onFailed(props).catch();
          }
          throw err;
        }
      })
    );
}
```

### 3.4 定时任务 (Cron)

Outline 支持两种方式触发定时任务：

#### 3.4.1 内部定时调度

**文件**: `server/services/cron.ts:1-36`

```typescript
export default function init() {
  async function run(schedule: TaskInterval) {
    const partition = {
      partitionIndex: 0,
      partitionCount: 1,
    };

    // 遍历所有任务，执行对应间隔的 CronTask
    for (const name in tasks) {
      const TaskClass = tasks[name];
      if (!(TaskClass.prototype instanceof CronTask)) {
        continue;
      }

      // @ts-expect-error
      const taskInstance = new TaskClass() as CronTask;

      if (taskInstance.cron.interval === schedule) {
        await taskInstance.schedule({ limit: 10000, partition });
      }
    }
  }

  // 每日定时任务
  setInterval(() => void run(TaskInterval.Day), Day.ms);
  // 每小时定时任务
  setInterval(() => void run(TaskInterval.Hour), Hour.ms);

  // 启动后立即执行一次（延迟 5 秒）
  setTimeout(() => {
    void run(TaskInterval.Day);
    void run(TaskInterval.Hour);
  }, 5 * Second.ms);
}
```

#### 3.4.2 HTTP API 触发

**文件**: `server/routes/api/cron/cron.ts:16-95`

```typescript
const cronHandler = async (ctx: APIContext<T.CronSchemaReq>) => {
  const period = Object.values(TaskInterval).includes(
    ctx.params.period as TaskInterval
  )
    ? (ctx.params.period as TaskInterval)
    : TaskInterval.Day;
  
  // 验证 UTILS_SECRET
  const token = (ctx.input.body.token ?? ctx.input.query.token) as string;
  if (!safeEqual(env.UTILS_SECRET, token)) {
    throw AuthenticationError("Invalid secret token");
  }

  // 执行对应间隔的 CronTask
  for (const name in tasks) {
    const TaskClass = tasks[name];
    if (!(TaskClass.prototype instanceof CronTask)) {
      continue;
    }

    // @ts-expect-error
    const taskInstance = new TaskClass() as CronTask;
    const cronConfig = taskInstance.cron;

    if (cronConfig.interval === period) {
      // 支持任务分区和时间窗口错开
      const taskDelay = CronTask.getStaggerDelay(name, cronConfig.interval);
      await taskInstance.schedule(
        { limit, partition: { partitionIndex: 0, partitionCount: 1 } },
        { delay: taskDelay }
      );
    }
  }

  ctx.body = { success: true };
};

router.get("cron.:period", validate(T.CronSchema), cronHandler);
router.post("cron.:period", validate(T.CronSchema), cronHandler);
```

#### 3.4.3 CronTask 基类

**文件**: `server/queues/tasks/base/CronTask.ts:6-195`

```typescript
export enum TaskInterval {
  Day = "daily",
  Hour = "hourly",
}

// 任务错开窗口，避免同时开始大量数据库操作
const staggerWindows: Record<TaskInterval, number> = {
  [TaskInterval.Hour]: 10 * Minute.ms,  // 10分钟
  [TaskInterval.Day]: 30 * Minute.ms,    // 30分钟
};

export abstract class CronTask extends BaseTask<Props> {
  /** 定时任务配置 */
  public abstract get cron(): TaskSchedule;

  /**
   * 计算任务的延迟时间，基于任务名称和间隔
   * 确保不同任务在错开窗口内均匀分布
   */
  public static getStaggerDelay(
    taskName: string,
    interval: TaskInterval
  ): number {
    const windowMs = staggerWindows[interval];
    let hash = 0;
    for (let i = 0; i < taskName.length; i++) {
      hash = ((hash << 5) - hash + taskName.charCodeAt(i)) | 0;
    }
    return Math.abs(hash) % windowMs;
  }

  /**
   * UUID 分区优化方法
   * 将 UUID 空间分成 N 个相等范围，支持分布式处理
   */
  protected getPartitionWhereClause(
    idField: string,
    partitionInfo: PartitionInfo | undefined
  ): WhereAttributeHash {
    if (!partitionInfo) {
      return {};
    }

    const [startUuid, endUuid] = this.getPartitionBounds(partitionInfo);
    return {
      [idField]: {
        [Op.gte]: startUuid,
        [Op.lte]: endUuid,
      },
    };
  }
}
```

### 3.5 任务优先级和重试策略

**文件**: `server/queues/tasks/base/BaseTask.ts:11-61`

```typescript
export enum TaskPriority {
  Background = 40,  // 后台任务，最低优先级
  Low = 30,         // 低优先级
  Normal = 20,      // 正常优先级
  High = 10,        // 高优先级
}

export abstract class BaseTask<T extends object> {
  /**
   * 调度任务到队列
   */
  public schedule(props: T, options?: JobOptions): Promise<Job> {
    return taskQueue().add(
      {
        name: this.constructor.name,
        props,
      },
      { ...options, ...this.options }
    );
  }

  /**
   * 执行任务
   */
  public abstract perform(props: T): Promise<unknown>;

  /**
   * 任务失败处理（所有重试用尽后）
   */
  public onFailed(props: T): Promise<void> {
    return Promise.resolve();
  }

  /**
   * 默认任务选项
   */
  public get options(): JobOptions {
    return {
      priority: TaskPriority.Normal,
      attempts: 5,  // 最多 5 次尝试
      backoff: {
        type: "exponential",  // 指数退避
        delay: 60 * 1000,     // 首次延迟 60 秒
      },
    };
  }
}
```

---

## 4. 邮件投递链路

### 4.1 邮件模板系统

#### 4.1.1 基础邮件类

**文件**: `server/emails/templates/BaseEmail.tsx:42-354`

```typescript
export enum EmailMessageCategory {
  Authentication = "authentication",  // 认证邮件
  Invitation = "invitation",          // 邀请邮件
  Notification = "notification",      // 通知邮件
  Marketing = "marketing",            // 营销邮件
  Internal = "internal",              // 内部邮件
}

export default abstract class BaseEmail<
  T extends EmailProps,
  S extends Record<string, unknown> | void = void,
> {
  private props: T;
  private metadata?: NotificationMetadata;

  protected abstract get category(): EmailMessageCategory;

  /**
   * 调度邮件发送（异步）
   */
  public schedule(options?: Bull.JobOptions) {
    // 检查 SMTP 配置
    if (!env.SMTP_FROM_EMAIL) {
      Logger.info(
        "email",
        `Email ${this.constructor.name} not sent due to missing SMTP_FROM_EMAIL configuration`
      );
      return;
    }

    const templateName = this.constructor.name;
    Metrics.increment("email.scheduled", { templateName });

    // 将 EmailTask 加入任务队列
    return taskQueue().add(
      {
        name: "EmailTask",
        props: {
          templateName,
          ...this.metadata,
          props: this.props,
        },
      },
      {
        priority: TaskPriority.Normal,
        attempts: 5,
        backoff: {
          type: "exponential",
          delay: 60 * 1000,
        },
        ...options,
      }
    );
  }

  /**
   * 立即发送邮件
   */
  public async send() {
    const templateName = this.constructor.name;
    
    // 前置钩子
    const bsResponse = await this.beforeSend?.(this.props);
    if (bsResponse === false) {
      Logger.info(
        "email",
        `Email ${templateName} not sent due to beforeSend hook`,
        this.props
      );
      return;
    }

    // 检查收件人
    if (!this.props.to) {
      Logger.info(
        "email",
        `Email ${templateName} not sent due to missing email address`,
        this.props
      );
      return;
    }

    // 获取通知（如果有）
    const notification = this.metadata?.notificationId
      ? await Notification.scope(["withActor", "withUser"]).findByPk(
          this.metadata?.notificationId
        )
      : undefined;

    // 如果通知已被查看，则不发送邮件
    if (notification?.viewedAt) {
      Logger.info(
        "email",
        `Email ${templateName} not sent as already viewed`,
        this.props
      );
      return;
    }

    // 生成 Message-ID 用于邮件线程
    const messageId = notification
      ? Notification.emailMessageId(notification.id)
      : undefined;

    // 生成 References 用于邮件线程
    const references = notification
      ? await Notification.emailReferences(notification)
      : undefined;

    // 检查是否延迟通知（超过 30 分钟）
    let subject = this.subject(data);
    if (notification) {
      if (notification.createdAt < subMinutes(new Date(), 30)) {
        subject = `${this.t("Delayed notification")}: ${subject}`;
      }
    }

    try {
      // 调用 mailer 发送
      await mailer.sendMail({
        to: this.props.to,
        replyTo: this.replyTo?.(data),
        from: this.from(data),
        subject,
        messageId,
        references,
        previewText: this.preview(data),
        component: (
          <>
            {this.render(data)}
            {notification ? this.pixel(notification) : null}
          </>
        ),
        text: this.renderAsText(data),
        headCSS: this.headCSS?.(data),
        unsubscribeUrl: this.unsubscribeUrl?.(data),
      });
      Metrics.increment("email.sent", { templateName });
    } catch (err) {
      Metrics.increment("email.sending_failed", { templateName });
      throw err;
    }

    // 更新通知的 emailedAt 时间
    if (notification) {
      try {
        notification.emailedAt = new Date();
        await notification.save();
      } catch (err) {
        Logger.error(`Failed to update notification`, err, this.metadata);
      }
    }
  }

  // 抽象方法，子类必须实现
  protected abstract subject(props: S & T): string;
  protected abstract preview(props: S & T): string;
  protected abstract renderAsText(props: S & T): string;
  protected abstract render(props: S & T): JSX.Element;

  // 可选钩子
  protected replyTo?(props: S & T): string | undefined;
  protected unsubscribeUrl?(props: T): string;
  protected headCSS?(props: T): string | undefined;
  protected beforeSend?(props: T): Promise<S | false>;
  protected fromName?(props: T): string | undefined;
}
```

#### 4.1.2 邮件模板示例

以 `DocumentPublishedOrUpdatedEmail` 为例，展示完整的邮件模板：

**文件**: `server/emails/templates/DocumentPublishedOrUpdatedEmail.tsx:1-243`

```typescript
type InputProps = EmailProps & {
  userId: string;
  documentId: string;
  revisionId?: string;
  actorName: string;
  eventType:
    | NotificationEventType.PublishDocument
    | NotificationEventType.UpdateDocument;
  teamUrl: string;
};

type BeforeSend = {
  document: Document;
  collection: Collection | null;
  unsubscribeUrl: string;
  body: string | undefined;
};

export default class DocumentPublishedOrUpdatedEmail extends BaseEmail<
  InputProps,
  BeforeSend
> {
  protected get category() {
    return EmailMessageCategory.Notification;
  }

  /**
   * 发送前预处理：加载文档数据、生成 diff
   */
  protected async beforeSend(props: InputProps) {
    const { documentId, revisionId } = props;
    const document = await Document.unscoped().findByPk(documentId, {
      includeState: true,
    });
    if (!document) {
      return false;  // 文档不存在，取消发送
    }

    const [collection, team] = await Promise.all([
      document.$get("collection"),
      document.$get("team"),
    ]);

    let body;
    // 如果启用了邮件预览，生成 diff
    if (revisionId && team?.getPreference(TeamPreference.PreviewsInEmails)) {
      body = await CacheHelper.getDataOrSet<string>(
        `diff:${revisionId}`,
        async () => {
          const revision = await Revision.findByPk(revisionId);
          if (revision) {
            const before = await revision.before();
            const content = await DocumentHelper.toEmailDiff(before, revision, {
              includeTitle: false,
              centered: false,
              signedUrls: 4 * Day.seconds,
              baseUrl: props.teamUrl,
            });
            // CSS 内联以兼容更多邮件客户端
            return content ? await HTMLHelper.inlineCSS(content) : undefined;
          }
          return;
        },
        30,
        10000
      );
    }

    return {
      document,
      collection,
      body,
      unsubscribeUrl: this.unsubscribeUrl(props),
    };
  }

  protected unsubscribeUrl({ userId, eventType }: InputProps) {
    return NotificationSettingsHelper.unsubscribeUrl(userId, eventType);
  }

  protected subject({ document, eventType }: Props) {
    return this.t(`"{{ documentTitle }}" {{ eventName }}`, {
      documentTitle: document.titleWithDefault,
      eventName: this.eventName(eventType),
    });
  }

  protected preview({ actorName, eventType }: Props): string {
    return this.t("{{ actorName }} {{ eventName }} a document", {
      actorName,
      eventName: this.eventName(eventType),
    });
  }

  protected fromName({ actorName }: Props) {
    return actorName;  // 发件人显示为操作用户名
  }

  /**
   * 渲染 React 组件为 HTML
   */
  protected render(props: Props) {
    const {
      document,
      actorName,
      collection,
      eventType,
      teamUrl,
      unsubscribeUrl,
      body,
    } = props;
    const documentLink = `${teamUrl}${document.url}?ref=notification-email`;
    const eventName = this.eventName(eventType);

    return (
      <EmailTemplate
        previewText={this.preview(props)}
        goToAction={{ url: documentLink, name: this.t("View Document") }}
      >
        <Header />
        <Body>
          <Heading>
            {this.t(`"{{ documentTitle }}" {{ eventName }}`, {
              documentTitle: document.titleWithDefault,
              eventName,
            })}
          </Heading>
          <p>
            {this.t("{{ actorName }} {{ eventName }} the document", {
              actorName,
              eventName,
            })}{" "}
            <a href={documentLink}>{document.titleWithDefault}</a>
            {collection?.name ? (
              <>
                ,{" "}
                {this.t("in the {{ collectionName }} collection", {
                  collectionName: collection.name,
                })}
              </>
            ) : (
              ""
            )}
            .
          </p>
          {body && (
            <>
              <EmptySpace height={20} />
              <Diff>
                <div dangerouslySetInnerHTML={{ __html: body }} />
              </Diff>
              <EmptySpace height={20} />
            </>
          )}
          <p>
            <Button href={documentLink}>{this.t("Open Document")}</Button>
          </p>
        </Body>
        <Footer
          unsubscribeUrl={unsubscribeUrl}
          unsubscribeText={this.t("Unsubscribe from these emails")}
        >
          <Link
            href={SubscriptionHelper.unsubscribeUrl(
              props.userId,
              props.documentId
            )}
          >
            {this.t("Unsubscribe from this doc")}
          </Link>
        </Footer>
      </EmailTemplate>
    );
  }
}
```

### 4.2 EmailTask 执行

**文件**: `server/queues/tasks/EmailTask.ts:1-22`

```typescript
export default class EmailTask extends BaseTask<Props> {
  public async perform({ templateName, props, ...metadata }: Props) {
    const EmailClass = emails[templateName];
    if (!EmailClass) {
      throw new Error(
        `Email task "${templateName}" template does not exist. Check the file name matches the class name.`
      );
    }

    // @ts-expect-error We won't instantiate an abstract class
    const email = new EmailClass(props, metadata);
    return email.send();
  }
}
```

### 4.3 SMTP 邮件投递

#### 4.3.1 SMTP 配置项

**文件**: `server/env.ts:386-466`

```typescript
// SMTP 主机（二选一：SMTP_HOST 或 SMTP_SERVICE）
public SMTP_HOST = this.toOptionalString(environment.SMTP_HOST);

// 知名邮件服务名称（如 Gmail, Outlook 等）
@CannotUseWith("SMTP_HOST")
@IsInCaseInsensitive(Object.keys(wellKnownServices))
public SMTP_SERVICE = this.toOptionalString(environment.SMTP_SERVICE);

// 邮件是否启用
@Public
public EMAIL_ENABLED =
  !!(this.SMTP_HOST || this.SMTP_SERVICE) || this.isDevelopment;

// SMTP 端口
@IsNumber()
@IsOptional()
@CannotUseWith("SMTP_SERVICE")
public SMTP_PORT = this.toOptionalNumber(environment.SMTP_PORT);

// SMTP 用户名
public SMTP_USERNAME = environment.SMTP_USERNAME;

// SMTP 密码
public SMTP_PASSWORD = environment.SMTP_PASSWORD;

// 发件人邮箱
@IsMailboxAddress()
@IsOptional()
public SMTP_FROM_EMAIL = this.toOptionalString(environment.SMTP_FROM_EMAIL);

// 回复邮箱
@IsMailboxAddress()
@IsOptional()
public SMTP_REPLY_EMAIL = this.toOptionalString(environment.SMTP_REPLY_EMAIL);

// TLS 密码套件
public SMTP_TLS_CIPHERS = this.toOptionalString(environment.SMTP_TLS_CIPHERS);

// 是否使用 TLS
public SMTP_SECURE = this.toBoolean(environment.SMTP_SECURE ?? "true");

// 是否禁用 STARTTLS
public SMTP_DISABLE_STARTTLS = this.toBoolean(
  environment.SMTP_DISABLE_STARTTLS ?? "false"
);
```

#### 4.3.2 Mailer 核心实现

**文件**: `server/emails/mailer.tsx:31-262`

```typescript
@trace({
  serviceName: "mailer",
})
export class Mailer {
  transporter: Transporter | undefined;

  constructor() {
    // 生产环境：使用配置的 SMTP
    if (env.SMTP_HOST || env.SMTP_SERVICE) {
      this.transporter = nodemailer.createTransport(this.getOptions());
    }
    
    // 开发环境：如果没有配置 SMTP，尝试创建测试账户
    if (useTestEmailService) {
      Logger.info(
        "email",
        "SMTP_USERNAME not provided, generating test account…"
      );
      void this.getTestTransportOptions().then((options) => {
        if (!options) {
          Logger.info(
            "email",
            "Couldn't generate a test account with ethereal.email at this time – emails will not be sent."
          );
          return;
        }
        this.transporter = nodemailer.createTransport(options);
      });
    }
  }

  /**
   * 发送邮件
   */
  sendMail = async (data: SendMailOptions): Promise<void> => {
    const transporter = this.transporter;

    // 开发环境日志
    if (env.isDevelopment) {
      Logger.debug(
        "email",
        [
          `Sending email:`,
          ``,
          `--------------`,
          `From:      ${data.from.address}`,
          `To:        ${data.to}`,
          `Subject:   ${data.subject}`,
          `Preview:   ${data.previewText}`,
          `--------------`,
          ``,
          data.text,
        ].join("\n")
      );
    }

    if (!transporter) {
      Logger.warn("No mail transport available");
      return;
    }

    // 使用 Oy 渲染 React 组件为 HTML
    const html = Oy.renderTemplate(
      data.component,
      {
        title: data.subject,
        headCSS: [baseStyles, data.headCSS].join(" "),
      } as Oy.RenderOptions,
      this.template
    );

    try {
      Logger.info("email", `Sending email "${data.subject}" to ${data.to}`);

      const info = await transporter.sendMail({
        from: data.from,
        replyTo: data.replyTo ?? env.SMTP_REPLY_EMAIL ?? env.SMTP_FROM_EMAIL,
        to: data.to,
        messageId: data.messageId,
        references: data.references,
        inReplyTo: data.references?.at(-1),
        subject: data.subject,
        html,
        text: data.text,
        // 列表头（用于一键退订）
        list: data.unsubscribeUrl
          ? {
              unsubscribe: {
                url: data.unsubscribeUrl,
                comment: "Unsubscribe from these emails",
              },
            }
          : undefined,
        // 附件（非云托管版本附加 logo）
        attachments: env.isCloudHosted
          ? undefined
          : [
              {
                filename: "header-logo.png",
                path: process.cwd() + "/public/email/header-logo.png",
                cid: "header-image",
              },
            ],
      });

      // 测试环境显示预览链接
      if (useTestEmailService) {
        Logger.info(
          "email",
          `Preview Url: ${nodemailer.getTestMessageUrl(info)}`
        );
      }
    } catch (err) {
      Logger.error(`Error sending email to ${data.to}`, err);
      throw err;  // 重新抛出以便队列重试
    }
  };

  /**
   * 获取 SMTP 配置选项
   */
  private getOptions(): SMTPTransport.Options {
    // 使用知名邮件服务（如 Gmail）
    if (env.SMTP_SERVICE) {
      return {
        service: env.SMTP_SERVICE,
        auth: {
          user: env.SMTP_USERNAME,
          pass: env.SMTP_PASSWORD,
        },
      };
    }

    // 使用自定义 SMTP 服务器
    return {
      name: env.SMTP_NAME,
      host: env.SMTP_HOST,
      port: env.SMTP_PORT,
      // 生产环境默认使用 TLS
      secure: env.SMTP_SECURE ?? env.isProduction,
      // 认证
      auth: env.SMTP_USERNAME
        ? {
            user: env.SMTP_USERNAME,
            pass: env.SMTP_PASSWORD,
          }
        : undefined,
      // TLS 配置
      ignoreTLS: env.SMTP_DISABLE_STARTTLS,
      tls: env.SMTP_SECURE
        ? env.SMTP_TLS_CIPHERS
          ? {
              ciphers: env.SMTP_TLS_CIPHERS,
            }
          : undefined
        : {
            rejectUnauthorized: false,  // 自签名证书兼容
          },
    };
  }

  /**
   * 创建测试邮箱账户（ethereal.email）
   */
  private async getTestTransportOptions(): Promise<
    SMTPTransport.Options | undefined
  > {
    try {
      const testAccount = await nodemailer.createTestAccount();
      return {
        host: "smtp.ethereal.email",
        port: 587,
        secure: false,
        auth: {
          user: testAccount.user,
          pass: testAccount.pass,
        },
      };
    } catch (_err) {
      return undefined;
    }
  }
}
```

---

## 5. 分享订阅通知（类摘要功能）

Outline 支持公开分享文档的订阅功能，外部用户可以订阅文档更新通知。

### 5.1 分享订阅数据模型

**文件**: `server/models/ShareSubscription.ts:24-187`

```typescript
@Scopes(() => ({
  active: {
    where: {
      confirmedAt: { [Op.not]: null },  // 已确认
      unsubscribedAt: null,              // 未退订
    },
  },
}))
@Table({ tableName: "share_subscriptions", modelName: "share_subscription" })
class ShareSubscription extends IdModel {
  @ForeignKey(() => Share)
  @Column(DataType.UUID)
  shareId: string;

  @ForeignKey(() => Document)
  @Column(DataType.UUID)
  documentId: string;  // 订阅范围文档（包含子文档）

  @Column(DataType.STRING)
  email: string;  // 订阅者邮箱

  @Column(DataType.STRING)
  emailFingerprint: string;  // 标准化邮箱指纹（用于防垃圾邮件）

  @Column(DataType.STRING)
  secret: string;  // 退订/确认签名密钥

  @Column(DataType.STRING(45))
  ipAddress: string | null;

  @Column(DataType.DATE)
  confirmedAt: Date | null;  // 确认时间

  @Column(DataType.DATE)
  unsubscribedAt: Date | null;  // 退订时间

  @Column(DataType.DATE)
  lastNotifiedAt: Date | null;  // 最后通知时间

  /**
   * 每个 IP 最多 3 个订阅（防垃圾邮件）
   */
  static maxSubscriptionsPerIP = 3;

  get isConfirmed(): boolean {
    return !!this.confirmedAt;
  }

  get isUnsubscribed(): boolean {
    return !!this.unsubscribedAt;
  }
}
```

### 5.2 分享订阅通知任务

**文件**: `server/queues/tasks/ShareSubscriptionNotificationsTask.ts:8-94`

```typescript
export default class ShareSubscriptionNotificationsTask extends BaseTask<RevisionEvent> {
  public async perform(event: RevisionEvent) {
    const document = await Document.findByPk(event.documentId);
    if (!document) {
      return;
    }

    // 收集文档及其所有祖先文档 ID
    const scopeIds: string[] = [document.id];
    let parentId = document.parentDocumentId;
    while (parentId) {
      scopeIds.push(parentId);
      const parent = await Document.findByPk(parentId, {
        attributes: ["id", "parentDocumentId"],
      });
      if (!parent) {
        break;
      }
      parentId = parent.parentDocumentId;
    }

    // 查找所有活跃的订阅
    const subscriptions = await ShareSubscription.scope("active").findAll({
      where: { documentId: scopeIds },
      include: [
        {
          model: Share.unscoped(),
          required: true,
          where: {
            published: true,
            revokedAt: null,
            allowSubscriptions: true,  // 分享链接允许订阅
          },
          include: [{ association: "team", required: true }],
        },
      ],
    });

    for (const subscription of subscriptions) {
      // 检查子文档访问权限
      if (
        subscription.documentId !== document.id &&
        !subscription.share.includeChildDocuments
      ) {
        continue;
      }

      // 节流：每 6 小时最多一次通知
      if (
        subscription.lastNotifiedAt &&
        subscription.lastNotifiedAt > subHours(new Date(), 6)
      ) {
        Logger.info(
          "processor",
          `suppressing share subscription notification to ${subscription.id} as recently notified`
        );
        continue;
      }

      // 构建分享链接
      const baseShareUrl = subscription.share.canonicalUrl;
      const shareUrl =
        document.id !== subscription.share.documentId && document.path
          ? `${baseShareUrl.replace(/\/$/, "")}${document.path}`
          : baseShareUrl;

      // 发送更新邮件
      await new ShareDocumentUpdatedEmail({
        to: subscription.email,
        shareSubscriptionId: subscription.id,
        documentTitle: document.titleWithDefault,
        shareUrl,
        revisionId: event.modelId,
      }).schedule();

      // 更新最后通知时间
      subscription.lastNotifiedAt = new Date();
      await subscription.save();
    }
  }

  public get options() {
    return {
      priority: TaskPriority.Background,  // 后台任务
    };
  }
}
```

---

## 6. 完整流程图

### 6.1 实时通知流程

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              用户操作 (如发布文档)                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        Event.createFromContext()                                   │
│  - 从请求上下文提取 actorId, teamId, ip                                             │
│  - 创建 Event 记录并保存到 PostgreSQL                                               │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        Event.enqueue() (@AfterSave 钩子)                          │
│  - 事务提交后将事件加入 globalEventQueue                                            │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    Worker 处理 globalEventQueue                                    │
│  - 遍历所有 processors，检查 applicableEvents                                      │
│  - 匹配的处理器加入 processorEventQueue                                             │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    NotificationsProcessor.perform()                                │
│  - 根据事件类型执行相应逻辑                                                          │
│  - 解析提及、查找订阅者                                                              │
│  - 创建 Notification 记录                                                           │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    Notification.createEvent() (@AfterCreate 钩子)                 │
│  - 触发 notifications.create 事件                                                    │
│  - 加入 globalEventQueue                                                             │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    EmailsProcessor.perform()                                        │
│  - 根据 Notification.event 类型选择邮件模板                                         │
│  - 调用 EmailClass.schedule() 加入 taskQueue                                       │
│  - 部分邮件有延迟（如 1 分钟）避免频繁通知                                           │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    EmailTask.perform()                                              │
│  - 实例化邮件模板类                                                                   │
│  - 调用 email.send()                                                                 │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    BaseEmail.send()                                                 │
│  - 执行 beforeSend 钩子（加载数据、生成 diff）                                       │
│  - 检查通知是否已被查看（viewedAt），如果已查看则取消发送                            │
│  - 调用 mailer.sendMail()                                                           │
│  - 更新 Notification.emailedAt                                                       │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    Mailer.sendMail()                                                │
│  - 使用 Oy.renderTemplate() 将 React 组件渲染为 HTML                                │
│  - 调用 nodemailer transporter.sendMail()                                           │
│  - 通过 SMTP 投递到邮件服务器                                                         │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 定时任务流程

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    内部定时器 (setInterval) 或 HTTP API                            │
│                    GET /api/cron.daily?token=UTILS_SECRET                          │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    CronTask.schedule()                                              │
│  - 根据任务名称计算 stagger delay（错开执行时间）                                     │
│  - 支持分区处理（分布式部署时）                                                       │
│  - 加入 taskQueue                                                                    │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    定时任务示例                                                       │
│  - CleanupOldNotificationsTask: 清理旧通知                                          │
│  - CleanupOldEventsTask: 清理旧事件                                                  │
│  - UpdateDocumentsPopularityScoreTask: 更新文档流行度                                │
│  - InviteReminderTask: 邀请提醒（如果是 CronTask）                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 关键设计特点

### 7.1 可靠性设计

| 特性 | 实现方式 |
|------|----------|
| **事务一致性** | 事件入队使用 `transaction.afterCommit()` 确保数据和队列一致 |
| **自动重试** | Bull 队列配置 `attempts: 5` + `exponential` 退避策略 |
| **失败处理** | 任务最后一次失败时调用 `onFailed()` 钩子 |
| **幂等性** | 通过 `Notification.emailedAt` 标记避免重复发送 |

### 7.2 用户体验优化

| 特性 | 实现方式 |
|------|----------|
| **免打扰** | 通知已查看（`viewedAt`）则不发送邮件 |
| **延迟通知** | 部分邮件延迟 1 分钟发送，给用户撤销操作的机会 |
| **节流控制** | 分享订阅每 6 小时最多一次通知 |
| **智能退订** | 邮件包含 `List-Unsubscribe` 头，支持一键退订 |
| **阅读追踪** | 邮件包含 1x1 像素图片，追踪邮件是否被打开 |

### 7.3 可观测性

| 指标 | 采集位置 |
|------|----------|
| `email.scheduled` | BaseEmail.schedule() |
| `email.sent` | BaseEmail.send() |
| `email.sending_failed` | BaseEmail.send() catch 块 |
| `queue.*.jobs.*` | createQueue() 事件监听 |

### 7.4 安全考虑

| 安全特性 | 实现方式 |
|----------|----------|
| **退订令牌** | 使用 `SHA256(userId + SECRET_KEY + eventType)` 签名 |
| **像素令牌** | 使用 `SHA256(notificationId + SECRET_KEY)` 签名 |
| **IP 限制** | 分享订阅每 IP 最多 3 个订阅 |
| **邮箱指纹** | Gmail 等邮箱标准化处理，防止 `user+tag@gmail.com` 绕过限制 |
| **Cron API 保护** | 需要 `UTILS_SECRET` 验证 |

---

## 8. 相关文件索引

| 功能模块 | 文件路径 |
|----------|----------|
| 邮件发送核心 | `server/emails/mailer.tsx` |
| 邮件模板基类 | `server/emails/templates/BaseEmail.tsx` |
| 邮件任务 | `server/queues/tasks/EmailTask.ts` |
| 通知模型 | `server/models/Notification.ts` |
| 通知处理器 | `server/queues/processors/NotificationsProcessor.ts` |
| 邮件处理器 | `server/queues/processors/EmailsProcessor.ts` |
| 工作进程 | `server/services/worker.ts` |
| 定时任务服务 | `server/services/cron.ts` |
| 定时任务 API | `server/routes/api/cron/cron.ts` |
| CronTask 基类 | `server/queues/tasks/base/CronTask.ts` |
| 队列配置 | `server/queues/index.ts` |
| 队列实现 | `server/queues/queue.ts` |
| 环境变量 | `server/env.ts` |
| 用户模型 | `server/models/User.ts` |
| 分享订阅 | `server/models/ShareSubscription.ts` |
| 分享订阅通知任务 | `server/queues/tasks/ShareSubscriptionNotificationsTask.ts` |
