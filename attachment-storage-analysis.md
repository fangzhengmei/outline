# Outline 附件存储机制分析报告

## 1. 概述

Outline 是一个支持实时协作的知识库应用，其附件存储系统设计灵活，支持两种存储后端：
- **S3 对象存储**（默认，适用于生产环境）
- **本地文件系统存储**（适用于开发和小型部署）

本报告详细分析 Outline 的附件存储机制、签名 URL 生成流程、权限控制策略，以及**上传入口与下载入口的职责分离**和**权限边界分层**。

---

## 2. 存储后端架构

### 2.1 存储后端选择机制

存储后端的选择由环境变量 `FILE_STORAGE` 决定：

```typescript
// server/storage/files/index.ts
const storage =
  env.FILE_STORAGE === "local" ? new LocalStorage() : new S3Storage();

export default storage;
```

**配置说明**：
- 当 `FILE_STORAGE=local` 时，使用本地文件系统存储
- 其他情况（包括未设置）使用 S3 存储

### 2.2 存储接口设计

所有存储后端都继承自抽象基类 `BaseStorage`，定义了统一的接口：

| 方法 | 功能描述 |
|------|----------|
| `getPresignedPost()` | 生成上传预签名信息 |
| `store()` | 存储文件到后端 |
| `deleteFile()` | 删除文件 |
| `getSignedUrl()` | 生成下载签名 URL |
| `getFileStream()` | 获取文件流 |
| `getUrlForKey()` | 获取文件的直接 URL |
| `getUploadUrl()` | 获取上传端点 URL |

### 2.3 S3 存储实现

**核心文件**：`server/storage/files/S3Storage.ts`

#### 2.3.1 初始化配置

S3 客户端使用 AWS SDK v3 构建，支持多种配置选项：

```typescript
this.client = new S3Client({
  bucketEndpoint: env.AWS_S3_ACCELERATE_URL ? true : false,
  forcePathStyle: env.AWS_S3_FORCE_PATH_STYLE,
  region: env.AWS_REGION,
  endpoint: this.getEndpoint(),
});
```

**环境变量配置**：
- `AWS_REGION` - S3 区域
- `AWS_S3_UPLOAD_BUCKET_NAME` - 存储桶名称
- `AWS_S3_UPLOAD_BUCKET_URL` - 存储桶 URL
- `AWS_S3_ACCELERATE_URL` - S3 Transfer Acceleration 加速 URL（可选）
- `AWS_S3_FORCE_PATH_STYLE` - 是否使用路径风格的访问
- `AWS_S3_ACL` - 默认 ACL 配置

#### 2.3.2 文件存储路径（Bucket 分层）

文件存储路径（key）采用** bucket 前缀分层**设计，格式如下：

```
{bucket}/{userId}/{attachmentId}/{fileName}
```

**Bucket 类型定义**：

```typescript
// server/models/helpers/AttachmentHelper.ts
export enum Buckets {
  public = "public",    // 公开桶 - 特殊公开附件
  uploads = "uploads",  // 私有桶 - 普通附件
  avatars = "avatars",  // 头像桶 - 用户/团队头像
}
```

**路径生成逻辑**：

```typescript
// server/models/helpers/AttachmentHelper.ts
static getKey({ id, name, userId }: { id: string; name: string; userId: string }) {
  const keyPrefix = `${Buckets.uploads}/${userId}/${id}`;
  return ValidateKey.sanitize(
    `${keyPrefix}/${name.slice(0, this.maximumFileNameLength)}`
  );
}
```

**注意**：当前 `getKey()` 方法只生成 `uploads/` 前缀的路径。`avatars/` 和 `public/` 前缀主要用于**历史数据兼容**和**特殊场景**（如测试、导入等）。

#### 2.3.3 文件上传流程

S3 存储使用 AWS SDK 的 `Upload` 类进行分块上传，支持大文件：

```typescript
public store = async ({ body, contentType, key }: StoreParams) => {
  const upload = new Upload({
    client: this.client,
    params: {
      ...(env.AWS_S3_ACL && { ACL: env.AWS_S3_ACL as ObjectCannedACL }),
      Bucket: this.getBucket(),
      Key: key,
      ContentType: contentType,
      ContentDisposition: this.getContentDisposition(contentType),
      Body: body,
    },
  });
  await upload.done();
  
  const endpoint = this.getPublicEndpoint(true);
  return `${endpoint}/${key}`;
};
```

**注意**：代码中注释掉了 `ContentLength` 参数，因为 AWS SDK v3 在处理大文件时存在已知的挂起问题。

### 2.4 本地存储实现

**核心文件**：`server/storage/files/LocalStorage.ts`

#### 2.4.1 初始化配置

本地存储使用 `FILE_STORAGE_LOCAL_ROOT_DIR` 环境变量指定根目录：

```typescript
private getFilePath(key: string) {
  invariant(
    env.FILE_STORAGE_LOCAL_ROOT_DIR,
    "FILE_STORAGE_LOCAL_ROOT_DIR is required"
  );
  
  return safeResolvePath(env.FILE_STORAGE_LOCAL_ROOT_DIR, key);
}
```

使用 `safeResolvePath` 防止路径遍历攻击。

#### 2.4.2 文件存储路径

