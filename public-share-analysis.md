# Outline 公开共享文档与登录用户视图差异分析报告

## 1. 概述

本文档深入分析 Outline 知识管理系统中，公开共享文档视图与登录用户文档视图之间的核心差异。分析涵盖三个主要维度：
- 公开分享链路的实现机制
- 权限裁剪策略
- 服务端差异化渲染

---

## 2. 公开分享链路实现

### 2.1 路由架构

#### 服务端路由配置
在 `server/routes/index.ts` 中，公开共享文档的路由采用 `/s/:shareId` 格式：

```typescript
// 核心共享路由
router.get("/s/:shareId.:format", shareDomains(), renderShare);
router.get("/s/:shareId", shareDomains(), renderShare);
router.get("/s/:shareId/doc/:documentSlug.:format", shareDomains(), renderShare);
router.get("/s/:shareId/doc/:documentSlug", shareDomains(), renderShare);
router.get("/s/:shareId/*", shareDomains(), renderShare);
```

关键特性：
- 支持 URL 自定义 slug（通过 `urlId` 字段）
- 支持自定义域名（通过 `shareDomains()` 中间件）
- 支持 Markdown 格式输出（`.md` 后缀）
- 支持文档子路径访问

#### 旧路由兼容
系统还保留了 `/share/:shareId` 格式的旧路由，并进行 301 重定向：

```typescript
router.use(
  ["/share/:shareId", "/share/:shareId/doc/:documentSlug", "/share/:shareId/*"],
  (ctx) => {
    const redirectPath = ctx.path.replace(/^\/share/, "/s");
    ctx.redirect(redirectPath + ctx.request.URL.search);
    ctx.status = 301;
  }
);
```

### 2.2 核心加载流程

#### `loadPublicShare` 函数
位于 `server/commands/shareLoader.ts`，是公开文档加载的核心入口：

**主要职责**：
1. **Share 记录验证**：检查 shareId 是否有效，是否已发布，是否已撤销
2. **团队和集合权限检查**：验证团队是否启用了分享功能，集合是否允许分享
3. **子文档访问控制**：当访问子文档时，验证其是否在共享树范围内
4. **共享树构建**：构建文档导航树用于侧边栏展示

**关键代码逻辑**：

```typescript
// 验证 Share 状态
const where: WhereOptions<Share> = {
  revokedAt: { [Op.is]: null },
  published: true,
};

// 检查团队和集合是否允许分享
if (
  !share.team.sharing ||
  (!isDraftWithoutCollection && !associatedCollection?.sharing)
) {
  throw AuthorizationError();
}

// 子文档访问验证
if (documentId && documentId !== share.documentId) {
  document = await Document.findByPk(documentId, { rejectOnEmpty: true });
  
  let isDocumentAccessible = share.documentId === document.id;
  
  if (share.includeChildDocuments) {
    const allIdsInSharedTree = getAllIdsInSharedTree(sharedTree);
    isDocumentAccessible = allIdsInSharedTree.includes(document.id);
  }
  
  if (!isDocumentAccessible) {
    throw AuthorizationError();
  }
}
```

#### `renderShare` 函数
位于 `server/routes/app.ts`，负责服务端渲染共享页面：

**主要职责**：
1. **URL 规范化**：如果使用 UUID 访问但存在自定义 slug，重定向到规范 URL
2. **访问统计**：更新分享的访问次数和最后访问时间
3. **多格式支持**：支持 HTML 和 Markdown 两种输出格式
4. **SSR 内容注入**：将文档标题、描述、内容注入到 HTML 中

**Markdown 输出支持**：
```typescript
// 支持 Accept: text/markdown 或 .md 后缀
const prefersMarkdown =
  ctx.params.format === "md" ||
  (acceptHeader.includes("text/markdown") &&
   ctx.accepts("text/markdown", "text/html") === "text/markdown");

if (prefersMarkdown && (document || collection)) {
  let markdown = await DocumentHelper.toMarkdown(document || collection!, {
    includeTitle: true,
    signedUrls: 86400, // 24小时有效期
    teamId: team?.id,
  });
  // ... 追加子文档列表
  ctx.type = "text/markdown";
  ctx.body = markdown;
  return;
}
```

### 2.3 API 接口层

#### `shares.info` 接口
位于 `server/routes/api/shares/shares.ts`，是前端获取共享数据的核心接口：

**关键特性**：
1. **可选认证**：使用 `auth({ optional: true })` 中间件，支持匿名和登录用户两种场景
2. **差异化处理**：
   - 匿名用户：直接使用 `loadPublicShare` 的结果
   - 登录用户：重新加载集合和文档，包含用户的成员关系信息
3. **权限感知序列化**：根据用户权限决定 `isPublic` 标志

