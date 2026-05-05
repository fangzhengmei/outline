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

## 3. 上传入口职责边界深度分析

### 3.1 核心概念澄清：上传信息字段分类

在深入分析上传流程之前，必须明确一个**关键区分**：预签名返回信息中的字段，哪些是实际使用的，哪些只是辅助信息。

#### 3.1.1 服务端返回值组装逻辑

**核心文件**：`server/routes/api/attachments/attachments.ts:141-162`

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
    uploadUrl: FileStorage.getUploadUrl(),           // 字段 A：实际上传端点
    form: {
      "Cache-Control": "max-age=31557600",            // 服务端添加的字段
      "Content-Type": contentType,                     // 服务端添加的字段
      ...presignedPost.fields,                         // 字段 B：来自 getPresignedPost().fields
    },
    attachment: {
      ...presentAttachment(attachment),
      url: attachment.redirectUrl,
    },
  },
};
```

**关键点**：
- `presignedPost` 包含两个属性：`url` 和 `fields`
- `presignedPost.url` **没有被使用**
- `presignedPost.fields` 被**展开合并到 form 中**

#### 3.1.2 客户端实际上传逻辑

**核心文件**：`app/utils/files.ts:123`

```typescript
// 第 123 行：实际上传
xhr.open("POST", data.uploadUrl, true);
xhr.send(formData);
```

**关键点**：
- 客户端使用的是 `data.uploadUrl`，**不是** `presignedPost.url`
- 客户端使用的是 `data.form` 中的字段，这些字段包含了 `presignedPost.fields`
- `presignedPost.url` 完全没有被直接使用

#### 3.1.3 上传信息字段分类总表

| 字段类别 | 字段名称 | 来源 | 客户端是否使用 | 实际用途 |
|----------|----------|------|----------------|----------|
| **实际上传端点** | `uploadUrl` | `FileStorage.getUploadUrl()` | ✅ 实际使用 | 客户端 POST 的目标 URL |
| **表单验证字段** | `form` | 服务端组装（含 `presignedPost.fields`） | ✅ 实际使用 | S3 或服务端用这些验证上传 |
| **服务端添加字段** | `Cache-Control`, `Content-Type` | 服务端在返回前添加 | ✅ 实际使用 | 控制缓存和内容类型 |
| **辅助信息（未使用）** | `presignedPost.url` | `FileStorage.getPresignedPost().url` | ❌ 未直接使用 | 仅为接口兼容，实际不用 |
| **实际使用的签名字段** | `presignedPost.fields` | `FileStorage.getPresignedPost().fields` | ✅ 实际使用 | 被合并到 form 中 |

---

### 3.2 S3 存储：上传端点与预签名 URL 的关系

#### 3.2.1 S3 相关方法实现

**核心文件**：`server/storage/files/S3Storage.ts`

```typescript
// 方法 1: getUploadUrl() - 返回实际上传端点
public getUploadUrl(isServerUpload?: boolean) {
  return this.getPublicEndpoint(isServerUpload);
}

private getPublicEndpoint(isServerUpload?: boolean) {
  if (env.AWS_S3_ACCELERATE_URL) {
    return env.AWS_S3_ACCELERATE_URL;
  }
  // ... 构建 S3 端点 URL
  return `${host}/${isServerUpload && isDocker ? "s3/" : ""}${
    env.AWS_S3_UPLOAD_BUCKET_NAME
  }`;
}

// 方法 2: getPresignedPost() - 返回预签名信息
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

#### 3.2.2 AWS createPresignedPost 返回值格式

根据 AWS SDK，`createPresignedPost` 返回：

```typescript
{
  url: string;           // S3 端点 URL（如 https://bucket.s3.amazonaws.com）
  fields: {              // 必须包含的表单字段
    key: string;                              // 文件存储路径
    Policy: string;                           // Base64 编码的上传策略
    "X-Amz-Algorithm": string;                // 签名算法（如 AWS4-HMAC-SHA256）
    "X-Amz-Credential": string;               // 凭证信息
    "X-Amz-Date": string;                     // 签名日期
    "X-Amz-Signature": string;                // HMAC-SHA256 签名
    "Content-Disposition": string;            // 内容 disposition
    // ... 其他字段（如 ACL 等）
  };
}
```

#### 3.2.3 S3 存储的关键特点