本地存储的路径结构与 S3 相同，但存储在本地文件系统中：

```
{FILE_STORAGE_LOCAL_ROOT_DIR}/{bucket}/{userId}/{attachmentId}/{fileName}
```

#### 2.4.3 文件上传流程

本地存储直接使用 Node.js 文件系统 API：

```typescript
public store = async ({ body, key }: StoreParams) => {
  const exists = await fs.pathExists(this.getFilePath(key));
  if (exists) {
    throw ValidationError(`File already exists at ${key}`);
  }

  await mkdir(this.getFilePath(path.dirname(key)), { recursive: true });

  // 根据 body 类型创建可读流
  let src: NodeJS.ReadableStream;
  if (body instanceof fs.ReadStream) {
    src = body;
  } else if (body instanceof Blob) {
    src = Readable.from(Buffer.from(await body.arrayBuffer()));
  } else {
    src = Readable.from(body);
  }

  const filePath = this.getFilePath(key);
  await fs.createFile(filePath);

  return new Promise<string>((resolve, reject) => {
    const dest = fs
      .createWriteStream(filePath)
      .on("error", reject)
      .on("finish", () => resolve(this.getUrlForKey(key)));

    src.on("error", (err) => { dest.end(); reject(err); })
       .pipe(dest);
  });
};
```

---

## 3. 上传入口职责分析

### 3.1 上传入口概述

Outline 提供三个主要的附件上传入口：

| 入口路由 | 功能描述 | 认证要求 |
|----------|----------|----------|
| `attachments.create` | 标准上传入口，返回预签名 URL | 必需 |
| `attachments.createFromUrl` | 从远程 URL 创建附件 | 必需 |
| `attachmentCreator` | 程序化创建（命令模式） | 内部调用 |

### 3.2 标准上传入口：attachments.create

**核心文件**：`server/routes/api/attachments/attachments.ts`

#### 3.2.1 中间件配置

```typescript
router.post(
  "attachments.create",
  rateLimiter(RateLimiterStrategy.TwentyFivePerMinute),  // 速率限制
  auth(),                                                  // 必需认证
  validate(T.AttachmentsCreateSchema),                    // 参数验证
  transaction(),                                           // 事务支持
  async (ctx: APIContext<T.AttachmentCreateReq>) => {
    // ... 业务逻辑
  }
);
```

#### 3.2.2 权限检查策略（严格）

上传入口采用**严格的权限检查策略**，根据 `preset` 类型进行不同的验证：

```typescript
const { id, name, documentId, contentType, size, preset } = ctx.input.body;
const { user } = ctx.state.auth;

// 权限检查逻辑
if (preset === AttachmentPreset.Avatar) {
  // 1. 头像预设：只需验证内容类型
  // 所有用户类型都可以上传头像，无需额外授权
  assertIn(contentType, AttachmentValidation.avatarContentTypes);
} else {
  // 2. 文档附件预设：检查文档更新权限
  if (preset === AttachmentPreset.DocumentAttachment && documentId) {
    const document = await Document.findByPk(documentId, {
      userId: user.id,
      transaction,
    });
    authorize(user, "update", document);  // 必须有文档更新权限
  }
  
  // 3. 表情符号预设：验证内容类型
  if (preset === AttachmentPreset.Emoji) {
    assertIn(contentType, AttachmentValidation.emojiContentTypes);
  }

  // 4. 通用检查：必须有团队附件创建权限
  authorize(user, "createAttachment", user.team);
}
```

#### 3.2.3 权限检查矩阵

| Preset 类型 | documentId | 权限检查 | 说明 |
|-------------|------------|----------|------|
| `Avatar` | 忽略 | 仅验证 content-type | 任何人可上传头像 |
| `DocumentAttachment` | 有值 | `update` 文档权限 + `createAttachment` 团队权限 | 必须能编辑文档 |
| `DocumentAttachment` | 无值 | `createAttachment` 团队权限 | 未关联文档的附件 |
| `Emoji` | 忽略 | 验证 content-type + `createAttachment` | 团队成员可上传表情 |
| `Import`/其他 | 忽略 | `createAttachment` 团队权限 | 团队成员可上传 |

#### 3.2.4 上传流程

```
1. 客户端请求 POST /api/attachments.create
   ├── preset: AttachmentPreset
   ├── documentId?: string
   ├── name: string
   ├── contentType: string
   └── size: number
   ↓
2. 服务端权限验证
   ├── 验证用户身份（auth()）
   ├── 根据 preset 检查权限
   │   ├── Avatar: 仅验证 content-type
   │   ├── DocumentAttachment + documentId: 检查文档 update 权限
   │   └── 其他: 检查 createAttachment 团队权限
   └── 验证文件大小
   ↓
3. 创建 Attachment 数据库记录
   ├── id: UUID
   ├── key: uploads/{userId}/{id}/{name}
   ├── acl: 根据 preset 确定
   ├── documentId: 可选关联
   ├── teamId: 用户所属团队
   └── userId: 上传者 ID
   ↓
4. 生成预签名 POST 信息
   └── FileStorage.getPresignedPost(ctx, key, acl, maxUploadSize, contentType)
   ↓
5. 返回上传信息
   ├── uploadUrl: 存储后端上传 URL
   ├── form: 表单字段（包含签名）
   └── attachment: 附件信息（含 redirectUrl）
   ↓
6. 客户端直接向存储后端上传文件
   └── 使用 uploadUrl 和 form 字段
```

