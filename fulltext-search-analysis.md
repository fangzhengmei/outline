# Outline 全文搜索权限裁剪机制分析报告

## 概述

Outline 的全文搜索基于 PostgreSQL 的 `tsvector` 全文索引实现，并通过 **SQL 层面的 WHERE 条件与 JOIN 关联协同过滤** 来确保用户只能搜索到自己有权限访问的文档。本文档详细分析其权限裁剪的实现机制，并纠正之前报告中的两处偏差。

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

## 二、权限裁剪主链路：SQL 层面的协同过滤

### 2.1 真正的执行顺序

`searchForUser()` 的权限裁剪完全在 **第一次 SQL 查询** 中完成，而非依赖后过滤。以下是真正的执行顺序：

```
┌─────────────────────────────────────────────────────────────────┐
│  1. buildWhere() 构建 [Op.or] 权限条件数组                       │
│     ├── $memberships.id$ IS NOT NULL     (文档用户成员资格)     │
│     ├── $groupMemberships.id$ IS NOT NULL (文档组成员资格)      │
│     ├── collectionId IN collectionIds     (有权访问的集合)      │
│     └── (用户自己的无集合草稿)             (可选, statusFilter)  │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 构建 include 关联（LEFT JOIN）                               │
│     ├── memberships (用户直接成员资格表)                         │
│     └── groupMemberships -> group -> groupUsers (组成员资格链)  │
│     目的：让 WHERE 中的 $xxx.id$ 虚拟列条件能够生效              │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. subQuery: false - 关键配置                                   │
│     ├── 禁用 Sequelize 默认的子查询包装                          │
│     ├── 让 JOIN 结果直接参与 WHERE 条件过滤                      │
│     └── 这是权限条件能够生效的核心前提                            │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 执行第一次查询（真正的权限过滤发生在这里）                    │
│     Document.unscoped().findAll({                                │
│       subQuery: false,  // 关键！                                │
│       include,            // JOIN 成员资格表                     │
│       where,              // 包含 [Op.or] 权限条件              │
│       ...findOptions,     // 全文搜索、排序                      │
│     })                                                             │
│                                                                  │
│     此时返回的 results 已经是用户有权访问的文档 ID 列表          │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. 第二次查询（仅获取完整关联数据，非权限过滤）                  │
│     Document.withMembershipScope(user.id).findAll({             │
│       where: { id: map(results, "id") }  // 基于已过滤的 ID     │
│     })                                                             │
│                                                                  │
│     目的：获取 collection、createdBy、updatedBy 等关联数据       │
│     注意：这一步不做权限过滤，因为前一步已经过滤过了              │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. 构建响应返回                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 `buildWhere()` 详解

**位置**: `PostgresSearchProvider.ts:609-793`

`buildWhere()` 是权限条件构建的核心。它创建一个包含 `[Op.or]` 数组的 WHERE 对象：

```typescript
const where: WhereOptions<Document> & {
  [Op.or]: WhereOptions<Document>[];
  [Op.and]: WhereOptions<Document>[];
} = {
  teamId,  // 基础条件：同一团队
  [Op.or]: [],  // 权限条件组 - OR 关系
  [Op.and]: [
    { deletedAt: { [Op.eq]: null } },  // 排除已删除
  ],
};
```

#### 四层权限 OR 条件（用户模式）

```typescript
if (model instanceof User) {
  // 条件1: 文档有用户直接成员资格
  // 依赖: include 中的 memberships JOIN
  where[Op.or].push({ "$memberships.id$": { [Op.ne]: null } });

  // 条件2: 文档有组直接成员资格
  // 依赖: include 中的 groupMemberships JOIN 链
  where[Op.or].push({ "$groupMemberships.id$": { [Op.ne]: null } });

  // 条件3: 用户自己的无集合草稿（可选）
  // 不依赖 JOIN，独立条件
  if (options.statusFilter?.includes(StatusFilter.Draft)) {
    where[Op.or].push({
      createdById: model.id,
      collectionId: { [Op.is]: null },
      publishedAt: { [Op.eq]: null },
      archivedAt: { [Op.eq]: null },
    });
  }
}