对于 S3 存储，有一个**重要的巧合**：

```
getUploadUrl() 返回值 = presignedPost.url 返回值
```

这两个值都是 S3 端点 URL（如 `https://bucket.s3.amazonaws.com`）。

**为什么是巧合**：
- AWS `createPresignedPost` 设计上返回的 `url` 就是上传端点
- Outline 的 `getUploadUrl()` 也是返回 S3 上传端点
- 所以两者值相同

**实际影响**：
- 虽然 `presignedPost.url` 没有被直接使用
- 但它的值与 `uploadUrl` 相同
- 这导致在 S3 场景下，即使错误地使用 `presignedPost.url`，也能正常工作

#### 3.2.4 S3 上传完整流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           S3 上传完整流程                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 客户端请求 POST /api/attachments.create                                  │
│     ├── preset: DocumentAttachment                                           │
│     ├── documentId: doc-123                                                  │
│     ├── name: screenshot.png                                                 │
│     ├── contentType: image/png                                               │
│     └── size: 102400                                                         │
│                                    ↓                                          │
│  2. 服务端处理                                                                │
│     ├── 权限检查（auth + authorize）                                          │
│     ├── 创建 Attachment 记录                                                  │
│     │   ├── id: att-456                                                      │
│     │   ├── key: uploads/user-789/att-456/screenshot.png                   │
│     │   └── acl: private                                                      │
│     │                                                                         │
│     ├── 调用 FileStorage.getUploadUrl()                                      │
│     │   └── 返回: "https://my-bucket.s3.us-east-1.amazonaws.com"            │
│     │                                                                         │
│     └── 调用 FileStorage.getPresignedPost()                                  │
│         └── 返回: {                                                           │
│               url: "https://my-bucket.s3.us-east-1.amazonaws.com",          │
│               // ⚠️  注意：这个 url 与 uploadUrl 相同，但**未被直接使用**     │
│               fields: {                                                       │
│                 key: "uploads/user-789/att-456/screenshot.png",             │
│                 Policy: "eyJleHBpcmF0aW9uIjoiMjAyNi0wNS0wNVQxO...",       │
│                 "X-Amz-Algorithm": "AWS4-HMAC-SHA256",                      │
│                 "X-Amz-Credential": "AKIA.../us-east-1/s3/aws4_request",   │
│                 "X-Amz-Date": "20260505T120000Z",                           │
│                 "X-Amz-Signature": "a1b2c3d4e5f6...",                       │
│                 "Content-Disposition": "inline",                              │
│               }                                                               │
│             }                                                                 │
│                                    ↓                                          │
│  3. 服务端返回（关键！）                                                        │
│     {                                                                         │
│       data: {                                                                 │
│         uploadUrl: "https://my-bucket.s3.us-east-1.amazonaws.com",           │
│         // ✅  实际上传端点 - 来自 getUploadUrl()                              │
│         // ⚠️  presignedPost.url 没有被直接使用！                              │
│                                                                               │
│         form: {                                                               │
│           "Cache-Control": "max-age=31557600",                              │
│           "Content-Type": "image/png",                                        │
│           key: "uploads/user-789/att-456/screenshot.png",                   │
│           Policy: "eyJleHBpcmF0aW9uIjoiMjAyNi0wNS0wNVQxO...",               │
│           "X-Amz-Algorithm": "AWS4-HMAC-SHA256",                            │
│           "X-Amz-Credential": "AKIA.../us-east-1/s3/aws4_request",         │
│           "X-Amz-Date": "20260505T120000Z",                                 │
│           "X-Amz-Signature": "a1b2c3d4e5f6...",                             │
│           "Content-Disposition": "inline",                                    │
│           // ✅  form 包含 presignedPost.fields 的所有内容                    │
│         },                                                                     │
│         attachment: { ... }                                                   │
│       }                                                                       │
│     }                                                                         │
│                                    ↓                                          │
│  4. 客户端实际上传                                                            │
│     ├── 构建 FormData                                                         │
│     │   ├── 遍历 data.form 的所有键值对                                       │
│     │   └── 添加 file 字段                                                    │
│     │                                                                         │
│     └── 发送 POST 请求到 data.uploadUrl                                       │
│         POST https://my-bucket.s3.us-east-1.amazonaws.com                   │
│         Content-Type: multipart/form-data                                    │
│                                                                               │
│         Body:                                                                 │
│         ------WebKitFormBoundary                                              │
│         Content-Disposition: form-data; name="Cache-Control"                │
│         max-age=31557600                                                      │
│                                                                               │
│         ------WebKitFormBoundary                                              │
│         Content-Disposition: form-data; name="key"                          │
│         uploads/user-789/att-456/screenshot.png                             │
│                                                                               │
│         ------WebKitFormBoundary                                              │
│         Content-Disposition: form-data; name="Policy"                       │
│         eyJleHBpcmF0aW9uIjoiMjAyNi0wNS0wNVQxO...                            │
│                                                                               │
│         ------WebKitFormBoundary                                              │
│         Content-Disposition: form-data; name="X-Amz-Signature"             │
│         a1b2c3d4e5f6...                                                       │
│                                                                               │
│         ------WebKitFormBoundary                                              │
│         Content-Disposition: form-data; name="file"; filename="screenshot.png"│
│         Content-Type: image/png                                               │
│                                                                               │
│         [二进制文件内容]                                                        │
│         ------WebKitFormBoundary--                                            │
│                                    ↓                                          │
│  5. S3 验证签名并存储文件                                                      │
│     ├── S3 验证 Policy 和 X-Amz-Signature                                    │
│     ├── 检查上传条件（文件大小、Content-Type 等）                              │
│     └── 存储文件到指定 key                                                     │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.2.5 S3 上传关键信息对照表