#### 3.2.5 ACL 设置

```typescript
// server/models/helpers/AttachmentHelper.ts
static presetToAcl(preset: AttachmentPreset) {
  switch (preset) {
    case AttachmentPreset.Avatar:
      return "public-read";  // 头像始终公开
    default:
      return env.AWS_S3_ACL;  // 其他使用环境变量配置
  }
}
```

**关键点**：
- 头像预设强制使用 `public-read` ACL
- 其他预设使用 `AWS_S3_ACL` 环境变量（通常是 `private`）

### 3.3 从 URL 创建入口：attachments.createFromUrl

```typescript
router.post(
  "attachments.createFromUrl",
  rateLimiter(RateLimiterStrategy.TwentyFivePerMinute),
  auth(),
  validate(T.AttachmentsCreateFromUrlSchema),
  async (ctx: APIContext<T.AttachmentCreateFromUrlReq>) => {
    const { id, url, documentId, preset } = ctx.input.body;

    // 仅支持文档附件类型且必须有 documentId
    if (preset !== AttachmentPreset.DocumentAttachment || !documentId) {
      throw ValidationError("Only document attachments can be created from a URL");
    }

    // 检查文档更新权限
    const document = await Document.findByPk(documentId, { userId: user.id });
    authorize(user, "update", document);

    // 创建附件记录
    const attachment = await sequelize.transaction(async (transaction) =>
      Attachment.createWithCtx(ctx, { /* ... */ })
    );

    // 调度后台任务下载文件
    const job = await new UploadAttachmentFromUrlTask().schedule({
      attachmentId: attachment.id,
      url,
    });

    // 等待任务完成
    const response = await job.finished();
    // ...
  }
);
```

**特点**：
- 必须关联文档
- 必须有文档更新权限
- 使用后台任务异步下载
- 同步等待任务完成后返回

### 3.4 程序化创建：attachmentCreator

**核心文件**：`server/commands/attachmentCreator.ts`

这是一个内部命令，用于程序化创建附件（如导入、头像同步等场景）：

```typescript
export default async function attachmentCreator({
  id, name, user, preset, ctx, url, buffer, type, fetchOptions
}: Props): Promise<Attachment | undefined> {
  
  const acl = AttachmentHelper.presetToAcl(preset);
  const key = AttachmentHelper.getKey({ id: randomUUID(), name, userId: user.id });

  if ("url" in rest) {
    // 从 URL 下载并存储
    const res = await FileStorage.storeFromUrl(url, key, acl, fetchOptions);
    if (!res) return;
    
    return Attachment.createWithCtx(ctx, {
      id, key, acl, size: res.contentLength, 
      contentType: res.contentType, teamId: user.teamId, userId: user.id
    });
  } else {
    // 从 Buffer 存储
    await FileStorage.store({ body: buffer, contentType: type, key, acl });
    
    return Attachment.createWithCtx(ctx, {
      id, key, acl, size: buffer.length, 
      contentType: type, teamId: user.teamId, userId: user.id
    });
  }
}
```

**使用场景**：
- `UploadUserAvatarTask` - 从 SSO 同步用户头像
- `UploadTeamAvatarTask` - 从 SSO 同步团队头像
- 导入任务 - 从导出文件恢复附件

---

## 4. 下载入口职责分析

### 4.1 下载入口概述

**唯一的下载入口**：`attachments.redirect`

| 入口路由 | 功能描述 | 认证要求 |
|----------|----------|----------|
| `attachments.redirect` (GET/POST) | 附件重定向入口，验证权限后重定向到实际 URL | **可选** |

### 4.2 重定向入口：attachments.redirect

**核心文件**：`server/routes/api/attachments/attachments.ts`

#### 4.2.1 中间件配置（关键：认证可选）

```typescript
// GET 方法
router.get(
  "attachments.redirect",
  auth({ optional: true }),  // ⚠️ 认证是可选的！
  validate(T.AttachmentsRedirectSchema),
  handleAttachmentsRedirect
);

// POST 方法
router.post(
  "attachments.redirect",
  auth({ optional: true }),  // ⚠️ 认证是可选的！
  validate(T.AttachmentsRedirectSchema),
  handleAttachmentsRedirect
);
```

**关键点**：`auth({ optional: true })` 意味着**未登录用户也可以访问此端点**！

#### 4.2.2 权限检查策略（宽松）

下载入口采用**宽松的权限检查策略**，与上传入口形成鲜明对比：

```typescript
const handleAttachmentsRedirect = async (
  ctx: APIContext<T.AttachmentsRedirectReq>
) => {
  const id = (ctx.input.body.id ?? ctx.input.query.id) as string;

  const user = ctx.state.auth?.user;  // 可能为 undefined！
  const attachment = await Attachment.findByPk(id, {
    rejectOnEmpty: true,
  });

  // ⚠️ 核心权限检查逻辑
  // 只有私有附件才需要检查权限
  if (attachment.isPrivate && attachment.teamId !== user?.teamId) {
    throw AuthorizationError();
  }

  // 更新最后访问时间
  await attachment.update(
    { lastAccessedAt: new Date() },
    { silent: true }
  );

  // 根据存储位置决定重定向方式
  if (attachment.isStoredInPublicBucket) {
    ctx.set("Cache-Control", `max-age=604800, immutable`);  // 7 天
    ctx.redirect(attachment.canonicalUrl);
  } else {
    ctx.set(
      "Cache-Control",
      `max-age=${BaseStorage.defaultSignedUrlExpires}, immutable`  // 5 分钟
    );
    ctx.redirect(await attachment.signedUrl);
  }
};
```

