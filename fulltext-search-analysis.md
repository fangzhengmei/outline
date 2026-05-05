# Outline 全文搜索权限裁剪机制分析报告

## 概述

Outline 的全文搜索基于 PostgreSQL 的 `tsvector` 全文索引实现，并通过 **数据库查询层面的多维度权限过滤** 来确保用户只能搜索到自己有权限访问的文档。本文档详细分析其权限裁剪的实现机制。

---

## 一、核心架构

### 1.1 搜索提供者

Outline 使用 `PostgresSearchProvider` 作为 PostgreSQL 全文搜索的核心实现：

- **位置**: `plugins/search-postgres/server/PostgresSearchProvider.ts`
- **基类**: `BaseSearchProvider` (`server/utils/BaseSearchProvider.ts`)

### 1.2 主要搜索方法

| 方法 | 用途 | 权限方式 |
|------|------|----------|
| `searchForUser()` | 用户级全文搜索 | 结合用户权限过滤 |
| `searchForTeam()` | 团队级搜索（共享文档场景） | 团队范围限制 |
| `searchTitlesForUser()` | 标题搜索 | 轻量级权限过滤 |
| `searchCollectionsForUser()` | 集合搜索 | 集合权限过滤 |

---

## 二、权限裁剪核心机制

### 2.1 `buildWhere()` 方法分析

**位置**: `PostgresSearchProvider.ts:609-793`

`buildWhere()` 是权限裁剪的核心方法，它构建包含多重权限条件的 SQL `WHERE` 子句，确保搜索从一开始就被限制在用户有权访问的范围内。

#### 2.1.1 基础过滤条件

```typescript
const where: WhereOptions<Document> & {
  [Op.or]: WhereOptions<Document>[];
  [Op.and]: WhereOptions<Document>[];
} = {
  teamId,  // 限制在用户/团队所属的团队
  [Op.or]: [],  // 权限条件的 OR 组合
  [Op.and]: [
    { deletedAt: { [Op.eq]: null } },  // 排除已删除文档
  ],
};
```

#### 2.1.2 四层权限 OR 条件

**用户模式下** (`model instanceof User`)，向 `[Op.or]` 数组添加以下条件：

**第一层：文档直接用户成员资格**
```typescript
{ "$memberships.id$": { [Op.ne]: null } }
// 通过 JOIN user_permissions 表检查用户是否有该文档的直接权限
```

**第二层：文档直接组成员资格**
```typescript
{ "$groupMemberships.id$": { [Op.ne]: null } }
// 通过 JOIN group_permissions -> groups -> group_users 检查组成员资格
```

**第三层：用户自己的无集合草稿（可选）**
```typescript
// 仅当 statusFilter 包含 Draft 时添加
{
  createdById: model.id,        // 用户自己创建
  collectionId: { [Op.is]: null },  // 没有归属集合
  publishedAt: { [Op.eq]: null },   // 未发布（草稿）
  archivedAt: { [Op.eq]: null },    // 未归档
}
```

**第四层：有权访问的集合**
```typescript
const collectionIds = await model.collectionIds();
if (collectionIds.length) {
  where[Op.or].push({ collectionId: collectionIds });
}
```

---

### 2.2 `collectionIds()` 方法深度分析

**位置**: `server/models/User.ts:476-553`

该方法获取用户有权访问的所有集合 ID，是权限过滤的关键数据源。

#### 2.2.1 集合权限判断逻辑

```typescript
const collectionStubs = await Collection.findAll({
  attributes: ["id"],
  where: {
    teamId: this.teamId,  // 同一团队内
    [Op.or]: [
      // 条件A: 非访客用户 + 集合有公开权限（Read/ReadWrite）
      ...(this.isGuest ? [] : [
        { permission: { [Op.in]: Object.values(CollectionPermission) } }
      ]),
      
      // 条件B: 用户有该集合的直接成员资格
      { "$memberships.id$": { [Op.ne]: null } },
      
      // 条件C: 用户所属的组有该集合的成员资格
      { "$groupMemberships.id$": { [Op.ne]: null } },
    ],
  },
  include: [
    // 关联用户成员资格
    {
      association: "memberships",
      where: { userId: this.id },
      required: false,
    },
    // 关联组成员资格（多级 JOIN）
    {
      association: "groupMemberships",
      required: false,
      include: [{
        association: "group",
        required: true,
        include: [{
          association: "groupUsers",
          required: true,
          where: { userId: this.id },
        }],
      }],
    },
  ],
});
```