**核心代码**：
```typescript
router.post(
  "shares.info",
  auth({ optional: true }),
  validate(T.SharesInfoSchema),
  async (ctx: APIContext<T.SharesInfoReq>) => {
    // 仅公开链接加载会发送 "id"
    if (id) {
      let { share, sharedTree, collection, document } = await loadPublicShare({
        id, collectionId, documentId, teamId: teamFromCtx?.id,
      });

      // 如果用户已认证，使用成员关系范围重新加载
      if (user) {
        collection = collection
          ? await Collection.findByPk(collection.id, { userId: user.id })
          : null;
        document = document
          ? await Document.findByPk(document.id, { userId: user.id })
          : null;
      }

      // 根据用户权限决定 isPublic 标志
      const [serializedCollection, serializedDocument] = await Promise.all([
        collection
          ? await presentCollection(ctx, collection, {
              isPublic: cannot(user, "read", collection),
              shareId: share.id,
              includeUpdatedAt: share.showLastUpdated,
            })
          : null,
        document
          ? await presentDocument(ctx, document, {
              isPublic: cannot(user, "read", document),
              shareId: share.id,
              includeUpdatedAt: share.showLastUpdated,
            })
          : null,
      ]);
      // ...
    }
  }
);
```

---

## 3. 权限裁剪机制

### 3.1 数据序列化层

#### 文档序列化 (`presentDocument`)
位于 `server/presenters/document.ts`，通过 `isPublic` 参数控制字段输出：

**公开文档隐藏的字段**：
| 字段 | 说明 | 安全考虑 |
|------|------|----------|
| `createdBy` | 创建者信息 | 保护用户隐私 |
| `updatedBy` | 最后更新者信息 | 保护用户隐私 |
| `collaboratorIds` | 协作者 ID 列表 | 保护用户隐私 |
| `collectionId` | 所属集合 ID | 隐藏内部结构 |
| `parentDocumentId` | 父文档 ID | 隐藏内部结构 |
| `lastViewedAt` | 最后查看时间 | 用户特定数据 |
| `tasks` | 任务状态 | 内部协作数据 |
| `templateId` | 模板 ID | 内部数据 |
| `insightsEnabled` | 洞察启用状态 | 内部设置 |
| `popularityScore` | 流行度分数 | 内部统计 |
| `sourceMetadata` | 源元数据 | 可能包含敏感信息 |

**可选隐藏字段**：
- `updatedAt`：根据 `showLastUpdated` 分享设置决定是否显示

**核心代码**：
```typescript
const res: Record<string, unknown> = {
  id: document.id,
  url: document.path,
  // ... 基础字段始终显示
  createdAt: document.createdAt,
  createdBy: undefined,  // 公开时隐藏
  updatedAt: document.updatedAt,  // 可能被删除
  updatedBy: undefined,  // 公开时隐藏
  collectionId: undefined,  // 公开时隐藏
  parentDocumentId: undefined,  // 公开时隐藏
  // ...
};

// 根据设置删除 updatedAt
if (options.isPublic && !options.includeUpdatedAt) {
  delete res.updatedAt;
}

// 非公开模式添加额外字段
if (!options.isPublic) {
  res.tasks = document.tasks;
  res.isCollectionDeleted = await document.isCollectionDeleted();
  res.collectionId = document.collectionId;
  res.parentDocumentId = document.parentDocumentId;
  res.createdBy = presentUser(document.createdBy);
  res.updatedBy = presentUser(document.updatedBy);
  res.collaboratorIds = document.collaboratorIds;
  // ... 更多字段
}
```

#### 集合序列化 (`presentCollection`)
位于 `server/presenters/collection.ts`，类似的权限裁剪逻辑：

**公开集合隐藏的字段**：
| 字段 | 说明 |
|------|------|
| `index` | 文档排序索引 |
| `sharing` | 分享设置 |
| `commenting` | 评论设置 |
| `templateManagement` | 模板管理设置 |
| `permission` | 当前用户权限 |
| `deletedAt` | 删除时间 |
| `archivedAt` | 归档时间 |
| `archivedBy` | 归档者 |
| `sourceMetadata` | 源元数据 |

### 3.2 文档内容处理

#### Prosemirror 内容转换
在 `DocumentHelper.toJSON` 方法中，对文档内容进行额外处理：

**公开文档的内容处理**：
1. **移除评论标记**：`removeMarks: ["comment"]`
2. **重写内部链接**：将 `/doc/xxx` 链接重写为 `/s/:shareId/doc/xxx` 格式
3. **签名附件 URL**：为附件生成有时效性的签名 URL

**核心代码**：
```typescript
// 在 shares.info 接口中调用
const data = await DocumentHelper.toJSON(
  document,
  options.isPublic
    ? {
        signedUrls: Hour.seconds,  // 1小时有效期
        teamId: document.teamId,
        removeMarks: ["comment"],  // 移除评论
        internalUrlBase: `/s/${options.shareId}`,  // 重写链接
      }
    : undefined
);
```

#### 链接重写实现
位于 `server/models/helpers/ProsemirrorHelper.ts` 中的 `replaceInternalUrls` 方法：

```typescript
export function replaceInternalUrls(
  data: ProsemirrorData,
  internalUrlBase: string
): ProsemirrorData {
  // 递归遍历 Prosemirror 节点
  // 将所有 /doc/xxx 格式的链接替换为 /s/:shareId/doc/xxx
  // 确保公开文档中的内部链接仍然有效
}
```

### 3.3 权限决定逻辑

#### `isPublic` 标志的计算
在 `shares.info` 接口中，`isPublic` 标志通过以下逻辑决定：

```typescript
isPublic: cannot(user, "read", collection/document)
```