### 4.3 权限检查逻辑详解

让我们拆解权限检查的条件：

```typescript
if (attachment.isPrivate && attachment.teamId !== user?.teamId) {
  throw AuthorizationError();
}
```

这是一个 **AND** 条件，意味着：

| 条件 | 结果 |
|------|------|
| `attachment.isPrivate = false` | **不检查权限，直接允许访问** |
| `attachment.isPrivate = true` AND `user?.teamId === attachment.teamId` | 允许访问 |
| `attachment.isPrivate = true` AND `user?.teamId !== attachment.teamId` | 拒绝访问（403） |
| `attachment.isPrivate = true` AND `user = undefined`（未登录） | `undefined !== teamId` 为 true，**拒绝访问** |

### 4.4 下载流程图

```
1. 客户端请求 GET/POST /api/attachments.redirect?id={attachmentId}
   ↓
2. 服务端验证（宽松）
   ├── 参数验证（id 必需）
   ├── 查找 Attachment 记录
   └── 权限检查（仅 isPrivate 附件）
       ├── 如果 isPrivate = false：直接通过
       ├── 如果 isPrivate = true：检查 teamId 是否匹配
       │   ├── 未登录用户：拒绝（403）
       │   ├── 不同团队用户：拒绝（403）
       │   └── 同团队用户：通过
       └── 通过后更新 lastAccessedAt
   ↓
3. 重定向决策
   ├── isStoredInPublicBucket = true
   │   ├── Cache-Control: max-age=604800 (7天)
   │   └── 重定向到 canonicalUrl（直接 URL）
   └── isStoredInPublicBucket = false
       ├── Cache-Control: max-age=300 (5分钟)
       └── 重定向到 signedUrl（签名 URL）
   ↓
4. 客户端访问实际文件 URL
   ├── canonicalUrl: 直接访问存储后端（公开资源）
   └── signedUrl: 带签名的临时 URL（有时效性）
```

---

## 5. 权限边界分层详解

### 5.1 两个独立的权限维度

Outline 的附件权限系统有**两个独立的维度**，它们控制不同的行为：

| 维度 | 决定因素 | 控制范围 |
|------|----------|----------|
| **ACL 维度 (isPrivate)** | `acl` 字段值 | **逻辑访问权限**（是否需要检查团队归属） |
| **存储位置维度 (isStoredInPublicBucket)** | `key` 的前缀 | **缓存策略和 URL 类型** |

### 5.2 ACL 维度详解

#### 5.2.1 ACL 类型

```typescript
// server/models/Attachment.ts
@Default("public-read")
@IsIn([["private", "public-read"]])
@Column
acl: string;
```

#### 5.2.2 isPrivate 判断

```typescript
// server/models/Attachment.ts
get isPrivate() {
  return this.acl === "private";
}
```

#### 5.2.3 ACL 权限边界

| ACL 值 | isPrivate | 访问权限要求 | 未登录用户 |
|--------|-----------|--------------|------------|
| `private` | `true` | 必须是同团队成员 | ❌ 拒绝（403） |
| `public-read` | `false` | **无任何权限要求** | ✅ 允许访问 |

**关键发现**：`public-read` 附件可以被**任何人**访问，包括**未登录用户**！

#### 5.2.4 测试用例验证

根据测试文件 `attachments.test.ts`：

```typescript
// 测试用例 1: 公开桶 + public-read ACL → 无需认证
it("should return a redirect for an attachment in a public bucket without authentication", async () => {
  const attachment = await buildAttachment({
    key: `public/${randomUUID()}/test.png`,  // 公开桶前缀
    acl: "public-read",
  });
  const res = await server.post("/api/attachments.redirect", {
    body: { id: attachment.id },
    // 注意：没有 token！
    redirect: "manual",
  });
  expect(res.status).toEqual(302);  // ✅ 成功重定向
});

// 测试用例 2: 私有桶 + public-read ACL → 无需认证
it("should return a redirect for a public-read attachment without authentication (not in public bucket)", async () => {
  const attachment = await buildAttachment({
    acl: "public-read",  // 公开 ACL
    // key 无 public/ 前缀，默认是 uploads/
  });
  const res = await server.post("/api/attachments.redirect", {
    body: { id: attachment.id },
    // 注意：没有 token！
    redirect: "manual",
  });
  expect(res.status).toEqual(302);  // ✅ 成功重定向
});

// 测试用例 3: private ACL + 不同团队 → 403
it("should not return a redirect for a private attachment belonging to a document user does not have access to", async () => {
  const user = await buildUser();  // 用户 A
  const attachment = await buildAttachment({
    teamId: document.teamId,  // 团队 B
    acl: "private",
  });
  const res = await server.post("/api/attachments.redirect", {
    body: {
      token: user.getJwtToken(),  // 用户 A 的 token
      id: attachment.id,
    },
  });
  expect(res.status).toEqual(403);  // ❌ 拒绝访问
});
```