// 条件4: 属于有权访问的集合
// 不依赖 JOIN，通过 collectionIds() 获取
const collectionIds = await model.collectionIds();
if (collectionIds.length) {
  where[Op.or].push({ collectionId: collectionIds });
}
```

### 2.3 `include` 关联详解

**位置**: `PostgresSearchProvider.ts:392-421`

```typescript
const include = [
  // LEFT JOIN user_permissions
  {
    association: "memberships",
    where: { userId: user.id },  // 仅当前用户的成员资格
    required: false,              // LEFT JOIN
    separate: false,              // 不单独查询，直接 JOIN
  },
  // LEFT JOIN group_permissions 
  //   -> INNER JOIN groups 
  //   -> INNER JOIN group_users (where userId = 当前用户)
  {
    association: "groupMemberships",
    required: false,
    separate: false,
    include: [{
      association: "group",
      required: true,  // INNER JOIN
      include: [{
        association: "groupUsers",
        required: true,  // INNER JOIN
        where: { userId: user.id },
      }],
    }],
  },
];
```

**关键点**：
- `required: false` = LEFT JOIN，不会排除没有成员资格的行
- 但 WHERE 中的 `$memberships.id$ IS NOT NULL` 会筛选出有成员资格的行
- `separate: false` = 直接在主查询中 JOIN，不使用单独查询

### 2.4 `subQuery: false` 的关键作用

**位置**: `PostgresSearchProvider.ts:426`

```typescript
const results = (await Document.unscoped().findAll({
  ...findOptions,
  subQuery: false,  // 关键配置！
  include,
  where,
  limit,
  offset,
})) as unknown as RankedDocument[];
```

**为什么重要**：

Sequelize 默认行为（`subQuery: true`）：
```sql
SELECT * FROM "documents"
WHERE "id" IN (
  SELECT "id" FROM "documents"
  LEFT JOIN "user_permissions" ...
  WHERE ...
)
-- 问题：子查询中的 $memberships.id$ 引用可能无法正确传递到外层
```

`subQuery: false` 后的行为：
```sql
SELECT * FROM "documents"
LEFT JOIN "user_permissions" AS "memberships" 
  ON "documents"."id" = "memberships"."documentId" 
  AND "memberships"."userId" = 'xxx'
LEFT JOIN "group_permissions" AS "groupMemberships" ...
WHERE 
  "documents"."teamId" = 'xxx'
  AND "documents"."deletedAt" IS NULL
  AND (
    "memberships"."id" IS NOT NULL          -- 条件1
    OR "groupMemberships"."id" IS NOT NULL  -- 条件2
    OR "documents"."collectionId" IN (...)   -- 条件3
    OR (...草稿条件...)                       -- 条件4
  )
  AND (全文搜索条件...)