这意味着：
- **匿名用户**：`user` 为 `undefined`，`cannot()` 返回 `true` → `isPublic = true`
- **登录但无权限用户**：`cannot()` 返回 `true` → `isPublic = true`
- **登录且有权限用户**：`cannot()` 返回 `false` → `isPublic = false`

#### 权限检查策略
从 `server/policies/document.ts` 可以看到，文档读取权限需要：
1. 用户属于同一团队
2. 满足以下任一条件：
   - 有文档成员关系（Read/ReadWrite/Admin）
   - 是草稿且为创建者
   - 对集合有读取权限

**关键代码**：
```typescript
allow(User, "read", Document, (actor, document) =>
  and(
    isTeamModel(actor, document),
    or(
      includesMembership(document, [
        DocumentPermission.Read,
        DocumentPermission.ReadWrite,
        DocumentPermission.Admin,
      ]),
      and(!!document?.isDraft, actor.id === document?.createdById),
      can(actor, "readDocument", document?.collection)
    )
  )
);
```

---

## 4. 服务端差异化渲染

### 4.1 SSR 渲染流程

#### `renderApp` 函数
位于 `server/routes/app.ts`，是通用的应用渲染函数：

**主要功能**：
1. **HTML 模板注入**：读取 `index.html` 模板
2. **环境变量注入**：将 `window.env` 注入到页面
3. **元数据设置**：设置 title、description、canonical URL 等
4. **资源加载**：根据环境决定使用 Vite 开发服务器还是生产构建

**共享页面特定参数**：
```typescript
export const renderApp = async (
  ctx: Context,
  next: Next,
  options: {
    title?: string;
    description?: string;
    content?: string;
    canonical?: string;
    isShare?: boolean;
    rootShareId?: string;
    allowIndexing?: boolean;
    analytics?: Integration<IntegrationType.Analytics>[];
  } = {}
) => {
  // 共享页面添加 sitemap 引用
  if (options.isShare) {
    headTags += `
    <link rel="sitemap" type="application/xml" href="/api/shares.sitemap?id=${escape(options.rootShareId || shareId)}">
    `;
  } else {
    // 非共享页面添加 PWA 相关标签
    headTags += prefetchTags;
    headTags += `
    <link rel="manifest" href="/static/manifest.webmanifest" />
    // ...
    `;
  }
};
```

#### `renderShare` 函数
共享页面的专用渲染函数，在 `renderApp` 基础上增加了：

1. **文档元数据提取**：从 `loadPublicShare` 结果中提取标题、描述
2. **内容预渲染**：使用 `DocumentHelper.toHTML` 生成 HTML 内容用于 SEO
3. **规范 URL 处理**：处理自定义域名和 URL slug
4. **访问统计**：更新分享的访问计数

**SEO 优化**：
```typescript
const title = document
  ? document.title
  : collection
    ? collection.name
    : publicBranding && team?.name
      ? team.name
      : undefined;

const content =
  document || collection
    ? await DocumentHelper.toHTML(document || collection!, {
        includeStyles: false,
        includeHead: false,
        includeTitle: true,
        signedUrls: true,
      })
    : undefined;

const canonicalUrl =
  share && share.canonicalUrl !== ctx.request.origin + ctx.request.url
    ? `${share.canonicalUrl}${
        documentSlug && document
          ? document.path
          : collectionSlug && collection
            ? collection.path
            : ""
      }`
    : undefined;
```

### 4.2 前端路由差异

#### 共享页面入口 (`SharedScene`)
位于 `app/scenes/Shared/index.tsx`，与普通文档页面的关键差异：

**主要差异点**：
1. **独立的路由上下文**：使用 `ShareContext.Provider` 提供共享信息
2. **不同的侧边栏组件**：使用 `Sidebar` from `"~/components/Sidebar/Shared"`
3. **不同的命令栏**：使用 `SharedCommandBar`
4. **可选登录状态**：使用 `useCurrentUser({ rejectOnEmpty: false })`

**ShareContext 提供的值**：
```typescript
<ShareContext.Provider
  value={{
    shareId,
    sharedTree: share.tree,
    allowSubscriptions: share.allowSubscriptions,
    showLastUpdated: share.showLastUpdated,
  }}
>
```

#### 共享文档视图 (`SharedDocument`)
位于 `app/scenes/Shared/Document.tsx`：

**关键特性**：
1. **只读模式**：`readOnly` 属性强制设置为 `true`
2. **空权限对象**：`abilities = useMemo(() => ({}), [])`
3. **可选品牌展示**：非自定义域名且无用户时显示 Outline 品牌
4. **目录位置感知**：使用团队设置的目录位置

**核心代码**：
```typescript
<DocumentComponent
  abilities={abilities}  // 空权限对象
  document={document}
  shareId={shareId}
  tocPosition={tocPosition}
  readOnly  // 强制只读
/>
{showBranding ? (
  <Branding href="//www.getoutline.com?ref=sharelink" />
) : null}
```

### 4.3 嵌入支持

#### iframe 嵌入控制
在 `renderShare` 中，根据团队偏好决定是否允许嵌入：

```typescript
// 允许共享页面嵌入到其他网站的 iframe 中，除非团队偏好阻止
const preventEmbedding =
  team?.getPreference(TeamPreference.PreventDocumentEmbedding) ?? false;
if (!preventEmbedding) {
  ctx.remove("X-Frame-Options");
}
```