| 信息项 | 来源 | 实际用途 | 说明 |
|--------|------|----------|------|
| **uploadUrl** | `getUploadUrl()` | 实际上传端点 | 客户端向此 URL 发送 POST |
| **presignedPost.url** | `getPresignedPost().url` | **未直接使用** | 与 uploadUrl 值相同（都是 S3 端点） |
| **form 中的签名字段** | `presignedPost.fields` | **核心验证信息** | S3 用这些字段验证上传合法性 |

**关键点**：
- 对于 S3，`getUploadUrl()` 和 `presignedPost.url` 返回的是同一个值
- 但 `presignedPost.url` 没有被直接使用，客户端使用的是 `uploadUrl`
- 真正重要的是 `presignedPost.fields` 中的签名信息，这些被合并到 `form` 中

---

### 3.3 本地存储：上传端点与预签名 URL 的显著差异

#### 3.3.1 本地存储相关方法实现

**核心文件**：`server/storage/files/LocalStorage.ts`

```typescript
// 方法 1: getUploadUrl() - 返回实际上传端点
public getUploadUrl() {
  return "/api/files.create";
}

// 方法 2: getPresignedPost() - 返回模拟的预签名信息
public async getPresignedPost(
  ctx: AppContext,
  key: string,
  acl: string,
  maxUploadSize: number,
  contentType = "image"
): Promise<Partial<PresignedPost>> {
  return Promise.resolve({
    url: this.getUrlForKey(key),                      // ⚠️  注意：这是 /api/files.get?key=...
    fields: {
      key,
      acl,
      maxUploadSize: String(maxUploadSize),
      contentType,
      [CSRF.fieldName]: ctx.cookies.get(CSRF.cookieName) || "",  // CSRF Token
    },
  });
}

// 方法 3: getUrlForKey() - 返回文件访问 URL（不是上传 URL！）
public getUrlForKey(key: string): string {
  return `/api/files.get?key=${key}`;
}
```

#### 3.3.2 本地存储的关键差异

对于本地存储，有一个**显著的差异**：

```
getUploadUrl() 返回值 ≠ presignedPost.url 返回值
```

具体来说：
- `getUploadUrl()` → `/api/files.create`（**上传端点**）
- `presignedPost.url` → `/api/files.get?key=...`（**访问/下载端点**）

**为什么有差异**：
- 本地存储是为了**模拟** S3 的接口
- `getPresignedPost()` 中的 `url` 字段被错误地设置为了文件访问 URL（`getUrlForKey()`）
- 而 `getUploadUrl()` 正确地返回了上传端点

**实际影响**：
- `presignedPost.url` 完全没有被使用
- 如果错误地使用 `presignedPost.url` 作为上传端点，会导致上传失败
- 客户端必须使用 `uploadUrl`