### 5.3 存储位置维度详解

#### 5.3.1 Bucket 类型

```typescript
// server/models/helpers/AttachmentHelper.ts
export enum Buckets {
  public = "public",    // 公开桶
  uploads = "uploads",  // 私有桶（默认）
  avatars = "avatars",  // 头像桶
}
```

#### 5.3.2 isStoredInPublicBucket 判断

```typescript
// server/models/Attachment.ts
get isStoredInPublicBucket() {
  const bucket = this.key.split("/")[0];
  return [Buckets.avatars, Buckets.public].includes(bucket as Buckets);
}
```

#### 5.3.3 存储位置的影响

| 存储位置 | key 前缀 | isStoredInPublicBucket | 重定向 URL 类型 | Cache-Control |
|----------|-----------|------------------------|------------------|---------------|
| 公开桶 | `public/` 或 `avatars/` | `true` | `canonicalUrl`（直接 URL） | 7 天 |
| 私有桶 | `uploads/` | `false` | `signedUrl`（签名 URL） | 5 分钟 |

#### 5.3.4 代码注释说明

```typescript
// server/models/Attachment.ts
/**
 * Whether the attachment is stored in a public bucket. This does not relate
 * to the ACL of the attachment itself. Previously "public" attachments were
 * stored in a separate bucket – now all attachments are stored in a private
 * bucket and ACL is checked per attachment.
 */
get isStoredInPublicBucket() {
  // ...
}
```

**关键理解**：
- `isStoredInPublicBucket` 是**历史遗留概念**
- 以前：公开附件存在单独的公开桶，私有附件存在私有桶
- 现在：所有附件都存在私有桶，通过 **ACL 字段**控制访问权限
- `isStoredInPublicBucket` 现在只影响**缓存策略和 URL 类型**，不影响**访问权限**

### 5.4 权限边界矩阵

综合两个维度，附件的权限边界如下：

| ACL | 存储位置 | isPrivate | 访问权限 | 未登录用户 | URL 类型 | 缓存时间 |
|-----|----------|-----------|----------|------------|----------|----------|
| `public-read` | 公开桶 | `false` | **完全公开** | ✅ 允许 | canonicalUrl | 7 天 |
| `public-read` | 私有桶 | `false` | **完全公开** | ✅ 允许 | signedUrl | 5 分钟 |
| `private` | 公开桶 | `true` | 同团队成员 | ❌ 拒绝 | canonicalUrl | 7 天 |
| `private` | 私有桶 | `true` | 同团队成员 | ❌ 拒绝 | signedUrl | 5 分钟 |

### 5.5 实际场景分析

#### 场景 1：用户头像

```typescript
// preset = AttachmentPreset.Avatar
// → acl = "public-read"
// → key = uploads/{userId}/{id}/{hash}

const attachment = await attachmentCreator({
  preset: AttachmentPreset.Avatar,  // 强制 public-read
  // ...
});
```

**权限状态**：
- `acl = "public-read"` → `isPrivate = false`
- 任何人（包括未登录用户）都可以访问
- 这是**有意的设计**：头像需要在登录页面、公开分享等场景显示

#### 场景 2：文档附件（默认配置）

```typescript
// preset = AttachmentPreset.DocumentAttachment
// → acl = env.AWS_S3_ACL (通常是 "private")
// → key = uploads/{userId}/{id}/{name}
```

**权限状态**：
- `acl = "private"` → `isPrivate = true`
- 仅同团队成员可以访问
- 未登录用户或其他团队用户会收到 403

#### 场景 3：导入的公开附件（历史数据）

```typescript
// 从旧系统导入，key 可能有 public/ 前缀
// key = public/{uuid}/test.png
// acl = "public-read"
```

**权限状态**：
- `acl = "public-read"` → `isPrivate = false`
- 完全公开访问
- `isStoredInPublicBucket = true` → 使用 canonicalUrl，缓存 7 天

### 5.6 上传与下载权限对比

| 检查点 | 上传入口（attachments.create） | 下载入口（attachments.redirect） |
|--------|-------------------------------|---------------------------------|
| **认证要求** | 必需（`auth()`） | 可选（`auth({ optional: true })`） |
| **文档权限检查** | 有（documentId 存在时检查 `update` 权限） | **无** |
| **团队权限检查** | 有（`createAttachment`） | 仅私有附件检查 `teamId` 匹配 |
| **公开附件** | N/A（创建时设置 ACL） | **完全公开，无需任何检查** |
| **策略目的** | 防止未授权上传 | 允许灵活的附件访问（头像、公开分享等） |

### 5.7 文档权限与附件权限的关系

这是最关键的设计决策：

#### 上传时：检查文档权限

```typescript
// attachments.create
if (preset === AttachmentPreset.DocumentAttachment && documentId) {
  const document = await Document.findByPk(documentId, { userId: user.id });
  authorize(user, "update", document);  // 必须有文档更新权限
}
```

#### 下载时：不检查文档权限

```typescript
// attachments.redirect
// 只检查 isPrivate 和 teamId，不检查 documentId 关联的文档权限
if (attachment.isPrivate && attachment.teamId !== user?.teamId) {
  throw AuthorizationError();
}
```