#### 嵌入路由
系统还提供了专门的嵌入路由：
```typescript
router.get("/embeds/gitlab", renderEmbed);
router.get("/embeds/github", renderEmbed);
router.get("/embeds/dropbox", renderEmbed);
router.get("/embeds/pinterest", renderEmbed);
```

### 4.4 自定义域名支持

#### `shareDomains` 中间件
处理自定义域名的核心逻辑：

**主要功能**：
1. **域名解析**：从请求域名中识别是否为自定义域名
2. **Root Share 查找**：根据域名查找对应的 `rootShare`
3. **上下文注入**：将 `rootShare` 注入到 `ctx.state.rootShare`
4. **路径重写**：对于自定义域名，根路径 `/` 映射到共享页面

**路由集成**：
```typescript
// 自定义域名的共享路由
router.use(shareDomains());

router.get("/doc/:documentSlug", async (ctx, next) => {
  if (ctx.state?.rootShare) {
    return renderShare(ctx, next);
  }
  return next();
});

router.get("/sitemap.xml", async (ctx) => {
  if (ctx.state?.rootShare) {
    ctx.redirect(`/api/shares.sitemap?id=${ctx.state?.rootShare.id}`);
  } else {
    ctx.status = 404;
  }
});

// 自定义域名的 catch-all 路由
router.get("*", async (ctx, next) => {
  if (ctx.state?.rootShare) {
    // 自定义域名只允许根路径，其他路径返回 404
    // 有效路径如 /doc/:documentSlug 和 /sitemap.xml 已在上面处理
    if (ctx.path !== "/") {
      ctx.status = 404;
      return;
    }
    return renderShare(ctx, next);
  }
  // ... 普通域名处理
});
```

---

## 5. 三类身份下的可见差异深度分析

### 5.1 三类身份定义

在同一分享链接下，系统根据用户身份状态进行差异化处理：

| 身份类型 | 定义 | `user` 对象 | `cannot(user, "read", document)` |
|---------|------|-------------|----------------------------------|
| **匿名访问** | 未登录用户 | `undefined` | `true`（`isPublic = true`） |
| **已登录但无权限** | 已登录，但对文档/集合无读取权限 | 存在 | `true`（`isPublic = true`） |
| **已登录且有权限** | 已登录，且对文档/集合有读取权限 | 存在 | `false`（`isPublic = false`） |

### 5.2 核心决策逻辑

#### `isPublic` 标志的计算链路

在 `shares.info` 接口（`server/routes/api/shares/shares.ts:87,94`）中，`isPublic` 标志通过以下逻辑决定：

```typescript
// 关键点 1：登录用户重新加载带成员关系的集合和文档
if (user) {
  collection = collection
    ? await Collection.findByPk(collection.id, { userId: user.id })
    : null;
  document = document
    ? await Document.findByPk(document.id, { userId: user.id })
    : null;
}

// 关键点 2：根据用户权限决定 isPublic 标志
isPublic: cannot(user, "read", collection/document)
```

#### `cannot` 函数的行为分析

位于 `server/policies/cancan.ts:166`：

```typescript
public cannot = (
  performer: Model,
  action: string,
  target: Model | null | undefined,
  options = {}
) => !this.can(performer, action, target, options);
```

**三类身份的具体行为**：

1. **匿名访问（`user = undefined`）**：
   - `cannot(undefined, "read", document)` → `!can(undefined, "read", document)`
   - `can` 函数中 `performer instanceof model` 对于 `undefined` 返回 `false`
   - 结果：`can(undefined, ...) = false` → `cannot(undefined, ...) = true`
   - 最终：`isPublic = true`

2. **已登录但无权限**：
   - 用户已登录，但 `can(user, "read", document)` 返回 `false`
   - 可能原因：不在同一团队、没有成员关系、不是集合成员
   - 结果：`cannot(user, ...) = true` → `isPublic = true`

3. **已登录且有权限**：
   - 用户已登录，且 `can(user, "read", document)` 返回 `true`
   - 满足条件：在同一团队、有成员关系、或是集合成员
   - 结果：`cannot(user, ...) = false` → `isPublic = false`

### 5.3 数据字段差异对比