#### 2.2.2 缓存机制

```typescript
// 使用 Redis 缓存，TTL 10秒
return (
  (await CacheHelper.getDataOrSet<string[]>(
    RedisPrefixHelper.getUserCollectionIdsKey(this.id),
    fetchCollectionIds,
    10  // TTL: 10秒
  )) ?? []
);
```

**缓存失效触发点**: `server/models/UserMembership.ts:311-315`
```typescript
@AfterDestroy
static async invalidateCollectionIdsAfterDestroy(model: UserMembership) {
  if (model.collectionId) {
    await CacheHelper.clearData(
      RedisPrefixHelper.getUserCollectionIdsKey(model.userId)
    );
  }
}
```

---

### 2.3 `searchForUser()` 完整流程

**位置**: `PostgresSearchProvider.ts:378-466`

#### 2.3.1 流程步骤

```
1. 构建带权限的 WHERE 条件
      ↓
2. 构建全文搜索选项（tsvector 匹配、排序）
      ↓
3. 通过 JOIN 关联成员资格表进行第一层过滤
      ↓
4. 执行全文搜索查询（带 LIMIT/OFFSET）
      ↓
5. 再次使用 withMembershipScope 获取完整文档数据
      ↓
6. 构建带上下文摘要的搜索响应
```

#### 2.3.2 关键代码解析

**第一步：构建 WHERE 条件**
```typescript
const where = await PostgresSearchProvider.buildWhere(user, options);
```

**第二步：关联成员资格表**
```typescript
const include = [
  // 用户直接成员资格
  {
    association: "memberships",
    where: { userId: user.id },
    required: false,
    separate: false,
  },
  // 组成员资格（多级关联）
  {
    association: "groupMemberships",
    required: false,
    separate: false,
    include: [{
      association: "group",
      required: true,
      include: [{
        association: "groupUsers",
        required: true,
        where: { userId: user.id },
      }],
    }],
  },
];
```

**第三步：执行搜索**
```typescript
const results = (await Document.unscoped().findAll({
  ...findOptions,      // 包含 tsvector 匹配和排序
  subQuery: false,     // 禁用子查询，允许 JOIN 过滤生效
  include,             // 成员资格关联
  where,               // 权限条件
  limit,
  offset,
})) as unknown as RankedDocument[];
```

**第四步：获取完整文档数据**
```typescript
const documents = await Document.withMembershipScope(
  user.id, 
  { includeDrafts: true }
).findAll({
  where: {
    teamId: user.teamId,
    id: map(results, "id"),  // 基于搜索结果的 ID 再次查询
  },
});
```

---

## 三、权限模型详解

### 3.1 文档访问权限判定逻辑

**位置**: `server/policies/document.ts:17-30`

```typescript
allow(User, "read", Document, (actor, document) =>
  and(
    isTeamModel(actor, document),  // 同一团队
    or(
      // 条件1: 文档有直接的成员资格（用户或组）
      includesMembership(document, [
        DocumentPermission.Read,
        DocumentPermission.ReadWrite,
        DocumentPermission.Admin,
      ]),
      // 条件2: 是用户自己的无集合草稿
      and(!!document?.isDraft, actor.id === document?.createdById),
      // 条件3: 对集合有读取权限
      can(actor, "readDocument", document?.collection)
    )
  )
);
```

### 3.2 成员资格检查 `includesMembership()`

**位置**: `server/policies/document.ts:270-303`