#### 代码中的明确说明

```typescript
// 来自 attachments.ts:279-286 的注释
// Private attachments are accessible to any member of the workspace they
// belong to. This is intentional and not a permission bypass – attachments
// are owned by the workspace (team), not by individual documents. Checking
// document-level permissions here would be insufficient anyway as attachments
// can exist independently of documents.
```

**翻译**：
> 私有附件对其所属工作区的任何成员都可访问。这是有意的设计，而非权限绕过——附件属于工作区（team），而非单个文档。在这里检查文档级别权限是不够的，因为附件可以独立于文档存在。

#### 设计意图分析

| 考虑因素 | 说明 |
|----------|------|
| **附件可以独立存在** | 头像、表情符号、导入文件等不需要关联文档 |
| **附件可能被多处引用** | 同一附件可能被复制到多个文档 |
| **团队级共享** | Outline 是团队协作工具，默认假设团队成员互相信任 |
| **简化实现** | 不需要维护附件-文档的权限同步 |

#### 安全边界

虽然附件在团队内共享，但仍有明确的安全边界：

```
┌─────────────────────────────────────────────────────────────┐
│                      安全边界                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Team A                                                    │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐   │
│  │  Document 1 │    │  Document 2 │    │  Attachment │   │
│  │  (私有)     │    │  (公开)     │    │  (私有)     │   │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘   │
│         │                  │                  │            │
│         └──────────────────┼──────────────────┘            │
│                            │                               │
│                     ┌──────┴──────┐                        │
│                     │  User A     │                        │
│                     │  (Team A)   │ ◄── 可以访问所有附件    │
│                     └─────────────┘                        │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Team B                                                    │
│  ┌─────────────┐                                           │
│  │  User B     │ ◄── 无法访问 Team A 的私有附件            │
│  │  (Team B)   │     (403 AuthorizationError)             │
│  └─────────────┘                                           │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  未登录用户                                                 │
│  ┌─────────────┐                                           │
│  │             │ ◄── 无法访问私有附件                       │
│  │             │     但可以访问 public-read 附件           │
│  └─────────────┘                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 签名 URL 生成机制

### 6.1 上传签名 URL（Presigned POST）

#### 6.1.1 S3 预签名 POST

```typescript
// server/storage/files/S3Storage.ts
public async getPresignedPost(
  _ctx: AppContext,
  key: string,
  _acl: string,
  maxUploadSize: number,
  contentType = "image"
) {
  const params: PresignedPostOptions = {
    Bucket: env.AWS_S3_UPLOAD_BUCKET_NAME as string,
    Key: key,
    Conditions: compact([
      ["content-length-range", 0, maxUploadSize],
      ["starts-with", "$Content-Type", contentType],
      ["starts-with", "$Cache-Control", ""],
    ]),
    Fields: {
      "Content-Disposition": this.getContentDisposition(contentType),
      key,
      ...(env.AWS_S3_ACL && { ACL: env.AWS_S3_ACL as ObjectCannedACL }),
    },
    Expires: 3600,  // 1 小时有效期
  };

  return createPresignedPost(this.client, params);
}
```

**返回值**：
```typescript
interface PresignedPost {
  url: string;           // S3 上传端点
  fields: {              // 必须包含的表单字段
    key: string;
    Policy: string;      // Base64 编码的策略
    "X-Amz-Algorithm": string;
    "X-Amz-Credential": string;
    "X-Amz-Date": string;
    "X-Amz-Signature": string;  // AWS 签名
  };
}
```

#### 6.1.2 本地存储模拟预签名

```typescript
// server/storage/files/LocalStorage.ts
public async getPresignedPost(
  ctx: AppContext,
  key: string,
  acl: string,
  maxUploadSize: number,
  contentType = "image"
): Promise<Partial<PresignedPost>> {
  return Promise.resolve({
    url: this.getUrlForKey(key),  // /api/files.get?key={key}
    fields: {
      key,
      acl,
      maxUploadSize: String(maxUploadSize),
      contentType,
      [CSRF.fieldName]: ctx.cookies.get(CSRF.cookieName) || "",  // CSRF Token
    },
  });
}
```

### 6.2 下载签名 URL（Signed URL）

#### 6.2.1 S3 签名 URL

```typescript
// server/storage/files/S3Storage.ts
public getSignedUrl = async (
  key: string,
  expiresIn = S3Storage.defaultSignedUrlExpires  // 300 秒 = 5 分钟
) => {
  const isDocker = env.AWS_S3_UPLOAD_BUCKET_URL.match(/http:\/\/s3:/);
  const params = {
    Bucket: this.getBucket(),
    Key: key,
  };

  if (isDocker) {
    // Docker 开发环境：直接返回本地 URL
    return `${this.getPublicEndpoint()}/${key}`;
  } else {
    // 生产环境：使用 AWS 签名 V4
    const clampedExpiresIn = Math.min(
      expiresIn,
      S3Storage.maxSignedUrlExpires  // 最多 7 天（AWS 限制）
    );

    const command = new GetObjectCommand(params);
    const url = await getSignedUrl(this.client, command, {
      expiresIn: clampedExpiresIn,
    });

    // 替换为加速 URL（如果配置了）
    if (env.AWS_S3_ACCELERATE_URL) {
      return url.replace(
        env.AWS_S3_UPLOAD_BUCKET_URL,
        env.AWS_S3_ACCELERATE_URL
      );
    }

    return url;
  }
};
```

#### 6.2.2 本地存储签名 URL（JWT）

```typescript
// server/storage/files/LocalStorage.ts
public getSignedUrl = async (
  key: string,
  expiresIn = LocalStorage.defaultSignedUrlExpires
) => {
  const sig = JWT.sign(
    {
      key,
      type: "attachment",
    },
    env.SECRET_KEY,
    {
      expiresIn,
    }
  );
  return Promise.resolve(`${env.URL}/api/files.get?sig=${sig}`);
};
```

**JWT Payload**：
- `key` - 文件存储路径
- `type` - 固定为 "attachment"
- `exp` - 过期时间

---

## 7. 附件生命周期管理

### 7.1 附件创建

创建流程有三种方式：

1. **标准上传流程**（attachments.create）：
   - 创建 Attachment 记录
   - 生成预签名 URL
   - 客户端直接上传到存储后端

2. **从 URL 创建**（attachments.createFromUrl）：
   - 创建 Attachment 记录
   - 调度后台任务 `UploadAttachmentFromUrlTask`
   - 服务端从远程 URL 下载并存储

3. **程序化创建**（attachmentCreator）：
   - 直接从 Buffer 或 URL 创建
   - 用于导入、头像同步等场景

### 7.2 附件删除

**钩子机制**：
```typescript
// server/models/Attachment.ts:175-190
@BeforeDestroy
static async deleteAttachmentFromS3(model: Attachment) {
  try {
    await FileStorage.deleteFile(model.key);
  } catch (err) {
    // S3 删除失败不阻塞数据库记录删除
    Logger.warn(
      `Failed to delete attachment file ${model.key} from storage`,
      {
        id: model.id,
        teamId: model.teamId,
        message: err.message,
      }
    );
  }
}
```

**删除时的权限检查**：
```typescript
// attachments.delete
if (attachment.documentId) {
  const document = await Document.findByPk(attachment.documentId, {
    userId: user.id,
    transaction,
  });
  authorize(user, "update", document);  // 检查文档更新权限
}