| 字段 | 匿名访问 | 已登录但无权限 | 已登录且有权限 | 原因说明 |
|------|---------|---------------|---------------|----------|
| `createdBy` | ❌ 隐藏 | ❌ 隐藏 | ✅ 完整显示 | 保护用户隐私，仅有权限用户可见 |
| `updatedBy` | ❌ 隐藏 | ❌ 隐藏 | ✅ 完整显示 | 保护用户隐私，仅有权限用户可见 |
| `collaboratorIds` | ❌ 隐藏 | ❌ 隐藏 | ✅ 完整显示 | 保护协作者隐私 |
| `collectionId` | ❌ 隐藏 | ❌ 隐藏 | ✅ 完整显示 | 隐藏内部结构信息 |
| `parentDocumentId` | ❌ 隐藏 | ❌ 隐藏 | ✅ 完整显示 | 隐藏内部结构信息 |
| `updatedAt` | ⚠️ 可选隐藏 | ⚠️ 可选隐藏 | ✅ 完整显示 | 取决于 `showLastUpdated` 分享设置 |
| `tasks` | ❌ 隐藏 | ❌ 隐藏 | ✅ 完整显示 | 内部协作数据 |
| `templateId` | ❌ 隐藏 | ❌ 隐藏 | ✅ 完整显示 | 内部模板信息 |
| `insightsEnabled` | ❌ 隐藏 | ❌ 隐藏 | ✅ 完整显示 | 内部设置 |
| `popularityScore` | ❌ 隐藏 | ❌ 隐藏 | ✅ 完整显示 | 内部统计 |
| `sourceMetadata` | ❌ 隐藏 | ❌ 隐藏 | ✅ 完整显示 | 可能包含敏感信息 |
| **文档内容** | 移除评论、重写链接 | 移除评论、重写链接 | 原始内容 | 保护评论隐私，确保链接有效 |
| **附件 URL** | 有时效性签名 | 有时效性签名 | 内部 URL | 防止未授权访问附件 |

**关键发现**：匿名访问和已登录但无权限的用户在数据字段上的可见性**完全相同**，都处于 `isPublic = true` 的脱敏状态。只有已登录且有读取权限的用户才能看到完整数据。

### 5.4 页面能力差异对比

| 能力 | 匿名访问 | 已登录但无权限 | 已登录且有权限 | 原因说明 |
|------|---------|---------------|---------------|----------|
| **编辑能力** | ❌ 只读 | ❌ 只读 | ✅ 根据权限 | 共享页面强制 `readOnly: true` |
| **命令栏** | `SharedCommandBar` | `SharedCommandBar` | 完整 `CommandBar` | 受限功能集 vs 完整功能 |
| **侧边栏** | `SharedSidebar` | `SharedSidebar` | 完整侧边栏 | 仅显示共享树 vs 完整导航 |
| **评论功能** | ❌ 禁用 | ❌ 禁用 | ✅ 根据权限 | 协作功能仅对有权限用户开放 |
| **任务功能** | ❌ 禁用 | ❌ 禁用 | ✅ 根据权限 | 协作功能仅对有权限用户开放 |
| **提及功能** | ❌ 禁用 | ❌ 禁用 | ✅ 根据权限 | 协作功能仅对有权限用户开放 |
| **品牌展示** | ⚠️ 可能显示 | ⚠️ 可能显示 | ❌ 不显示 | 非自定义域名且无用户时显示 |
| **嵌入支持** | ✅ 可选 | ✅ 可选 | 不适用 | iframe 嵌入控制 |
| **订阅功能** | ✅ 可选 | ✅ 可选 | 不适用 | 基于 `allowSubscriptions` 设置 |

**代码依据**（`app/scenes/Shared/Document.tsx`）：

```typescript
function SharedDocument({ document }: Props) {
  // ...
  const abilities = useMemo(() => ({}), []);  // 空权限对象
  const showBranding = !isCustomDomain && !user;  // 品牌展示条件
  
  return (
    <>
      <DocumentComponent
        abilities={abilities}    // 空权限
        document={document}
        shareId={shareId}
        tocPosition={tocPosition}
        readOnly                // 强制只读
      />
      {showBranding ? (
        <Branding href="//www.getoutline.com?ref=sharelink" />
      ) : null}
    </>
  );
}
```

### 5.5 为什么同一分享链接下会有不同表现？

#### 设计动机

1. **无缝身份切换体验**：
   - 用户在匿名访问共享文档后登录，系统应自动识别其权限
   - 如果用户原本就有该文档的访问权限，应切换到完整功能视图
   - 不需要用户重新访问原始链接

2. **权限优先级**：
   - 分享链接提供的是"最低权限访问"
   - 用户自身的权限（团队成员、集合成员、文档成员）具有更高优先级
   - 当用户同时拥有分享链接和直接权限时，使用直接权限

3. **数据一致性**：
   - 登录用户应看到与在团队内部访问相同的数据
   - 包括用户信息、协作数据、内部结构等
   - 避免因访问方式不同导致的数据不一致

#### 技术实现要点

1. **双阶段加载**：
   - 第一阶段：使用 `loadPublicShare` 加载基本共享信息（不依赖用户）
   - 第二阶段：如果用户已登录，使用 `userId` 重新加载集合和文档
   - 重新加载的数据包含用户的成员关系信息

2. **权限感知序列化**：
   - `presentDocument` 和 `presentCollection` 根据 `isPublic` 参数决定输出字段
   - `isPublic` 由 `cannot(user, "read", document)` 动态计算
   - 确保数据输出与用户实际权限匹配

3. **前端组件自适应**：
   - 虽然共享页面使用 `SharedScene` 入口
   - 但数据字段的完整程度由后端序列化决定
   - 前端根据返回的数据字段调整 UI 展示

---

## 6. 边界场景处理链路分析

### 6.1 分享撤销处理链路

#### 场景描述
分享被创建者或管理员撤销后，后续访问应如何处理？

#### 处理链路

**1. 撤销操作执行**（`server/models/Share.ts:255`）：

```typescript
revoke(ctx: APIContext) {
  const { user } = ctx.state.auth;
  this.revokedAt = new Date();    // 设置撤销时间戳
  this.revokedById = user.id;     // 记录撤销者
  return this.saveWithCtx(ctx, undefined, { name: "revoke" });
}
```