```typescript
function includesMembership(
  document: Document | null,
  permissions: DocumentPermission[]
) {
  // 需要预加载 memberships 和 groupMemberships
  invariant(document.memberships, "...");
  invariant(document.groupMemberships, "...");

  const permissionSet = new Set(permissions);
  const membershipIds: string[] = [];

  // 检查用户直接成员资格
  for (const membership of document.memberships) {
    if (permissionSet.has(membership.permission as DocumentPermission)) {
      membershipIds.push(membership.id);
    }
  }

  // 检查组成员资格
  for (const membership of document.groupMemberships) {
    if (permissionSet.has(membership.permission as DocumentPermission)) {
      membershipIds.push(membership.id);
    }
  }

  return membershipIds.length > 0 ? membershipIds : false;
}
```

---

## 四、多层权限防护机制

### 4.1 第一层：SQL 查询层面过滤（最关键）

**WHERE 条件结构**：
```sql
WHERE 
  teamId = '用户团队ID'
  AND deletedAt IS NULL
  AND (
    -- 权限 OR 条件
    "$memberships.id$" IS NOT NULL          -- 文档用户成员资格
    OR "$groupMemberships.id$" IS NOT NULL   -- 文档组成员资格
    OR collectionId IN (用户有权的集合ID)     -- 集合权限
    OR (用户自己的无集合草稿条件)               -- 私有草稿
  )
  AND (全文搜索条件...)
```

### 4.2 第二层：`withMembershipScope` 作用域

**位置**: `server/models/Document.ts:707-721`

```typescript
static withMembershipScope(
  userId: string,
  options?: FindOptions<Document> & { includeDrafts?: boolean }
) {
  return this.scope([
    options?.includeDrafts ? "withDrafts" : "defaultScope",
    "withoutState",
    { method: ["withViews", userId] },
    { method: ["withMembership", userId, options?.paranoid] },
  ]);
}
```

**`withMembership` 作用域**：自动预加载用户和组成员资格关联。

### 4.3 第三层：`findByIds()` 的后过滤

**位置**: `server/models/Document.ts:808-857`

```typescript
return documents.filter(
  (doc) =>
    // 非私有集合且非访客用户 → 有权
    (!doc.collection?.isPrivate && !user?.isGuest) ||
    // 集合有用户成员资格
    (doc.collection?.memberships.length || 0) > 0 ||
    // 集合有组成员资格
    (doc.collection?.groupMemberships.length || 0) > 0 ||
    // 文档有用户成员资格
    doc.memberships.length > 0 ||
    // 文档有组成员资格
    doc.groupMemberships.length > 0
);
```

---

## 五、特殊场景处理

### 5.1 草稿文档（Draft）

**搜索条件中的草稿处理** (`buildWhere` 中)：
```typescript
if (options.statusFilter?.includes(StatusFilter.Draft)) {
  where[Op.or].push({
    createdById: model.id,
    collectionId: { [Op.is]: null },
    publishedAt: { [Op.eq]: null },
    archivedAt: { [Op.eq]: null },
  });
}
```

**状态查询的草稿处理**：
```typescript
if (options.statusFilter?.includes(StatusFilter.Draft) && model instanceof User) {
  statusQuery.push({
    [Op.and]: [
      {
        publishedAt: { [Op.eq]: null },
        archivedAt: { [Op.eq]: null },
        [Op.or]: [
          { createdById: model.id },
          { "$memberships.id$": { [Op.ne]: null } },
        ],
      },
    ],
  });
}
```

### 5.2 共享文档搜索（`searchForTeam`）

**位置**: `PostgresSearchProvider.ts:185-274`

团队级搜索用于公开共享链接场景，权限条件更简单：
- 仅限制 `teamId`
- 不加入用户特定的成员资格条件
- 可以通过 `share` 参数进一步限制范围

```typescript
if (options.share) {
  if (options.share.collectionId) {
    // 限制在共享集合的文档范围内
    documentIds = sharedCollection.getAllDocumentIds();
  } else if (options.share.documentId && options.share.includeChildDocuments) {
    // 限制在共享文档及其子文档范围内
    documentIds = [sharedDocument.id, ...childDocumentIds];
  }
  
  where[Op.and].push({ id: documentIds });
}
```

### 5.3 指定集合搜索

当用户指定 `collectionId` 时：
```typescript
const collectionIds = options.collectionId
  ? [options.collectionId]  // 使用指定的集合
  : await model.collectionIds();  // 否则获取所有有权集合

// 并且假设路由层已经做了授权检查
// "authorization has already been done in the router"
```