authorize(user, "delete", attachment);  // 检查附件删除权限
```

### 7.3 临时附件过期

部分附件类型有过期时间：

```typescript
// server/models/helpers/AttachmentHelper.ts
static presetToExpiry(preset: AttachmentPreset) {
  switch (preset) {
    case AttachmentPreset.Import:
    case AttachmentPreset.WorkspaceImport:
      return addHours(new Date(), 24);  // 24 小时后过期
    default:
      return undefined;  // 永不过期
  }
}
```

---

## 8. 安全考虑

### 8.1 路径遍历防护

**Key 验证**：
```typescript
// server/models/Attachment.ts
@BeforeCreate
static async sanitizeKey(model: Attachment) {
  model.key = ValidateKey.sanitize(model.key);
  return model;
}

@BeforeUpdate
static async preventKeyChange(model: Attachment) {
  if (model.changed("key")) {
    throw ValidationError("Cannot change the key of an attachment");
  }
}
```

**本地存储路径解析**：
```typescript
// server/storage/files/LocalStorage.ts
private getFilePath(key: string) {
  invariant(
    env.FILE_STORAGE_LOCAL_ROOT_DIR,
    "FILE_STORAGE_LOCAL_ROOT_DIR is required"
  );
  
  return safeResolvePath(env.FILE_STORAGE_LOCAL_ROOT_DIR, key);
}
```

使用 `safeResolvePath`（来自 `resolve-path` 库）防止路径遍历攻击。

### 8.2 CSRF 防护

本地存储上传时包含 CSRF token：
```typescript
// server/storage/files/LocalStorage.ts
[CSRF.fieldName]: ctx.cookies.get(CSRF.cookieName) || "",
```

### 8.3 内容安全

**Content-Disposition 策略**：
```typescript
// server/storage/files/BaseStorage.ts
public getContentDisposition(contentType?: string) {
  if (!contentType) {
    return "attachment";
  }

  if (
    FileHelper.isAudio(contentType) ||
    FileHelper.isVideo(contentType) ||
    this.safeInlineContentTypes.includes(contentType)
  ) {
    return "inline";  // 安全类型可以内联显示
  }

  return "attachment";  // 其他类型作为附件下载
}