#### 3.3.3 本地存储上传完整流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        本地存储上传完整流程                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 客户端请求 POST /api/attachments.create                                  │
│                                    ↓                                          │
│  2. 服务端处理                                                                │
│     ├── 权限检查（auth + authorize）                                          │
│     ├── 创建 Attachment 记录                                                  │
│     │   ├── id: att-456                                                      │
│     │   ├── key: uploads/user-789/att-456/screenshot.png                   │
│     │   └── acl: private                                                      │
│     │                                                                         │
│     ├── 调用 FileStorage.getUploadUrl()                                      │
│     │   └── 返回: "/api/files.create"  ← ✅ 实际上传端点                      │
│     │                                                                         │
│     └── 调用 FileStorage.getPresignedPost()                                  │
│         └── 返回: {                                                           │
│               url: "/api/files.get?key=uploads/user-789/att-456/screenshot.png",│
│               // ⚠️  关键差异：这个 url 是文件访问 URL，不是上传 URL！        │
│               // ⚠️  这个 url 实际上**没有被使用**！                           │
│                                                                               │
│               fields: {                                                       │
│                 key: "uploads/user-789/att-456/screenshot.png",             │
│                 acl: "private",                                               │
│                 maxUploadSize: "10485760",                                   │
│                 contentType: "image/png",                                      │
│                 "x-csrf-token": "abc123xyz",  ← CSRF Token                  │
│               }                                                               │
│             }                                                                 │
│                                    ↓                                          │
│  3. 服务端返回（关键！）                                                        │
│     {                                                                         │
│       data: {                                                                 │
│         uploadUrl: "/api/files.create",  ← ✅ 来自 getUploadUrl()             │
│                                                                               │
│         form: {                                                               │
│           "Cache-Control": "max-age=31557600",                              │
│           "Content-Type": "image/png",                                        │
│           key: "uploads/user-789/att-456/screenshot.png",                   │
│           acl: "private",                                                      │
│           maxUploadSize: "10485760",                                          │
│           contentType: "image/png",                                            │
│           "x-csrf-token": "abc123xyz",                                         │
│           // ⚠️  presignedPost.url 没有出现在这里！                           │
│           // ✅  form 包含 presignedPost.fields 的所有内容                    │
│         },                                                                     │
│         attachment: { ... }                                                   │
│       }                                                                       │
│     }                                                                         │
│                                    ↓                                          │
│  4. 客户端实际上传                                                            │
│     ├── 构建 FormData（包含 form 中的所有字段 + file）                        │
│     │                                                                         │
│     └── 发送 POST 请求到 data.uploadUrl                                       │
│         POST /api/files.create                                                │
│         Cookie: ...（包含 CSRF Cookie）                                        │
│         Content-Type: multipart/form-data                                    │
│                                                                               │
│         Body:                                                                 │
│         ------WebKitFormBoundary                                              │
│         Content-Disposition: form-data; name="key"                          │
│         uploads/user-789/att-456/screenshot.png                             │
│                                                                               │
│         ------WebKitFormBoundary                                              │
│         Content-Disposition: form-data; name="x-csrf-token"                 │
│         abc123xyz                                                              │
│                                                                               │
│         ------WebKitFormBoundary                                              │
│         Content-Disposition: form-data; name="file"; filename="screenshot.png"│
│         Content-Type: image/png                                               │
│                                                                               │
│         [二进制文件内容]                                                        │
│         ------WebKitFormBoundary--                                            │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.3.4 本地存储上传关键信息对照表

| 信息项 | 来源 | 实际用途 | 说明 |
|--------|------|----------|------|
| **uploadUrl** | `getUploadUrl()` | 实际上传端点 | 客户端向此 URL 发送 POST |
| **presignedPost.url** | `getPresignedPost().url` | **完全未使用** | 返回的是 `/api/files.get?key=...`（访问 URL），不是上传 URL |
| **form 中的字段** | `presignedPost.fields` | **辅助验证信息** | 包含 key、acl、maxUploadSize、contentType、CSRF token |

**关键点**：
- 对于本地存储，`getUploadUrl()` 返回 `/api/files.create`，而 `presignedPost.url` 返回 `/api/files.get?key=...`
- 这两个 URL 是**完全不同**的！
- `presignedPost.url` 完全没有被使用
- `presignedPost.fields` 中的 CSRF token 是本地存储的安全验证机制
- 注意：`/api/files.create` 路由在当前代码中似乎不存在，这可能是一个待实现的功能或设计问题

