# Outline 附件存储机制分析报告

## 1. 概述

Outline 是一个支持实时协作的知识库应用，其附件存储系统设计灵活，支持两种存储后端：
- **S3 对象存储**（默认，适用于生产环境）
- **本地文件系统存储**（适用于开发和小型部署）

本报告详细分析 Outline 的附件存储机制、签名 URL 生成流程以及权限控制策略。

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

#### 2.3.2 文件存储路径

文件存储路径（key）格式如下：
```
uploads/{userId}/{attachmentId}/{fileName}
```

由 `AttachmentHelper.getKey()` 方法生成：

```typescript
// server/models/helpers/AttachmentHelper.ts
static getKey({ id, name, userId }: { id: string; name: string; userId: string }) {
  const keyPrefix = `${Buckets.uploads}/${userId}/${id}`;
  return ValidateKey.sanitize(
    `${keyPrefix}/${name.slice(0, this.maximumFileNameLength)}`
  );
}
```

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
{FILE_STORAGE_LOCAL_ROOT_DIR}/uploads/{userId}/{attachmentId}/{fileName}
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

## 3. 上传签名 URL 生成机制

### 3.1 整体流程

附件上传采用预签名 URL 模式，流程如下：

```
1. 客户端请求 attachments.create 接口
   ↓
2. 服务端创建 Attachment 数据库记录
   ↓
3. 服务端调用 FileStorage.getPresignedPost() 生成上传签名
   ↓
4. 返回 uploadUrl 和 form 字段给客户端
   ↓
5. 客户端使用返回的信息直接向存储后端上传文件
```

### 3.2 S3 预签名 POST 生成

**核心方法**：`S3Storage.getPresignedPost()`

```typescript
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

**预签名参数说明**：
- `Bucket` - 目标存储桶
- `Key` - 文件存储路径
- `Conditions` - 上传条件限制：
  - 文件大小限制（0 到 maxUploadSize）
  - Content-Type 必须以指定前缀开头
  - Cache-Control 可以为空
- `Fields` - 必须包含的表单字段：
  - Content-Disposition（内联或附件）
  - key（文件路径）
  - ACL（如果配置了）
- `Expires` - 签名有效期（3600 秒 = 1 小时）

**返回值结构**：
```typescript
interface PresignedPost {
  url: string;           // 上传端点 URL
  fields: {              // 必须包含的表单字段
    key: string;
    Policy: string;
    "X-Amz-Algorithm": string;
    "X-Amz-Credential": string;
    "X-Amz-Date": string;
    "X-Amz-Signature": string;
    // ... 其他字段
  };
}
```

### 3.3 本地存储模拟预签名

**核心方法**：`LocalStorage.getPresignedPost()`

本地存储不使用真实的 S3 签名，而是返回模拟的 POST 信息：

```typescript
public async getPresignedPost(
  ctx: AppContext,
  key: string,
  acl: string,
  maxUploadSize: number,
  contentType = "image"
): Promise<Partial<PresignedPost>> {
  return Promise.resolve({
    url: this.getUrlForKey(key),
    fields: {
      key,
      acl,
      maxUploadSize: String(maxUploadSize),
      contentType,
      [CSRF.fieldName]: ctx.cookies.get(CSRF.cookieName) || "",
    },
  });
}
```

**关键差异**：
- `url` 指向本地 API 端点 `/api/files.get?key={key}`
- 包含 CSRF token 用于安全验证
- 不需要 AWS 签名相关字段

### 3.4 附件创建接口详解

**核心文件**：`server/routes/api/attachments/attachments.ts` - `attachments.create` 路由

#### 3.4.1 权限检查

在生成上传签名之前，系统会进行权限验证：

```typescript
// 所有用户类型都可以上传头像，无需额外授权
if (preset === AttachmentPreset.Avatar) {
  assertIn(contentType, AttachmentValidation.avatarContentTypes);
} else {
  // 文档附件需要检查文档更新权限
  if (preset === AttachmentPreset.DocumentAttachment && documentId) {
    const document = await Document.findByPk(documentId, {
      userId: user.id,
      transaction,
    });
    authorize(user, "update", document);  // 检查文档更新权限
  }
  // 表情符号类型检查
  if (preset === AttachmentPreset.Emoji) {
    assertIn(contentType, AttachmentValidation.emojiContentTypes);
  }
  // 检查团队附件创建权限
  authorize(user, "createAttachment", user.team);
}
```

#### 3.4.2 附件记录创建

```typescript
const modelId = id ?? randomUUID();
const acl = AttachmentHelper.presetToAcl(preset);
const key = AttachmentHelper.getKey({
  id: modelId,
  name,
  userId: user.id,
});