protected safeInlineContentTypes = [
  "application/pdf",
  "image/png",
  "image/jpeg",
  "image/gif",
  "image/webp",
];
```

**注意**：SVG 被故意排除，因为 SVG 文件可以包含 JavaScript，存在安全风险。

### 8.4 速率限制

附件创建接口有速率限制：
```typescript
// server/routes/api/attachments/attachments.ts
router.post(
  "attachments.create",
  rateLimiter(RateLimiterStrategy.TwentyFivePerMinute),  // 每分钟 25 次
  // ...
);
```

### 8.5 公开附件的安全考虑

由于 `public-read` 附件可以被任何人访问，使用时需要注意：

| 风险 | 说明 | 缓解措施 |
|------|------|----------|
| 敏感信息泄露 | 如果上传了敏感文件到公开附件 | 默认使用 `private` ACL |
| 未授权访问 | 未登录用户可以访问 | 仅对头像等必需场景使用 `public-read` |
| 链接分享 | 知道 URL 就可以访问 | 私有附件使用团队级权限控制 |

---

## 9. 关键设计决策总结

### 9.1 存储架构

| 决策 | 说明 |
|------|------|
| 双存储后端支持 | S3 用于生产，本地存储用于开发 |
| 抽象接口设计 | BaseStorage 统一接口，便于扩展其他存储后端 |
| Bucket 前缀分层 | `uploads/`、`public/`、`avatars/` 用于不同场景 |

### 9.2 入口职责分离

| 入口 | 职责 | 权限策略 |
|------|------|----------|
| **上传入口** | 创建附件记录，生成上传凭证 | **严格**：认证必需，检查文档/团队权限 |
| **下载入口** | 验证访问权限，重定向到实际 URL | **宽松**：认证可选，仅检查 ACL 和团队归属 |

### 9.3 权限模型

| 决策 | 说明 |
|------|------|
| **ACL 维度** | `private` 需要团队归属检查，`public-read` 完全公开 |
| **存储位置维度** | 仅影响缓存策略和 URL 类型，不影响访问权限 |
| **工作区所有权** | 附件属于 team，而非 document |
| **团队内共享** | 同一团队成员可访问所有私有附件 |
| **上传时检查文档权限** | 防止未授权用户上传附件到文档 |
| **下载时不检查文档权限** | 附件独立于文档存在，简化权限模型 |

### 9.4 安全策略

| 决策 | 说明 |
|------|------|
| 签名 URL 模式 | 私有附件通过 API 端点验证后重定向到签名 URL |
| 路径遍历防护 | 多层验证：ValidateKey.sanitize + safeResolvePath |
| 内容类型限制 | SVG 等危险类型不作为内联内容显示 |
| 速率限制 | 防止滥用上传接口 |

---

## 10. 代码索引

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| 附件模型 | `server/models/Attachment.ts` | 全文 |
| S3 存储 | `server/storage/files/S3Storage.ts` | 全文 |
| 本地存储 | `server/storage/files/LocalStorage.ts` | 全文 |
| 存储基类 | `server/storage/files/BaseStorage.ts` | 全文 |
| 存储工厂 | `server/storage/files/index.ts` | 1-8 |
| 附件路由 | `server/routes/api/attachments/attachments.ts` | 全文 |
| 附件策略 | `server/policies/attachment.ts` | 全文 |
| 附件助手 | `server/models/helpers/AttachmentHelper.ts` | 全文 |
| 附件创建命令 | `server/commands/attachmentCreator.ts` | 全文 |
| MCP 附件工具 | `server/tools/attachments.ts` | 全文 |
| 附件测试 | `server/routes/api/attachments/attachments.test.ts` | 全文 |

---

## 11. 环境变量配置参考

### 11.1 通用配置

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `FILE_STORAGE` | 存储类型：`local` 或 `s3` | `s3` |
| `SECRET_KEY` | JWT 签名密钥（本地存储签名 URL） | 必需 |
| `URL` | 应用 URL（本地存储签名 URL） | 必需 |

### 11.2 S3 配置

| 变量名 | 说明 |
|--------|------|
| `AWS_REGION` | S3 区域 |
| `AWS_S3_UPLOAD_BUCKET_NAME` | 存储桶名称 |
| `AWS_S3_UPLOAD_BUCKET_URL` | 存储桶访问 URL |
| `AWS_S3_ACCELERATE_URL` | S3 Transfer Acceleration URL（可选） |
| `AWS_S3_FORCE_PATH_STYLE` | 使用路径风格访问（boolean） |
| `AWS_S3_ACL` | 默认 ACL：`private` 或 `public-read` |
| `AWS_ACCESS_KEY_ID` | AWS 访问密钥 ID |
| `AWS_SECRET_ACCESS_KEY` | AWS 秘密访问密钥 |

### 11.3 本地存储配置

| 变量名 | 说明 |
|--------|------|
| `FILE_STORAGE_LOCAL_ROOT_DIR` | 本地存储根目录 |

### 11.4 上传限制

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `FILE_STORAGE_UPLOAD_MAX_SIZE` | 普通附件最大大小 |  |
| `FILE_STORAGE_IMPORT_MAX_SIZE` | 导入文件最大大小 |  |
| `FILE_STORAGE_WORKSPACE_IMPORT_MAX_SIZE` | 工作区导入最大大小 |  |

---

## 附录：Preset 类型说明

```typescript
// @shared/types
enum AttachmentPreset {
  Avatar,             // 用户/团队头像 → ACL: public-read
  DocumentAttachment, // 文档附件 → ACL: 由 AWS_S3_ACL 决定
  Emoji,              // 自定义表情 → ACL: 由 AWS_S3_ACL 决定
  Import,             // 导入临时文件 → ACL: 由 AWS_S3_ACL 决定, 24h 过期
  WorkspaceImport,    // 工作区导入 → ACL: 由 AWS_S3_ACL 决定, 24h 过期
}
```

---

**报告生成时间**：2026-05-05
**分析基于代码版本**：当前工作目录