---

### 3.4 S3 与本地存储上传机制差异对照表

这是本报告的核心对比表格，清晰展示两种存储后端在上传机制上的差异：

| 对比项 | S3 存储 | 本地存储 | 差异说明 |
|--------|----------|----------|----------|
| **实际上传端点** | `getUploadUrl()` → S3 端点 URL | `getUploadUrl()` → `/api/files.create` | S3 直接上传到 AWS，本地存储上传到 Outline 服务端 |
| **presignedPost.url 值** | 与 `uploadUrl` **相同**（S3 端点） | 与 `uploadUrl` **不同**（`/api/files.get?key=...`） | **这是最关键的差异** |
| **presignedPost.url 含义** | 上传端点 URL | 访问/下载端点 URL | S3 的 presignedPost.url 是上传端点，本地存储的 presignedPost.url 是访问 URL |
| **presignedPost.url 是否使用** | 未直接使用（但值与 uploadUrl 相同） | **完全未使用** | 两种存储的 presignedPost.url 都未被直接使用 |
| **核心验证机制** | AWS 签名（`Policy` + `X-Amz-Signature`） | CSRF Token | S3 使用 AWS Signature V4，本地存储使用简单的 CSRF 验证 |
| **fields 包含内容** | `key`, `Policy`, `X-Amz-*`, `Content-Disposition`, `ACL` | `key`, `acl`, `maxUploadSize`, `contentType`, `x-csrf-token` | S3 的 fields 包含复杂的签名信息，本地存储的 fields 相对简单 |
| **上传目标** | 直接上传到 S3 服务 | 上传到 Outline 服务端 | S3 是云服务，本地存储通过 Outline 服务中转 |
| **签名有效期** | 3600 秒（1 小时） | 无（CSRF token 有效期由 Cookie 决定） | S3 的签名有明确有效期，本地存储依赖 CSRF Cookie |
| **接口兼容性** | `getUploadUrl()` 和 `presignedPost.url` 可互换 | `getUploadUrl()` 和 `presignedPost.url` 不可互换 | S3 场景下即使误用 presignedPost.url 也能工作，本地存储则不行 |

---

### 3.5 上传入口职责边界总结

#### 3.5.1 核心职责划分

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        上传入口职责边界                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  attachments.create 接口的职责：                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  1. 权限检查                                                           │   │
│  │     - 验证用户身份（必需认证）                                          │   │
│  │     - 检查文档更新权限（如果有 documentId）                              │   │
│  │     - 检查团队附件创建权限                                               │   │
│  │                                                                         │   │
│  │  2. 创建附件元数据                                                       │   │
│  │     - 生成 UUID 作为附件 ID                                              │   │
│  │     - 生成存储路径 key                                                   │   │
│  │     - 确定 ACL（访问控制列表）                                           │   │
│  │     - 插入数据库记录                                                      │   │
│  │                                                                         │   │
│  │  3. 生成上传凭证（核心！）                                                │   │
│  │     - ✅ uploadUrl: 实际上传端点（来自 getUploadUrl()）                  │   │
│  │     - ✅ form: 必须包含的表单字段（含 presignedPost.fields）             │   │
│  │     - ❌ presignedPost.url: 辅助信息，未直接使用                         │   │
│  │                                                                         │   │
│  │  4. 返回上传信息                                                         │   │
│  │     - uploadUrl                                                         │   │
│  │     - form                                                              │   │
│  │     - attachment（含 redirectUrl 用于后续访问）                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ⚠️  重要澄清：presignedPost 返回值的实际用途                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  presignedPost = {                                                      │   │
│  │    url: "...",      ← ⚠️  这个字段**没有被直接使用**！                  │   │
│  │    fields: { ... }  ← ✅  这些字段**被合并到 form 中实际使用**          │   │
│  │  }                                                                      │   │
│  │                                                                         │   │
│  │  客户端实际使用的是：                                                    │   │
│  │  - ✅ uploadUrl（来自 FileStorage.getUploadUrl()）                       │   │
│  │  - ✅ form（包含 presignedPost.fields）                                  │   │
│  │  - ❌ presignedPost.url（未直接使用）                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.5.2 上传信息字段分类（最终版）