**关键点**：
- 撤销是**软删除**，通过设置 `revokedAt` 时间戳实现
- 记录撤销者 ID 用于审计

**2. 访问时的查询过滤**（`server/commands/shareLoader.ts:36`）：

```typescript
const where: WhereOptions<Share> = {
  revokedAt: {
    [Op.is]: null,    // 只查询未撤销的分享
  },
  published: true,
};
```

**3. 找不到时的错误处理**（`server/commands/shareLoader.ts:71`）：

```typescript
if (
  !share ||                        // 包括已撤销的情况
  !!share.team.suspendedAt ||
  !!share.collection?.archivedAt ||
  !!share.document?.archivedAt
) {
  throw NotFoundError();           // 抛出 404 错误
}
```

**4. SSR 层的异常捕获**（`server/routes/app.ts:256`）：

```typescript
try {
  const result = await loadPublicShare({...});
  // ...
} catch (_err) {
  // If the share or document does not exist, return a 404.
  ctx.status = 404;
}
```

#### 完整处理流程图

```
用户访问分享链接
        ↓
┌─────────────────────────────────────┐
│  shareDomains() 中间件识别域名       │
│  （自定义域名查找 rootShare）        │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│  renderShare() 处理函数              │
│  调用 loadPublicShare()             │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│  loadPublicShare() 查询条件：        │
│  WHERE revokedAt IS NULL            │
│    AND published = true             │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│  查询结果为空？                      │
│  （已撤销的分享不会被匹配）          │
└─────────────────────────────────────┘
        ↓
   ┌────┴────┐
   │         │
  是        否
   │         │
   ↓         ↓
 throw    正常返回
NotFoundError  数据
   │
   ↓
 renderShare
  捕获异常
   │
   ↓
设置 status = 404
   │
   ↓
 返回 404 页面
```

#### 设计决策：为什么返回 404 而不是 403？

**安全考虑**：
- 返回 403（Forbidden）会泄露"该分享链接确实存在，但你没有权限访问"的信息
- 返回 404（Not Found）不会泄露任何信息，攻击者无法判断：
  - 链接是否真实存在
  - 链接是否已被撤销
  - 链接是否从未发布

**测试验证**（`server/commands/shareLoader.test.ts:323`）：

```typescript
describe("inactive share when requested with id", () => {
  it("should throw error when share is not published", async () => {
    const share = await buildShare({
      published: false,
    });
    await expect(loadPublicShare({ id: share.id })).rejects.toThrow();
  });
  // ... 更多测试用例
});
```

### 6.2 子文档越权访问处理链路

#### 场景描述
用户尝试通过修改 URL 中的 `documentSlug` 访问分享范围外的文档。

#### 处理链路

**1. 路由参数解析**（`server/routes/index.ts`）：

```typescript
router.get("/s/:shareId/doc/:documentSlug", shareDomains(), renderShare);
```

**2. `loadPublicShare` 中的访问检查**（`server/commands/shareLoader.ts:109`）：

```typescript
if (documentId && documentId !== share.documentId) {
  // 访问的不是根文档，需要额外检查
  
  // 步骤 1：获取文档（如果不存在直接抛出 404）
  document = await Document.findByPk(documentId, {
    rejectOnEmpty: true,
  });

  // 步骤 2：初始化可访问性判断
  let isDocumentAccessible = share.documentId === document.id;

  // 步骤 3：如果开启了子文档共享，检查是否在共享树中
  if (share.includeChildDocuments) {
    const allIdsInSharedTree = getAllIdsInSharedTree(sharedTree);
    isDocumentAccessible = allIdsInSharedTree.includes(document.id);
  }

  // 步骤 4：不可访问则抛出授权错误
  if (!isDocumentAccessible) {
    throw AuthorizationError();
  }
}
```

**3. `getAllIdsInSharedTree` 函数**（`server/commands/shareLoader.ts:245`）：

```typescript
export function getAllIdsInSharedTree(
  sharedTree: NavigationNode | null
): string[] {
  if (!sharedTree) {
    return [];
  }

  const ids = [sharedTree.id];
  for (const child of sharedTree.children) {
    ids.push(...getAllIdsInSharedTree(child));  // 递归获取所有子节点
  }
  return ids;
}
```

#### 完整处理流程图

```
用户访问 /s/:shareId/doc/:targetDocSlug
        ↓
┌─────────────────────────────────────┐
│  解析参数：                           │
│  shareId = URL 中的分享 ID           │
│  documentId = 从 slug 解析的文档 ID  │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│  loadPublicShare() 执行：           │
│  1. 查询 Share 记录                  │
│  2. 检查团队/集合分享设置            │
│  3. 构建 sharedTree（共享文档树）    │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│  documentId !== share.documentId？  │
│  （访问的不是根文档？）              │
└─────────────────────────────────────┘
        ↓
   ┌────┴────┐
   │         │
  是        否
   │         │
   ↓         ↓
 需要     直接返回
 检查      根文档
   │
   ↓
┌─────────────────────────────────────┐
│  Document.findByPk(documentId)      │
│  如果文档不存在 → 404               │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│  share.includeChildDocuments = ?    │
│  （是否开启了子文档共享？）          │
└─────────────────────────────────────┘
        ↓
   ┌────┴────┐
   │         │
 true     false
   │         │
   ↓         ↓
 检查      仅允许
是否在     访问
共享树中    根文档
   │         │
   ↓         ↓
 在树中？   documentId
   │       === share.documentId？
   ↓            │
┌──┴──┐       ┌──┴──┐
│     │       │     │
是    否      是    否
│     │       │     │
↓     ↓       ↓     ↓
允许  抛出    允许  抛出
访问 AuthorizationError
```