const attachment = await Attachment.createWithCtx(ctx, {
  id: modelId,
  key,
  acl,
  size,
  expiresAt: AttachmentHelper.presetToExpiry(preset),
  contentType,
  documentId,
  teamId: user.teamId,
  userId: user.id,
});
```

#### 3.4.3 生成预签名并返回

```typescript
const presignedPost = await FileStorage.getPresignedPost(
  ctx,
  key,
  acl,
  maxUploadSize,
  contentType
);

ctx.body = {
  data: {
    uploadUrl: FileStorage.getUploadUrl(),
    form: {
      "Cache-Control": "max-age=31557600",  // 1 年缓存
      "Content-Type": contentType,
      ...presignedPost.fields,
    },
    attachment: {
      ...presentAttachment(attachment),
      url: attachment.redirectUrl,  // 返回重定向 URL 而非直接 URL
    },
  },
};
```

---

## 4. 附件访问权限控制

### 4.1 附件访问流程

附件访问采用重定向 + 签名 URL 模式：

```
1. 客户端请求 attachment.redirectUrl
   格式：/api/attachments.redirect?id={attachmentId}
   ↓
2. 服务端验证权限
   ↓
3. 服务端生成签名 URL（私有附件）或直接 URL（公共附件）
   ↓
4. 302 重定向到最终 URL
```

### 4.2 访问权限检查机制

**核心方法**：`handleAttachmentsRedirect()` (server/routes/api/attachments/attachments.ts:269-307)

```typescript
const handleAttachmentsRedirect = async (
  ctx: APIContext<T.AttachmentsRedirectReq>
) => {
  const id = (ctx.input.body.id ?? ctx.input.query.id) as string;

  const user = ctx.state.auth?.user;
  const attachment = await Attachment.findByPk(id, {
    rejectOnEmpty: true,
  });

  // 关键权限检查逻辑
  // 私有附件仅对所属工作区的成员可访问
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

### 4.3 权限继承关系分析

#### 4.3.1 核心设计理念

根据代码注释和实现，Outline 的附件权限设计采用**工作区级别的权限模型**，而非文档级别的细粒度控制：

```typescript
// 来自 attachments.ts:279-286 的注释
// Private attachments are accessible to any member of the workspace they
// belong to. This is intentional and not a permission bypass – attachments
// are owned by the workspace (team), not by individual documents. Checking
// document-level permissions here would be insufficient anyway as attachments
// can exist independently of documents.
```

**关键要点**：
1. **附件所有权**：附件属于工作区（team），而非单独的文档
2. **访问范围**：私有附件对工作区内的所有成员可见
3. **设计原因**：
   - 附件可以独立于文档存在
   - 检查文档级别权限不够充分
   - 这是**有意的设计决策**，而非权限绕过

#### 4.3.2 附件与文档的关系

Attachment 模型与 Document 的关联是可选的：

```typescript
// server/models/Attachment.ts:254-259
@BelongsTo(() => Document, "documentId")
document: Document;

@ForeignKey(() => Document)
@Column(DataType.UUID)
documentId: string | null;  // 可以为 null
```

**关联场景**：
- 当 `documentId` 有值时：附件关联到特定文档
- 当 `documentId` 为 null 时：附件独立存在（如头像、表情符号等）

#### 4.3.3 权限检查对比

| 操作 | 权限检查 | 检查级别 |
|------|----------|----------|
| **创建附件** | `authorize(user, "update", document)` + `authorize(user, "createAttachment", team)` | 文档更新权限 + 团队创建权限 |
| **访问附件** | `attachment.teamId === user?.teamId` | 仅团队级别 |
| **删除附件** | `authorize(user, "delete", attachment)` | 所有者或管理员 |
| **列出附件** | 按 `userId` 或 `documentId` 过滤 | 用户/文档级别 |

#### 4.3.4 附件级别的权限策略

**核心文件**：`server/policies/attachment.ts`

```typescript
import { Attachment, User, Team } from "@server/models";
import { allow } from "./cancan";
import { and, isOwner, isTeamModel, or } from "./utils";

// 团队成员可以创建附件
allow(User, "createAttachment", Team, isTeamModel);

// 附件的读取、更新、删除权限
allow(User, ["read", "update", "delete"], Attachment, (actor, attachment) =>
  and(
    isTeamModel(actor, attachment),        // 必须是同一团队
    or(actor.isAdmin, isOwner(actor, attachment))  // 管理员或所有者
  )
);
```

**策略分析**：
1. `createAttachment` - 任何团队成员都可以创建附件
2. `read`/`update`/`delete` - 需要满足：
   - 是同一团队成员 (`isTeamModel`)
   - 并且是管理员 **或者** 是附件的创建者 (`isOwner`)

#### 4.3.5 实际访问控制的差异

注意：**策略文件中的权限定义**与**实际重定向端点的实现**存在差异：

**策略文件定义** (`attachment.ts`)：
- `read` 权限需要 `isTeamModel AND (isAdmin OR isOwner)`

**实际实现** (`attachments.ts:handleAttachmentsRedirect`)：
- 仅检查 `attachment.teamId === user?.teamId`

**这意味着**：
- 任何团队成员都可以访问团队内的任何私有附件
- 策略文件中的 `isOwner` 和 `isAdmin` 限制在访问时**未被应用**

这是一个重要的设计决策，体现了 Outline "附件属于工作区" 的核心理念。

### 4.4 ACL 类型与访问方式

#### 4.4.1 ACL 类型定义

```typescript
// server/models/Attachment.ts:61-64
@Default("public-read")
@IsIn([["private", "public-read"]])
@Column
acl: string;
```

#### 4.4.2 ACL 与预设的映射

```typescript
// server/models/helpers/AttachmentHelper.ts:72-79
static presetToAcl(preset: AttachmentPreset) {
  switch (preset) {
    case AttachmentPreset.Avatar:
      return "public-read";  // 头像始终公开
    default:
      return env.AWS_S3_ACL;  // 其他使用环境变量配置
  }
}
```

#### 4.4.3 URL 类型解析

根据 ACL 和存储位置，附件有三种 URL 类型：

| URL 类型 | 获取方式 | 使用场景 |
|----------|----------|----------|
| `url` | `attachment.url` | 智能选择，根据 isPrivate 返回 redirectUrl 或 canonicalUrl |
| `redirectUrl` | `attachment.redirectUrl` | 私有附件，先访问 API 端点验证权限 |
| `canonicalUrl` | `attachment.canonicalUrl` | 公共附件，直接指向存储后端 |
| `signedUrl` | `attachment.signedUrl` | 临时签名 URL，有过期时间 |

**核心逻辑**：
```typescript
// server/models/Attachment.ts:117-119
get url() {
  return this.isPrivate ? this.redirectUrl : this.canonicalUrl;
}
```

### 4.5 签名 URL 生成机制

#### 4.5.1 S3 签名 URL

```typescript
// server/storage/files/S3Storage.ts:141-174
public getSignedUrl = async (
  key: string,
  expiresIn = S3Storage.defaultSignedUrlExpires
) => {
  const isDocker = env.AWS_S3_UPLOAD_BUCKET_URL.match(/http:\/\/s3:/);
  const params = {
    Bucket: this.getBucket(),
    Key: key,
  };

  if (isDocker) {
    // Docker 开发环境直接返回本地 URL
    return `${this.getPublicEndpoint()}/${key}`;
  } else {
    // 生产环境使用 AWS 签名 V4
    const clampedExpiresIn = Math.min(
      expiresIn,
      S3Storage.maxSignedUrlExpires  // 最多 7 天
    );

    const command = new GetObjectCommand(params);
    const url = await getSignedUrl(this.client, command, {
      expiresIn: clampedExpiresIn,
    });

    // 如果配置了加速 URL，替换为加速域名
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

**签名 URL 有效期**：
- 默认：`defaultSignedUrlExpires = 300` 秒（5 分钟）
- 最大：`maxSignedUrlExpires = Week.seconds`（7 天，AWS S3 签名 V4 限制）

#### 4.5.2 本地存储签名 URL

本地存储使用 JWT 实现签名 URL：

```typescript
// server/storage/files/LocalStorage.ts:113-128
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
- `exp` - 过期时间（由 expiresIn 决定）

---

## 5. 附件生命周期管理

### 5.1 附件创建

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
   - 用于导入、迁移等场景

### 5.2 附件删除

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

**设计要点**：
- 使用 `@BeforeDestroy` 钩子在删除数据库记录前删除存储文件
- 存储删除失败不阻塞数据库删除（容错设计）
- 记录错误日志便于后续排查和清理

### 5.3 临时附件过期

部分附件类型有过期时间：

```typescript
// server/models/helpers/AttachmentHelper.ts:87-95
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

**过期附件**：
- 导入相关的临时附件 24 小时后过期
- `expiresAt` 字段标记过期时间
- 需要额外的清理任务来删除过期附件

---

## 6. 安全考虑

### 6.1 路径遍历防护

**Key 验证**：
```typescript
// server/models/Attachment.ts:162-166
@BeforeCreate
static async sanitizeKey(model: Attachment) {
  model.key = ValidateKey.sanitize(model.key);
  return model;
}
```

**Key 变更保护**：
```typescript
// server/models/Attachment.ts:168-173
@BeforeUpdate
static async preventKeyChange(model: Attachment) {
  if (model.changed("key")) {
    throw ValidationError("Cannot change the key of an attachment");
  }
}
```

**本地存储路径解析**：
```typescript
// server/storage/files/LocalStorage.ts:180-187
private getFilePath(key: string) {
  invariant(
    env.FILE_STORAGE_LOCAL_ROOT_DIR,
    "FILE_STORAGE_LOCAL_ROOT_DIR is required"
  );
  
  return safeResolvePath(env.FILE_STORAGE_LOCAL_ROOT_DIR, key);
}
```

使用 `safeResolvePath`（来自 `resolve-path` 库）防止路径遍历攻击。

### 6.2 CSRF 防护

本地存储上传时包含 CSRF token：
```typescript
// server/storage/files/LocalStorage.ts:32
[CSRF.fieldName]: ctx.cookies.get(CSRF.cookieName) || "",
```

### 6.3 内容安全

**Content-Disposition 策略**：
```typescript
// server/storage/files/BaseStorage.ts:277-291
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
```

**安全内联类型**：
```typescript
// server/storage/files/BaseStorage.ts:297-303
protected safeInlineContentTypes = [
  "application/pdf",
  "image/png",
  "image/jpeg",
  "image/gif",
  "image/webp",
];
```

**注意**：SVG 被故意排除，因为 SVG 文件可以包含 JavaScript，存在安全风险。

### 6.4 速率限制

附件创建接口有速率限制：
```typescript
// server/routes/api/attachments/attachments.ts:83-84
router.post(
  "attachments.create",
  rateLimiter(RateLimiterStrategy.TwentyFivePerMinute),  // 每分钟 25 次
  // ...
);
```

---

## 7. 关键设计决策总结

### 7.1 存储架构

| 决策 | 说明 |
|------|------|
| 双存储后端支持 | S3 用于生产，本地存储用于开发 |
| 抽象接口设计 | BaseStorage 统一接口，便于扩展其他存储后端 |
| 路径命名规范 | `uploads/{userId}/{attachmentId}/{fileName}` 便于追踪和管理 |

### 7.2 权限模型

| 决策 | 说明 |
|------|------|
| 工作区所有权 | 附件属于 team，而非 document |
| 团队内共享 | 同一团队成员可访问所有私有附件 |
| 创建时检查文档权限 | 上传时验证文档更新权限，但访问时不检查 |
| 独立附件支持 | 附件可以不关联任何文档（头像、表情等） |

### 7.3 安全策略

| 决策 | 说明 |
|------|------|
| 签名 URL 模式 | 私有附件通过 API 端点验证后重定向到签名 URL |
| 路径遍历防护 | 多层验证：ValidateKey.sanitize + safeResolvePath |
| 内容类型限制 | SVG 等危险类型不作为内联内容显示 |
| 速率限制 | 防止滥用上传接口 |

---

## 8. 代码索引

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

---

## 9. 环境变量配置参考

### 9.1 通用配置

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `FILE_STORAGE` | 存储类型：`local` 或 `s3` | `s3` |
| `SECRET_KEY` | JWT 签名密钥（本地存储签名 URL） | 必需 |
| `URL` | 应用 URL（本地存储签名 URL） | 必需 |

### 9.2 S3 配置

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

### 9.3 本地存储配置

| 变量名 | 说明 |
|--------|------|
| `FILE_STORAGE_LOCAL_ROOT_DIR` | 本地存储根目录 |

### 9.4 上传限制

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `FILE_STORAGE_UPLOAD_MAX_SIZE` | 普通附件最大大小 |  |
| `FILE_STORAGE_IMPORT_MAX_SIZE` | 导入文件最大大小 |  |
| `FILE_STORAGE_WORKSPACE_IMPORT_MAX_SIZE` | 工作区导入最大大小 |  |

---

**报告生成时间**：2026-05-05
**分析基于代码版本**：当前工作目录