| 字段类别 | 字段名称 | 来源 | 是否必需 | 实际用途 |
|----------|----------|------|----------|----------|
| **实际上传端点** | `uploadUrl` | `getUploadUrl()` | ✅ 必需 | 客户端 POST 的目标 URL |
| **签名/验证字段** | `form`（含 `presignedPost.fields`） | 服务端组装 | ✅ 必需 | S3 或服务端用这些验证上传 |
| **服务端添加字段** | `Cache-Control`, `Content-Type` | 服务端在返回前添加 | ✅ 必需 | 控制缓存和内容类型 |
| **辅助信息（未使用）** | `presignedPost.url` | `getPresignedPost().url` | ❌ 未使用 | 仅为接口兼容，实际不用 |

#### 3.5.3 为什么 presignedPost.url 未被使用？

这是一个有趣的设计决策，有以下几种可能的原因：

1. **接口兼容考虑**：
   - Outline 使用 `BaseStorage` 抽象层统一 S3 和本地存储接口
   - `getPresignedPost()` 是 S3 风格的接口
   - `presignedPost.url` 是为了保持与 AWS SDK 接口的兼容性

2. **实际使用更清晰**：
   - `uploadUrl` 明确表示"上传端点"
   - `form` 明确表示"表单字段"
   - 这种分离比使用 `presignedPost.url` 更清晰

3. **本地存储的特殊性**：
   - 对于本地存储，`presignedPost.url` 甚至不是上传端点
   - 使用独立的 `uploadUrl` 避免了混淆

4. **历史原因**：
   - 可能早期版本使用过 `presignedPost.url`
   - 后来改为使用 `uploadUrl`，但保留了 `presignedPost.url` 以保持兼容性

---

### 3.6 标准上传入口：attachments.create（权限检查）

#### 3.6.1 中间件配置

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

#### 3.6.2 权限检查策略（严格）

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

#### 3.6.3 权限检查矩阵

| Preset 类型 | documentId | 权限检查 | 说明 |
|-------------|------------|----------|------|
| `Avatar` | 忽略 | 仅验证 content-type | 任何人可上传头像 |
| `DocumentAttachment` | 有值 | `update` 文档权限 + `createAttachment` 团队权限 | 必须能编辑文档 |
| `DocumentAttachment` | 无值 | `createAttachment` 团队权限 | 未关联文档的附件 |
| `Emoji` | 忽略 | 验证 content-type + `createAttachment` | 团队成员可上传表情 |
| `Import`/其他 | 忽略 | `createAttachment` 团队权限 | 团队成员可上传 |

#### 3.6.4 ACL 设置

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

### 3.7 从 URL 创建入口：attachments.createFromUrl

```typescript
router.post(
  "attachments.createFromUrl",
  rateLimiter(RateLimiterStrategy.TwentyFivePerMinute),
  auth(),
  validate(T.AttachmentsCreateFromUrlSchema),
  async (ctx: APIContext<T.AttachmentsCreateFromUrlReq>) => {
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

### 3.8 程序化创建：attachmentCreator

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
    {
      lastAccessedAt: new Date(),
    },
    {
      silent: true,
    }
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

让我拆解权限检查的条件：

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

**重要声明**：这两个维度是**完全独立**的，**存储位置维度不影响访问权限**。

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
┌─────────────────────────────────────────────────────────────────────────────┐
│                              安全边界                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Team A                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                     │
│  │  Document 1 │    │  Document 2 │    │  Attachment │                     │
│  │  (私有)     │    │  (公开)     │    │  (私有)     │                     │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                     │
│         │                  │                  │                              │
│         └──────────────────┼──────────────────┘                              │
│                            │                                                 │
│                     ┌──────┴──────┐                                          │
│                     │  User A     │                                          │
│                     │  (Team A)   │ ◄── 可以访问所有附件                     │
│                     └─────────────┘                                          │
│                                                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Team B                                                                      │
│  ┌─────────────┐                                                             │
│  │  User B     │ ◄── 无法访问 Team A 的私有附件                              │
│  │  (Team B)   │     (403 AuthorizationError)                               │
│  └─────────────┘                                                             │
│                                                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  未登录用户                                                                   │
│  ┌─────────────┐                                                             │
│  │             │ ◄── 无法访问私有附件                                          │
│  │             │     但可以访问 public-read 附件                               │
│  └─────────────┘                                                             │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
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
   - 生成上传信息（uploadUrl + form）
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
    FileHelper.is