```

**结论**：`subQuery: false` 让 `include` 的 JOIN 结果直接参与 WHERE 条件过滤，这是 `$memberships.id$ IS NOT NULL` 这类条件能够生效的核心前提。

---

## 三、集合权限获取与缓存机制

### 3.1 `collectionIds()` 方法

**位置**: `server/models/User.ts:476-553`

该方法获取用户有权访问的所有集合 ID，用于 `buildWhere` 中的条件4。

```typescript
public collectionIds = async (options: FindOptions<Collection> = {}) => {
  const fetchCollectionIds = async () => {
    const collectionStubs = await Collection.findAll({
      attributes: ["id"],
      where: {
        teamId: this.teamId,
        [Op.or]: [
          // 条件A: 非访客用户 + 集合有公开权限
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
        { association: "memberships", where: { userId: this.id }, required: false },
        { association: "groupMemberships", ... /* 组成员资格链 */ },
      ],
    });
    return Array.from(new Set(collectionStubs.map((c) => c.id)));
  };

  // 缓存逻辑
  const hasOptions = options.transaction || options.paranoid === false || options.lock;
  if (hasOptions) {
    return fetchCollectionIds();  // 有特殊选项时不缓存
  }
  return (
    (await CacheHelper.getDataOrSet<string[]>(
      RedisPrefixHelper.getUserCollectionIdsKey(this.id),
      fetchCollectionIds,
      10  // TTL: 10秒
    )) ?? []
  );
};
```

### 3.2 缓存失效机制（修正：创建和删除都会失效）

**位置**: `server/models/UserMembership.ts:227-234` 和 `311-318`

之前的报告只提到了删除时失效，实际上 **创建和删除** 时都会失效缓存：

```typescript
// 新增授权时失效
@AfterCreate
static async invalidateCollectionIdsAfterCreate(model: UserMembership) {
  if (model.collectionId) {
    await CacheHelper.clearData(
      RedisPrefixHelper.getUserCollectionIdsKey(model.userId)
    );
  }
}

// 移除授权时失效
@AfterDestroy
static async invalidateCollectionIdsAfterDestroy(model: UserMembership) {
  if (model.collectionId) {
    await CacheHelper.clearData(
      RedisPrefixHelper.getUserCollectionIdsKey(model.userId)
    );
  }
}
```

**注意**：这两个钩子只有在 `model.collectionId` 存在时才会失效缓存。也就是说：
- **集合级别的成员资格变化** → 失效 `collectionIds` 缓存
- **文档级别的成员资格变化** → 不失效 `collectionIds` 缓存（因为 `collectionId` 为 null）

文档级别的成员资格变化不需要失效缓存，因为：
1. 文档权限条件 (`$memberships.id$ IS NOT NULL`) 直接通过 JOIN 实时检查
2. 不依赖 `collectionIds()` 的缓存结果

---

## 四、关于 `findByIds()` 的澄清

### 4.1 它不是搜索流程的一部分

**位置**: `server/models/Document.ts:808-857`

`findByIds()` 是一个独立的方法，用于通过 ID 列表查找文档，**不是 `searchForUser()` 流程的一部分**。

```typescript
static async findByIds(
  ids: string[],
  options: Omit<FindOptions<Document>, "where"> & { userId?: string } = {}
): Promise<Document[]> {
  // ... 查询逻辑 ...
  
  if (!userId) {
    return documents;
  }

  // 后过滤逻辑 - 仅当提供了 userId 时执行
  return documents.filter(
    (doc) =>
      (!doc.collection?.isPrivate && !user?.isGuest) ||
      (doc.collection?.memberships.length || 0) > 0 ||
      (doc.collection?.groupMemberships.length || 0) > 0 ||
      doc.memberships.length > 0 ||
      doc.groupMemberships.length > 0
  );
}
```

### 4.2 为什么它不是主链路

`searchForUser()` 的流程：
1. **第一次查询**：`Document.unscoped().findAll()` - 带 `subQuery: false` + `include` + `where` - **这是真正的权限过滤**
2. **第二次查询**：`Document.withMembershipScope().findAll()` - 基于已过滤的 ID，**仅获取关联数据**

注意：第二次查询用的是 `withMembershipScope`，不是 `findByIds`。

`withMembershipScope` 的作用：
- 应用 `withMembership` scope，预加载成员资格关联
- 不做额外的过滤，因为查询条件是 `id IN (已过滤的 ID 列表)`

---

## 五、SQL 层面的权限过滤原理解析

### 5.1 完整的 SQL 结构

当执行 `searchForUser()` 时，生成的 SQL 大致如下：

```sql
SELECT 
  "documents"."id",
  ts_rank(...) AS "searchRanking"
FROM "documents"

-- JOIN 1: 用户直接成员资格（LEFT JOIN）
LEFT JOIN "user_permissions" AS "memberships" 
  ON "documents"."id" = "memberships"."documentId" 
  AND "memberships"."userId" = '当前用户ID'
  AND "memberships"."deletedAt" IS NULL

-- JOIN 2: 组成员资格链（LEFT JOIN + INNER JOIN）
LEFT JOIN "group_permissions" AS "groupMemberships" 
  ON "documents"."id" = "groupMemberships"."documentId"
  AND "groupMemberships"."deletedAt" IS NULL
INNER JOIN "groups" AS "groupMemberships->group" 
  ON "groupMemberships"."groupId" = "groupMemberships->group"."id"
  AND "groupMemberships->group"."deletedAt" IS NULL
INNER JOIN "group_users" AS "groupMemberships->group->groupUsers" 
  ON "groupMemberships->group"."id" = "groupMemberships->group->groupUsers"."groupId"
  AND "groupMemberships->group->groupUsers"."userId" = '当前用户ID'

WHERE 
  -- 基础条件
  "documents"."teamId" = '用户团队ID'
  AND "documents"."deletedAt" IS NULL
  
  -- 权限 OR 条件组（核心）
  AND (
    -- 条件1: 文档有用户直接成员资格
    "memberships"."id" IS NOT NULL
    
    -- 条件2: 文档有组直接成员资格
    OR "groupMemberships"."id" IS NOT NULL
    
    -- 条件3: 属于有权访问的集合
    OR "documents"."collectionId" IN ('集合ID1', '集合ID2', ...)
    
    -- 条件4: 用户自己的无集合草稿（如果 statusFilter 包含 Draft）
    OR (
      "documents"."createdById" = '当前用户ID'
      AND "documents"."collectionId" IS NULL
      AND "documents"."publishedAt" IS NULL
      AND "documents"."archivedAt" IS NULL
    )
  )
  
  -- 全文搜索条件
  AND "searchVector" @@ to_tsquery('english', '搜索词')

ORDER BY 
  "searchRanking" DESC,
  "updatedAt" DESC

LIMIT 15 OFFSET 0;
```

### 5.2 为什么 LEFT JOIN + IS NOT NULL 有效

让我们拆解这个模式：

```sql
-- LEFT JOIN：所有文档都会保留，没有成员资格的行中 memberships 列为 NULL
LEFT JOIN "user_permissions" AS "memberships" 
  ON "documents"."id" = "memberships"."documentId"
  AND "memberships"."userId" = '当前用户ID'

-- WHERE 条件：筛选出有成员资格的行
WHERE "memberships"."id" IS NOT NULL
```

**执行过程**：
1. 假设有 1000 个文档
2. LEFT JOIN 后仍然是 1000 行（因为 LEFT JOIN 不排除行）
3. 其中 900 行的 `memberships.id` 为 NULL（没有成员资格）
4. 100 行的 `memberships.id` 有值（有成员资格）
5. `WHERE "memberships"."id" IS NOT NULL` 筛选出这 100 行

**关键点**：
- `required: false` = LEFT JOIN = 不排除行
- 但 `WHERE` 中的 `IS NOT NULL` 会排除没有成员资格的行
- 这是一种"先 JOIN 后过滤"的模式

### 5.3 组成员资格的特殊情况

组成员资格的 JOIN 链是：

```sql
LEFT JOIN "group_permissions" AS "groupMemberships" ...
INNER JOIN "groups" AS "groupMemberships->group" ...
INNER JOIN "group_users" AS "groupMemberships->group->groupUsers" 
  ON ... AND "userId" = '当前用户ID'
```

**为什么用 INNER JOIN？**

- 外层是 LEFT JOIN (`groupMemberships`)
- 内层是 INNER JOIN (`group` 和 `groupUsers`)
- 这样的组合效果：
  - 如果用户不是任何组的成员 → `groupUsers` 匹配失败 → 整个 JOIN 链结果为 NULL
  - 如果用户是某个组的成员，且该组对文档有权限 → `groupMemberships.id` 有值

所以 `$groupMemberships.id$ IS NOT NULL` 能够正确筛选出用户通过组拥有权限的文档。

---

## 六、特殊场景处理

### 6.1 草稿文档（Draft）

草稿有两种情况，处理方式不同：

**情况 A：无集合的草稿**
```typescript
// 在 buildWhere 的 [Op.or] 中添加
{
  createdById: model.id,        // 自己创建
  collectionId: { [Op.is]: null },  // 没有集合
  publishedAt: { [Op.eq]: null },   // 未发布
  archivedAt: { [Op.eq]: null },    // 未归档
}
```
- 这种草稿不依赖任何成员资格或集合权限
- 只能由创建者自己访问

**情况 B：有集合的草稿**
- 不在 `[Op.or]` 中添加特殊条件
- 仍然受集合权限或文档成员资格的限制
- 见 `server/policies/document.ts:26` 的注释：
  > "Drafts in collections remain gated by the collection/membership checks above."

### 6.2 共享文档搜索（`searchForTeam`）

**位置**: `PostgresSearchProvider.ts:185-274`

团队级搜索用于公开共享链接场景，权限条件更简单：

```typescript
async searchForTeam(team: Team, options: SearchOptions = {}) {
  const where = await PostgresSearchProvider.buildWhere(team, {
    ...options,
    statusFilter: [...(options.statusFilter || []), StatusFilter.Published],
  });
  
  // 注意：buildWhere 接收的是 Team，不是 User
  // 所以不会添加用户特定的成员资格条件
  
  if (options.share) {
    // 进一步限制在共享范围内
    where[Op.and].push({ id: documentIds });
  }
  
  // ... 执行查询
}
```

**区别**：
- `searchForUser`：添加 `$memberships.id$ IS NOT NULL` 等用户特定条件
- `searchForTeam`：不添加用户特定条件，仅限制 `teamId`

### 6.3 指定集合搜索

当用户指定 `collectionId` 时：

```typescript
const collectionIds = options.collectionId
  ? [options.collectionId]  // 使用指定的集合
  : await model.collectionIds();  // 否则获取所有有权集合

if (options.collectionId) {
  where[Op.and].push({ collectionId: options.collectionId });
}
// 注意：这里用的是 [Op.and]，不是 [Op.or]
// 因为指定集合后，必须属于该集合
```

**注释说明**：
> "If collectionId is passed as an option it is assumed that the authorization has already been done in the router"

意思是：路由层已经检查过用户对该集合的访问权限，所以这里可以直接使用。

---

## 七、性能优化策略

### 7.1 `subQuery: false` 的性能影响

**优点**：
- 让 JOIN 条件直接参与 WHERE 过滤
- 数据库可以更有效地使用索引
- 避免子查询的额外开销

**注意事项**：
- 当 include 多个关联时，可能产生 JOIN 爆炸（笛卡尔积）
- 但在搜索场景中，只 include 了 memberships 和 groupMemberships，且都有 `where` 条件限制，风险较低

### 7.2 两阶段查询策略

```
阶段1: 搜索 ID + 权限过滤
  - 只 select id 和 searchRanking
  - 不加载大字段（text, searchVector）
  - 带全文搜索和排序
  - 带 LIMIT/OFFSET

阶段2: 获取完整数据
  - 基于阶段1的 ID 列表
  - 加载完整关联数据（collection, createdBy, etc.）
  - 不带全文搜索（因为已经过滤过）
```

**优点**：
- 阶段1的排序更高效（不需要处理大字段）
- 阶段2可以利用 `withMembershipScope` 等预定义作用域

### 7.3 缓存策略

| 缓存内容 | TTL | 失效时机 |
|---------|-----|---------|
| 用户集合权限列表 (`collectionIds`) | 10秒 | 集合级别成员资格创建/删除时 |

**为什么只缓存 collectionIds**：
- `collectionIds` 需要查询 Collection 表 + 多级 JOIN，相对耗时
- 文档权限条件通过 JOIN 实时检查，无法有效缓存
- 缓存 TTL 只有 10秒，平衡了性能和数据一致性

---

## 八、关键代码位置总结

| 功能 | 文件位置 | 关键方法/行号 |
|------|---------|--------------|
| PostgreSQL 搜索核心 | `plugins/search-postgres/server/PostgresSearchProvider.ts` | 全文 |
| 权限条件构建 | `PostgresSearchProvider.ts` | `buildWhere()`: 609 |
| 搜索主流程 | `PostgresSearchProvider.ts` | `searchForUser()`: 378 |
| 关键配置 | `PostgresSearchProvider.ts:426` | `subQuery: false` |
| 用户集合权限获取 | `server/models/User.ts` | `collectionIds()`: 476 |
| 缓存失效（创建） | `server/models/UserMembership.ts` | `invalidateCollectionIdsAfterCreate()`: 227 |
| 缓存失效（删除） | `server/models/UserMembership.ts` | `invalidateCollectionIdsAfterDestroy()`: 311 |
| 文档作用域 | `server/models/Document.ts` | `withMembershipScope()`: 707 |

---

## 九、之前报告的两处偏差修正

### 偏差 1：把无关的 `findByIds()` 当成主链路

**之前的错误描述**：
> 第三层：`findByIds()` 的后过滤

**事实**：
- `findByIds()` 是一个独立的方法，不是 `searchForUser()` 流程的一部分
- `searchForUser()` 的第二次查询用的是 `withMembershipScope`，不是 `findByIds`
- 真正的权限过滤发生在 **第一次查询** 的 SQL 层面，不是后过滤

**修正后的理解**：
```
主链路 = buildWhere 构建 [Op.or] 条件 
       + include 定义 JOIN 关联 
       + subQuery: false 让条件生效
       + 第一次查询执行过滤
```

### 偏差 2：集合权限缓存的失效场景漏了新增授权

**之前的错误描述**：
> 缓存失效触发点: `server/models/UserMembership.ts:311-315`（仅删除）

**事实**：
- `@AfterCreate` 时也会失效：`invalidateCollectionIdsAfterCreate()` (行 227-234)
- `@AfterDestroy` 时失效：`invalidateCollectionIdsAfterDestroy()` (行 311-318)
- 只有集合级别的成员资格变化才会失效缓存（`model.collectionId` 存在时）

**修正后的理解**：
```typescript
// 创建和删除时都会失效
@AfterCreate
@AfterDestroy
static async invalidateCollectionIdsAfterCreate/Destroy(model: UserMembership) {
  if (model.collectionId) {  // 仅集合级别变化
    await CacheHelper.clearData(...);
  }
}
```

---

## 十、设计亮点总结

1. **SQL 层面过滤**：所有权限检查都在 WHERE + JOIN 中完成，避免先查后过滤的性能问题
2. **OR 条件设计**：文档成员资格、组成员资格、集合权限、私有草稿四者是 OR 关系，任一满足即可
3. **`subQuery: false` 的巧妙运用**：让 Sequelize 生成更高效的查询，让 JOIN 条件直接参与过滤
4. **两阶段查询**：先搜索 ID（轻量），再获取详情（完整关联）
5. **精确的缓存失效**：只在集合级别权限变化时失效 `collectionIds` 缓存，文档级别权限实时检查
6. **草稿的特殊处理**：无集合的草稿独立于权限体系，只能由创建者访问

---

## 附录：完整执行流程图（修正版）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         用户发起搜索请求                                  │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  PostgresSearchProvider.searchForUser(user, options)                    │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  步骤 1: 调用 buildWhere(user, options)                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  1.1 初始化 where 对象                                                 │ │
│  │      {                                                                 │ │
│  │        teamId: user.teamId,                                           │ │
│  │        [Op.or]: [],    // 权限条件组                                  │ │
│  │        [Op.and]: [{ deletedAt: null }]                                │ │
│  │      }                                                                 │ │
│  │                                                                       │ │
│  │  1.2 向 [Op.or] 添加用户特定条件                                        │ │
│  │      ├── { "$memberships.id$": { [Op.ne]: null } }                  │ │
│  │      ├── { "$groupMemberships.id$": { [Op.ne]: null } }             │ │
│  │      └── (可选) 用户自己的无集合草稿条件                               │ │
│  │                                                                       │ │
│  │  1.3 获取 collectionIds()（可能走 Redis 缓存，TTL 10秒）             │ │
│  │      向 [Op.or] 添加: { collectionId: collectionIds }                │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  步骤 2: 构建 include 关联（LEFT JOIN）                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  include = [                                                          │ │
│  │    {                                                                   │ │
│  │      association: "memberships",         // user_permissions 表      │ │
│  │      where: { userId: user.id },         // 仅当前用户               │ │
│  │      required: false,                    // LEFT JOIN                 │ │
│  │      separate: false,                    // 不单独查询                │ │
│  │    },                                                                  │ │
│  │    {                                                                   │ │
│  │      association: "groupMemberships",    // group_permissions 表     │ │
│  │      required: false,                    // LEFT JOIN                 │ │
│  │      include: [{                         // 组成员资格链              │ │
│  │        association: "group",             // INNER JOIN groups        │ │
│  │        required: true,                                                │ │
│  │        include: [{                                                     │ │
│  │          association: "groupUsers",      // INNER JOIN group_users   │ │
│  │          required: true,                                               │ │
│  │          where: { userId: user.id },     // 仅当前用户               │ │
│  │        }],                                                             │ │
│  │      }],                                                               │ │
│  │    }                                                                   │ │
│  │  ]                                                                     │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  步骤 3: 执行第一次查询（真正的权限过滤）                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  Document.unscoped().findAll({                                        │ │
│  │    ...findOptions,        // 全文搜索、ts_rank 排序                  │ │
│  │    subQuery: false,       // 关键！禁用子查询包装                     │ │
│  │    include,               // JOIN 成员资格表                          │ │
│  │    where,                 // 包含 [Op.or] 权限条件                   │ │
│  │    limit,                 // 分页                                      │ │
│  │    offset,                // 分页                                      │ │
│  │  })                                                                     │ │
│  │                                                                       │ │
│  │  此时生成的 SQL 核心逻辑:                                              │ │
│  │  SELECT "documents"."id", ts_rank(...) AS "searchRanking"            │ │
│  │  FROM "documents"                                                      │ │
│  │  LEFT JOIN "user_permissions" AS "memberships" ON ...                │ │
│  │  LEFT JOIN "group_permissions" AS "groupMemberships" ON ...          │ │
│  │  INNER JOIN "groups" ...                                               │ │
│  │  INNER JOIN "group_users" ...                                          │ │
│  │  WHERE                                                                  │ │
│  │    "documents"."teamId" = 'xxx'                                        │ │
│  │    AND "documents"."deletedAt" IS NULL                                 │ │
│  │    AND (                                                                │ │
│  │      "memberships"."id" IS NOT NULL          -- 条件1                 │ │
│  │      OR "groupMemberships"."id" IS NOT NULL  -- 条件2                 │ │
│  │      OR "documents"."collectionId" IN (...)    -- 条件3               │ │
│  │      OR (...草稿条件...)                      -- 条件4（可选）         │ │
│  │    )                                                                    │ │
│  │    AND "searchVector" @@ to_tsquery('english', '搜索词')             │ │
│  │  ORDER BY "searchRanking" DESC                                         │ │
│  │  LIMIT 15 OFFSET 0                                                      │ │
│  │                                                                       │ │
│  │  返回结果: 已过滤的文档 ID 列表 + searchRanking                       │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  步骤 4: 执行第二次查询（仅获取完整关联数据）                             │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  Document.withMembershipScope(user.id, { includeDrafts: true })     │ │
│  │    .findAll({                                                          │ │
│  │      where: {                                                          │ │
│  │        teamId: user.teamId,                                            │ │
│  │        id: map(results, "id"),  // 基于第一次查询的结果               │ │
│  │      },                                                                 │ │
│  │    })                                                                   │ │
│  │                                                                       │ │
│  │  目的:                                                                  │ │
│  │    - 获取 collection 关联数据                                          │ │
│  │    - 获取 createdBy、updatedBy 关联数据                                │ │
│  │    - 预加载 memberships 和 groupMemberships                            │ │
│  │                                                                       │ │
│  │  注意: 这一步不做权限过滤，因为查询条件是 id IN (已过滤的列表)        │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  步骤 5: 构建搜索响应                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  buildResponse({                                                      │ │
│  │    query,     // 搜索词，用于生成上下文摘要                           │ │
│  │    results,   // 第一次查询的结果（含 searchRanking）                │ │
│  │    documents, // 第二次查询的结果（含完整关联数据）                   │ │
│  │    count,     // 总数                                                 │ │
│  │  })                                                                     │ │
│  │                                                                       │ │
│  │  每个结果包含:                                                         │ │
│  │    - ranking: searchRanking                                           │ │
│  │    - context: 上下文摘要（高亮匹配词）                                 │ │
│  │    - document: 完整的 Document 对象                                   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         返回有权限的搜索结果                              │
└─────────────────────────────────────────────────────────────────────────┘
```