#### 测试用例验证（`server/commands/shareLoader.test.ts`）

**用例 1：includeChildDocuments = true 时访问越权文档**：
```typescript
it("should throw error when the requested document is not part of the share (includeChildDocuments = true)", async () => {
  // 创建分享文档 A 和非分享文档 B
  // 尝试通过分享链接访问文档 B
  await expect(
    loadPublicShare({ id: share.id, documentId: anotherDocument.id })
  ).rejects.toThrow();  // 期望抛出错误
});
```

**用例 2：includeChildDocuments = false 时访问子文档**：
```typescript
it("should throw error when the child document is requested for a share with includeChildDocuments = false", async () => {
  // 创建分享文档 A 和其子文档 B
  // 设置 includeChildDocuments = false
  // 尝试访问子文档 B
  await expect(
    loadPublicShare({ id: share.id, documentId: childDocument.id })
  ).rejects.toThrow();  // 期望抛出错误
});
```

### 6.3 自定义域名异常路径处理链路

#### 场景描述
用户通过自定义域名访问非预期路径（如 `/settings`、`/api/xxx` 等）。

#### 处理链路

**1. `shareDomains` 中间件**（`server/middlewares/shareDomains.ts`）：

```typescript
export default function shareDomains() {
  return async function shareDomainsMiddleware(ctx: Context, next: Next) {
    const isCustomDomain = parseDomain(ctx.host).custom;
    
    if (env.isDevelopment || (isCustomDomain && env.isCloudHosted)) {
      const share = await Share.unscoped().findOne({
        where: {
          domain: ctx.hostname,    // 根据域名查找对应的 Share
          published: true,
          revokedAt: { [Op.is]: null },
        },
      });
      ctx.state.rootShare = share;    // 注入到请求上下文
    }
    
    return next();
  };
}
```

**2. 路由优先级处理**（`server/routes/index.ts`）：

```typescript
// 顺序很重要！shareDomains() 中间件先执行
router.use(shareDomains());

// 有效路径 1：子文档访问
router.get("/doc/:documentSlug", async (ctx, next) => {
  if (ctx.state?.rootShare) {
    return renderShare(ctx, next);  // 自定义域名下走分享渲染
  }
  return next();
});

// 有效路径 2：站点地图
router.get("/sitemap.xml", async (ctx) => {
  if (ctx.state?.rootShare) {
    ctx.redirect(`/api/shares.sitemap?id=${ctx.state?.rootShare.id}`);
  } else {
    ctx.status = 404;
  }
});

// Catch-all 路由（最后执行）
router.get("*", async (ctx, next) => {
  if (ctx.state?.rootShare) {
    // 只允许根路径，其他路径返回 404
    if (ctx.path !== "/") {
      ctx.status = 404;
      return;
    }
    return renderShare(ctx, next);
  }
  // ... 普通域名处理
});
```

#### 完整处理流程图

```
用户通过自定义域名 docs.example.com/xxx 访问
        ↓
┌─────────────────────────────────────┐
│  shareDomains() 中间件执行：         │
│  1. 检测到是自定义域名               │
│  2. 查找 domain = docs.example.com   │
│     且 published = true              │
│     且 revokedAt IS NULL 的 Share    │
│  3. 设置 ctx.state.rootShare = share │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│  路由匹配（按顺序）：                │
│  检查请求路径是否匹配已知路由         │
└─────────────────────────────────────┘
        ↓
   ┌────┼────┐
   │    │    │
  /doc/:slug  /sitemap.xml  其他路径
   │    │    │
   ↓    ↓    ↓
 有效  有效  进入
路径  路径  catch-all
   │    │    │
   ↓    ↓    ↓
renderShare  检查
          path === "/"？
               │
          ┌────┴────┐
          │         │
         是        否
          │         │
          ↓         ↓
       renderShare  status=404
```

#### 自定义域名下的有效路径列表

| 路径 | 处理方式 | 说明 |
|------|---------|------|
| `/` | `renderShare` | 根路径，访问共享首页 |
| `/doc/:documentSlug` | `renderShare` | 访问共享子文档 |
| `/sitemap.xml` | 重定向到 `/api/shares.sitemap` | 站点地图（SEO） |
| 其他任意路径 | 返回 404 | 防止未授权访问 |

#### 设计意图

1. **URL 空间隔离**：
   - 自定义域名的 URL 空间完全受控于分享配置
   - 不允许访问 `/settings`、`/api`、`/admin` 等内部路径
   - 防止信息泄露和安全漏洞

2. **简洁的用户体验**：
   - 访问自定义域名根路径直接进入共享文档
   - 不需要用户记住复杂的 `/s/:shareId` 路径
   - 符合品牌化需求