---

## 六、性能优化策略

### 6.1 数据库索引利用

1. **全文索引**: `searchVector` 列的 GIN 索引
2. **外键索引**: `teamId`, `collectionId`, `createdById` 等都有索引
3. **成员资格表索引**: `user_permissions` 和 `group_permissions` 表有联合索引

### 6.2 缓存策略

| 缓存内容 | TTL | 触发失效 |
|---------|-----|---------|
| 用户集合权限列表 (`collectionIds`) | 10秒 | 成员资格增删时 |

### 6.3 查询优化

1. **`subQuery: false`**: 禁用 Sequelize 的子查询包装，让 JOIN 过滤直接生效
2. **先搜索 ID 再获取详情**: 减少大字段（`text`, `searchVector`）在排序阶段的加载
3. **`separate: true`**: 对于复杂嵌套关联，使用单独查询避免 JOIN 爆炸

---

## 七、关键代码位置总结

| 功能 | 文件位置 | 关键方法/行号 |
|------|---------|--------------|
| PostgreSQL 搜索核心 | `plugins/search-postgres/server/PostgresSearchProvider.ts` | 全文 |
| 用户集合权限获取 | `server/models/User.ts` | `collectionIds()`: 476 |
| 文档权限策略 | `server/policies/document.ts` | `read` 权限: 17 |
| 成员资格检查 | `server/policies/document.ts` | `includesMembership()`: 270 |
| 文档作用域 | `server/models/Document.ts` | `withMembershipScope()`: 707 |
| 缓存失效 | `server/models/UserMembership.ts` | `invalidateCollectionIdsAfterDestroy()`: 311 |

---

## 八、设计亮点

1. **数据库层面过滤**: 所有权限检查都在 SQL 查询中完成，避免先查后过滤的性能问题
2. **多层 OR 条件**: 文档成员资格、组成员资格、集合权限三者是 OR 关系，任一满足即可
3. **双重保险**: 搜索时的 WHERE 过滤 + 获取详情时的 `withMembershipScope`
4. **智能缓存**: 集合权限列表缓存，减少重复查询
5. **草稿特殊处理**: 无集合的私有草稿只能由创建者访问

---

## 九、流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                     用户发起搜索请求                              │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  PostgresSearchProvider.searchForUser(user, options)           │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. buildWhere(user, options)  构建带权限的 WHERE 条件          │
│     ├── teamId = 用户团队ID                                      │
│     ├── deletedAt IS NULL                                        │
│     └── [Op.or] 权限条件组                                       │
│         ├── 文档有用户成员资格 ($memberships.id$ IS NOT NULL)  │
│         ├── 文档有组成员资格 ($groupMemberships.id$ IS NOT NULL)│
│         ├── 属于有权访问的集合 (collectionId IN collectionIds)  │
│         └── 是用户自己的无集合草稿 (statusFilter 包含 Draft 时) │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 获取 collectionIds()  (可能走 Redis 缓存)                   │
│     查询用户有权访问的所有集合 ID                                 │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 构建 include 关联                                             │
│     ├── memberships (用户直接成员资格)                           │
│     └── groupMemberships -> group -> groupUsers (组成员资格)    │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 执行全文搜索查询                                              │
│     Document.unscoped().findAll({                                │
│       ...findOptions,  // ts_rank 排序, searchVector @@ query  │
│       subQuery: false, // 允许 JOIN 过滤生效                     │
│       include,         // 成员资格关联                            │
│       where,           // 权限条件                                │
│       limit, offset    // 分页                                    │
│     })                                                             │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. 使用 withMembershipScope 获取完整文档数据                    │
│     Document.withMembershipScope(user.id).findAll({             │
│       where: { id: 搜索结果的 ID 列表 }                          │
│     })                                                             │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. 构建搜索响应                                                  │
│     ├── 计算 searchRanking                                       │
│     ├── 提取上下文摘要 (buildResultContext)                      │
│     └── 返回 results + total                                     │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                     返回有权限的搜索结果                          │
└─────────────────────────────────────────────────────────────────┘
```