3. **安全边界**：
   - 自定义域名与主应用域名完全隔离
   - Cookie 不会跨域名泄露
   - 防止通过自定义域名访问非共享内容

---

## 7. 差异对比总结

### 7.1 数据层差异

| 维度 | 匿名访问 | 已登录但无权限 | 已登录且有权限 |
|------|---------|---------------|---------------|
| **用户信息** | 隐藏 createdBy、updatedBy、collaboratorIds | 隐藏 createdBy、updatedBy、collaboratorIds | 完整显示 |
| **结构信息** | 隐藏 collectionId、parentDocumentId | 隐藏 collectionId、parentDocumentId | 完整显示 |
| **时间戳** | 可选隐藏 updatedAt | 可选隐藏 updatedAt | 完整显示 |
| **协作数据** | 隐藏 tasks、comments | 隐藏 tasks、comments | 完整显示 |
| **内部设置** | 隐藏 sharing、commenting、permission 等 | 隐藏 sharing、commenting、permission 等 | 完整显示 |
| **文档内容** | 移除评论标记，重写内部链接 | 移除评论标记，重写内部链接 | 原始内容 |
| **附件 URL** | 有时效性的签名 URL | 有时效性的签名 URL | 内部 URL |

### 7.2 功能层差异

| 功能 | 匿名访问 | 已登录但无权限 | 已登录且有权限 |
|------|---------|---------------|---------------|
| **编辑能力** | 只读模式 (`readOnly: true`) | 只读模式 (`readOnly: true`) | 根据权限决定 |
| **命令栏** | `SharedCommandBar`（受限功能） | `SharedCommandBar`（受限功能） | 完整 `CommandBar` |
| **侧边栏** | `SharedSidebar`（仅共享树） | `SharedSidebar`（仅共享树） | 完整侧边栏 |
| **协作功能** | 禁用评论、任务、提及 | 禁用评论、任务、提及 | 根据权限启用 |
| **品牌展示** | 可能显示 Outline 品牌 | 可能显示 Outline 品牌 | 不显示 |
| **嵌入支持** | 可嵌入 iframe（可选） | 可嵌入 iframe（可选） | 不适用 |

### 7.3 安全层差异

| 安全措施 | 公开共享视图 | 登录用户视图 |
|----------|-------------|-------------|
| **身份验证** | 可选（通过 `auth({ optional: true })`） | 必需 |
| **权限检查** | 基于分享链接有效性 + 用户权限 | 基于团队/集合/文档权限 |
| **访问范围** | 限制在共享树内 | 根据用户权限决定 |
| **数据脱敏** | 多级脱敏（用户信息、结构信息、协作数据） | 无 |
| **URL 签名** | 附件 URL 有时效性签名 | 内部 URL 无签名 |
| **索引控制** | `allowIndexing` 设置控制搜索引擎索引 | 无（通常不索引） |

---

## 6. 关键代码位置索引

### 6.1 服务端代码

| 功能 | 文件路径 | 关键函数/类 |
|------|----------|-------------|
| 共享路由配置 | `server/routes/index.ts` | 路由定义 |
| 共享页面渲染 | `server/routes/app.ts` | `renderShare`, `renderApp` |
| 共享 API | `server/routes/api/shares/shares.ts` | `shares.info` 等 |
| 共享加载逻辑 | `server/commands/shareLoader.ts` | `loadPublicShare` |
| 文档序列化 | `server/presenters/document.ts` | `presentDocument` |
| 集合序列化 | `server/presenters/collection.ts` | `presentCollection` |
| 文档内容处理 | `server/models/helpers/DocumentHelper.tsx` | `DocumentHelper` |
| 文档权限策略 | `server/policies/document.ts` | 权限规则 |
| 共享权限策略 | `server/policies/share.ts` | 权限规则 |

### 6.2 前端代码

| 功能 | 文件路径 | 关键组件/函数 |
|------|----------|--------------|
| 共享页面入口 | `app/scenes/Shared/index.tsx` | `SharedScene` |
| 共享文档视图 | `app/scenes/Shared/Document.tsx` | `Document` |
| 共享集合视图 | `app/scenes/Shared/Collection.tsx` | `Collection` |
| 共享上下文 | `shared/hooks/useShare.ts` | `ShareContext` |
| 路由辅助 | `app/utils/routeHelpers.ts` | `sharedModelPath` |

---

## 7. 总结

Outline 的公开共享文档系统采用了多层级的安全设计和差异化渲染策略：

1. **链路层**：通过独立的路由体系、自定义域名支持、多格式输出（HTML/Markdown）构建完整的分享生态。

2. **数据层**：在序列化阶段进行精细的权限裁剪，隐藏敏感的用户信息、内部结构和协作数据。

3. **内容层**：对文档内容进行处理，移除评论、重写内部链接、生成有时效性的附件 URL。

4. **渲染层**：服务端进行 SEO 优化的 SSR 渲染，前端使用独立的组件树确保只读模式和受限功能。

5. **安全层**：多层安全检查确保只有授权用户能访问，数据脱敏保护隐私，URL 签名防止未授权访问。

这种设计确保了公开共享文档的安全性、可访问性和用户体验之间的平衡，同时保持了代码的可维护性和扩展性。
