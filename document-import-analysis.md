# Outline 文档导入功能分析

## 1. 概述

Outline 支持从多种格式导入文档，主要分为**两条独立的导入路径**：

### 路径一：文件上传导入（Markdown、Docx、HTML、CSV 等）
- **触发方式**：用户上传本地文件
- **处理方式**：同步 API 调用
- **核心组件**：`documentImporter.ts` + `DocumentConverter.ts`

### 路径二：API 集成导入（Notion）
- **触发方式**：OAuth 授权后从 Notion API 拉取
- **处理方式**：事件驱动 + 异步任务队列
- **核心组件**：`NotionImportsProcessor` + `NotionAPIImportTask` + `NotionClient` + `NotionConverter`

---

## 2. 路径一：文件上传导入（Markdown、Docx 等）

### 2.1 整体架构

```
用户上传文件 → documentImporter.ts → DocumentConverter.ts → Prosemirror → Y.Doc → 数据库
```

### 2.2 核心入口

**文件**：`server/commands/documentImporter.ts`

**主流程** (`documentImporter.ts:48-97`):
```typescript
async function documentImporter({
  mimeType,
  fileName,
  content,
  user,
  ctx,
}: Props): Promise<ImportResult> {
  // 1. 从文件名提取标题（去除扩展名）
  const fileTitle = fileName.replace(
    new RegExp(`\\.(${extensions.join("|")})$`, "i"),
    ""
  );

  // 2. 使用 DocumentConverter 转换内容
  const {
    doc,
    title: extractedTitle,
    icon,
  } = await DocumentConverter.convert(content, fileName, mimeType);

  // 3. 标题优先级：提取的标题 > 文件名
  let title = extractedTitle || fileTitle;

  // 4. 替换外部图片为附件
  const processedDoc = await ProsemirrorHelper.replaceImagesWithAttachments(
    ctx,
    doc,
    user
  );

  // 5. 序列化为 Markdown 文本
  let text = serializer.serialize(processedDoc).trim();

  // 6. 转换为 Y.Doc 状态（用于实时协作）
  const state = convertToState(processedDoc.toJSON() as ProsemirrorData, title);

  return { text, state, title, icon };
}
```

### 2.3 格式转换器

**文件**：`server/utils/DocumentConverter.ts`

#### 转换策略

`DocumentConverter.convert()` 采用**双路转换策略**：

```typescript
public static async convert(
  content: Buffer | string,
  fileName: string,
  mimeType: string
): Promise<ConvertResult> {
  let doc: Node;

  // 策略 A：尝试转换为 HTML（适用于 Docx、HTML 等）
  const html = await this.convertToHtml(content, fileName, mimeType);
  if (html !== undefined) {
    doc = this.htmlToProsemirror(html);
  } else {
    // 策略 B：转换为 Markdown（适用于 Markdown、CSV 等）
    const markdown = await this.convertToMarkdown(
      content,
      fileName,
      mimeType
    );
    doc = ProsemirrorHelper.toProsemirror(markdown);
  }

  // 后处理：提取标题、图标
  // ...
}
```

### 2.4 Markdown 导入详细流程

#### 流程图

```
Markdown 文件 → 读取为字符串 → Frontmatter 处理 → Prosemirror 解析 → 后处理
```

#### 关键步骤

**1. Frontmatter 处理** (`DocumentConverter.ts:444-469`):
```typescript
private static processFrontmatter(content: string): string {
  // 匹配以 --- 开头和结尾的 YAML frontmatter
  const frontmatterRegex = /^---\n([\s\S]*?)\n---(?:\n|$)/;
  const match = content.match(frontmatterRegex);

  if (!match) {
    return content;
  }

  // 验证 YAML 语法
  try {
    yaml.load(frontmatterContent);
  } catch {
    return content; // 无效 YAML，原样返回
  }

  // 转换为 YAML 代码块
  const yamlCodeblock = `\`\`\`yaml\n${frontmatterContent}\n\`\`\`\n\n`;
  return yamlCodeblock + remainingContent;
}
```

**2. Markdown → Prosemirror**:
通过 `ProsemirrorHelper.toProsemirror(markdown)` 转换。

### 2.5 Docx 导入详细流程

#### 流程图

```
Docx 文件 → Mammoth 转换 → HTML → JSDOM 解析 → 预处理 → Prosemirror 转换
```

#### 关键步骤

**1. Docx → HTML** (`DocumentConverter.ts:260-270`):
```typescript
private static async docxToHtml(content: Buffer | string): Promise<string> {
  if (content instanceof Buffer) {
    // 使用 mammoth 库转换
    const { value } = await traceFunction({ spanName: "convertToHtml" })(
      mammoth.convertToHtml
    )({
      buffer: content,
    });
    return value;
  }
  throw FileImportError("Unsupported Word file");
}
```

**2. HTML → Prosemirror** (`DocumentConverter.ts:85-111`):
```typescript
public static htmlToProsemirror(content: Buffer | string): Node {
  const dom = new JSDOM(content);
  const document = dom.window.document;

  // 移除不需要的元素
  const elementsToRemove = document.querySelectorAll(
    "script, style, title, head, meta, link"
  );
  elementsToRemove.forEach((el) => el.remove());

  // 预处理 HTML（处理图片等）
  this.preprocessHtmlForImport(document);

  // 使用 Prosemirror DOMParser 解析
  const domParser = ProsemirrorDOMParser.fromSchema(schema);
  return domParser.parse(document.body);
}
```

**3. HTML 预处理** (`DocumentConverter.ts:119-175`):
- 过滤表情符号图片
- 移除 Jira 图标
- 处理 Confluence 图片尺寸
- 从 data URI 提取图片尺寸

#### 特殊处理：Confluence Word 导出

Confluence 导出的 "Word" 文件实际上是**多部分邮件消息**：

```typescript
private static async confluenceToHtml(
  content: Buffer | string
): Promise<string> {
  // 检测是否为 Confluence 格式
  if (!content.includes("Content-Type: multipart/related")) {
    throw FileImportError("Unsupported Word file");
  }

  // 使用 mailparser 解析邮件格式
  const parsed = await simpleParser(content);
  
  // 替换附件引用为 data URI
  for (const attachment of parsed.attachments) {
    // ... 替换逻辑
  }

  return html;
}
```

### 2.6 支持的格式总结

| 格式 | MIME 类型 | 转换策略 | 依赖库 |
|------|----------|---------|--------|
| **Markdown** | `text/markdown`, `text/plain` | 直接解析 | 内部解析器 |
| **Docx** | `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | Docx → HTML → Prosemirror | `mammoth` |
| **HTML** | `text/html` | HTML → Prosemirror | `jsdom` |
| **CSV** | `text/csv` | CSV → Markdown → Prosemirror | `@fast-csv/parse` |
| **Confluence Word** | `application/msword` | 邮件解析 → HTML → Prosemirror | `mailparser` |

---

## 3. 路径二：Notion API 导入（完整链路）

### 3.1 整体架构

Notion 导入采用**事件驱动 + 异步任务队列**的架构，支持：
- 递归拉取子页面和子数据库
- 异步下载附件
- 层次化存储（Collection + Document）

#### 核心组件

| 组件 | 文件 | 职责 |
|------|------|------|
| **NotionImportsProcessor** | `plugins/notion/server/processors/NotionImportsProcessor.ts` | 事件监听、任务调度、最终持久化 |
| **NotionAPIImportTask** | `plugins/notion/server/tasks/NotionAPIImportTask.ts` | API 数据拉取、格式转换、附件处理 |
| **NotionClient** | `plugins/notion/server/notion.ts` | Notion API 封装、速率限制、自动重试 |
| **NotionConverter** | `plugins/notion/server/utils/NotionConverter.ts` | Notion Block → Prosemirror 映射 |

#### 完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           阶段 1: 导入初始化                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  用户授权 OAuth → 创建 Import 记录 → 触发 imports.create 事件                │
│                                                                              │
│  NotionImportsProcessor.onCreation()                                        │
│    ├── buildTasksInput()                                                     │
│    │   └── NotionClient.fetchRootPages()  ← 获取用户根页面/数据库          │
│    ├── 创建 ImportTask（每 3 个页面一个任务）                               │
│    └── 调度第一个 NotionAPIImportTask                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                           阶段 2: 任务执行（循环）                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  NotionAPIImportTask.perform()                                               │
│    ├── onProcess()                                                            │
│    │   ├── process()  ← 由 NotionAPIImportTask 实现                         │
│    │   │   └── processPage()                                                 │
│    │   │       ├── 页面类型: fetchPage() → NotionConverter.page()          │
│    │   │       ├── 数据库类型: fetchDatabase() → 空内容 + 子页面列表        │
│    │   │       └── parseChildPages()  ← 递归提取子页面/数据库               │
│    │   ├── uploadAttachments()  ← 处理图片/视频/附件                        │
│    │   │   ├── 提取 Prosemirror 中的媒体节点                                │
│    │   │   ├── 创建 Attachment 记录                                          │
│    │   │   ├── 调度 UploadAttachmentsForImportTask（异步下载）              │
│    │   │   └── replaceAttachmentUrls()  ← 替换为内部 URL                    │
│    │   ├── 保存 ImportTask.output                                            │
│    │   ├── 创建子页面的 ImportTask                                           │
│    │   └── scheduleNextTask()  ← 调度下一个任务                              │
│    └── 循环直到所有任务完成                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                           阶段 3: 数据持久化                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  所有任务完成 → 触发 imports.processed 事件                                  │
│                                                                              │
│  NotionImportsProcessor.onProcessed()                                        │
│    └── createCollectionsAndDocuments()                                       │
│        ├── 遍历所有 ImportTask.output                                        │
│        ├── 维护 idMap: Notion externalId → Outline internalId              │
│        ├── updateMentionsAndAttachments()                                   │
│        │   ├── 转换 @mention 的 modelId 为内部 ID                           │
│        │   └── 更新附件 size 属性                                            │
│        ├── 根页面 → 创建 Collection                                          │
│        ├── 子页面 → 创建 Document（处理父子关系）                            │
│        ├── 更新 Collection 文档结构                                          │
│        └── 标记 Import 为 Completed                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 阶段 1：导入初始化详解

#### 3.2.1 事件处理器

**文件**：`server/queues/processors/ImportsProcessor.ts`

**事件监听** (`ImportsProcessor.ts:40-44`):
```typescript
static applicableEvents: Event["name"][] = [
  "imports.create",    // 导入创建时
  "imports.processed", // 所有任务完成时
  "imports.delete",    // 导入删除时
];
```

#### 3.2.2 Notion 处理器注册

**文件**：`plugins/notion/server/processors/NotionImportsProcessor.ts`

```typescript
export class NotionImportsProcessor extends ImportsProcessor<IntegrationService.Notion> {
  protected canProcess(
    importModel: Import<IntegrationService.Notion>
  ): boolean {
    return importModel.service === IntegrationService.Notion;
  }
}
```

#### 3.2.3 构建任务输入

**文件**：`plugins/notion/server/processors/NotionImportsProcessor.ts:29-59`

```typescript
protected async buildTasksInput(
  importModel: Import<IntegrationService.Notion>,
  transaction: Transaction
): Promise<NotionImportTaskInput> {
  // 1. 获取 OAuth 认证信息
  const integration = await Integration.scope("withAuthentication").findByPk(
    importModel.integrationId,
    { rejectOnEmpty: true }
  );

  // 2. 创建 Notion API 客户端
  const notion = new NotionClient(integration.authentication.token);

  // 3. 拉取用户工作区的根页面/数据库
  const rootPages = await notion.fetchRootPages();

  // 4. 更新 Import 的 input 字段
  const importInput: NotionImportInput = rootPages.map((page) => ({
    type: page.type,
    externalId: page.id,
    permission: defaultPermission,
  }));
  importModel.input = importInput;
  await importModel.save({ transaction });

  // 5. 返回任务输入（用于创建 ImportTask）
  return rootPages.map((page) => ({
    type: page.type,
    externalId: page.id,
  }));
}
```

### 3.3 阶段 2：任务执行详解

#### 3.3.1 Notion API 客户端

**文件**：`plugins/notion/server/notion.ts`

**核心特性**：
- **双层速率限制**：实现默认限流 + 平台硬限制处理
- **自动重试**：超时和速率限制错误
- **递归获取**：自动获取 block 的子节点

##### 实现默认限流（客户端主动限流）

这是在客户端层面主动控制请求速率，**避免触发** Notion 平台的限流。

**配置** (`notion.ts:71-88`):
```typescript
constructor(
  accessToken: string,
  rateLimit: { window: number; limit: number } = {
    window: Second.ms,  // 1 秒（时间窗口）
    limit: 3,           // 3 次请求（窗口内最大请求数）
  },
  // ...
) {
  this.client = new Client({ auth: accessToken });
  this.limiter = RateLimit(rateLimit.limit, {
    timeUnit: rateLimit.window,
    uniformDistribution: true,  // 均匀分布，避免突发请求
  });
  // ...
}
```

**限流机制**：
- 使用 `async-sema` 库的 `RateLimit`
- **默认配置**：每秒 3 次请求（`window: 1000ms, limit: 3`）
- **均匀分布** (`uniformDistribution: true`)：在 1 秒内均匀分配 3 次请求，避免突发流量
- **使用方式**：每次 API 调用前调用 `await this.limiter()` 等待令牌

**特点**：
- 这是**主动限流**，在客户端控制请求速率
- 目的是**避免触发** Notion 平台的硬限制
- 配置是**代码硬编码**的默认值

##### 平台硬限制（服务端被动限流）

当客户端限流失效或请求量过大时，Notion 服务端会返回 **429 Too Many Requests** 错误，这就是**平台硬限制**。

**错误处理** (`notion.ts:125-151`):
```typescript
// 检查是否为速率限制错误
if (
  error instanceof APIResponseError &&
  error.code === APIErrorCode.RateLimited  // Notion API 返回的限流错误码
) {
  if (retries < this.maxRetries) {
    retries++;
    const headers = error.headers as Record<string, string>;
    
    // 读取 Retry-After header 获取需要等待的时间
    const retryAfter = headers["Retry-After"]
      ? parseInt(headers["Retry-After"], 10) * 1000  // 转换为毫秒
      : undefined;
    
    // 优先使用服务端建议的等待时间，否则使用指数退避
    const delay = retryAfter ?? this.retryDelay * retries;
    
    Logger.info(
      "task",
      `Notion API rate limit hit, retrying in ${delay}ms (retry ${retries}/${this.maxRetries})`
    );

    // 等待后重试
    await new Promise((resolve) => setTimeout(resolve, delay));
    continue;
  }
  
  // 超过最大重试次数
  Logger.warn(
    `Notion API rate limit exceeded after ${this.maxRetries} retries`,
    { error: error.message }
  );
}
```

**特点**：
- 这是**被动限流**，当 Notion 服务端返回 `APIErrorCode.RateLimited` 时触发
- 读取 `Retry-After` header 获取服务端建议的等待时间
- 如果没有 `Retry-After`，使用**指数退避**策略（`retryDelay * retries`）
- 最多重试 **3 次**

##### 两种限流的对比

| 特性 | 实现默认限流 | 平台硬限制 |
|------|-------------|-----------|
| **层面** | 客户端主动控制 | 服务端被动返回 |
| **触发时机** | 每次 API 调用前（等待令牌） | 收到服务端 429 响应后 |
| **配置来源** | 代码硬编码（默认值） | Notion 服务端动态返回 |
| **等待策略** | 均匀分布（RateLimit 库） | 指数退避 + Retry-After header |
| **目的** | 避免触发限流 | 处理已触发的限流 |
| **重试机制** | 无（只是等待） | 最多 3 次重试 |
| **错误码** | 无 | `APIErrorCode.RateLimited` |

**带重试的 API 调用完整流程** (`notion.ts:96-157`):
```typescript
private async fetchWithRetry<T>(apiCall: () => Promise<T>): Promise<T> {
  let retries = 0;

  while (true) {
    try {
      // 第一层：实现默认限流 - 主动等待令牌
      await this.limiter();
      return await apiCall();
    } catch (error) {
      // 处理超时错误
      if (error instanceof RequestTimeoutError) {
        if (retries < this.maxRetries) {
          retries++;
          await new Promise((resolve) => setTimeout(resolve, this.retryDelay * retries));
          continue;
        }
      }
      
      // 第二层：平台硬限制 - 被动处理服务端返回的限流
      if (error instanceof APIResponseError && 
          error.code === APIErrorCode.RateLimited) {
        if (retries < this.maxRetries) {
          retries++;
          // 读取 Retry-After header
          const headers = error.headers as Record<string, string>;
          const retryAfter = headers["Retry-After"]
            ? parseInt(headers["Retry-After"], 10) * 1000
            : undefined;
          const delay = retryAfter ?? this.retryDelay * retries;
          
          await new Promise((resolve) => setTimeout(resolve, delay));
          continue;
        }
      }
      
      throw error;
    }
  }
}
```

**拉取页面及递归获取 Blocks** (`notion.ts:218-225`):
```typescript
async fetchPage(
  pageId: string,
  { titleMaxLength }: { titleMaxLength: number }
) {
  const pageInfo = await this.fetchPageInfo(pageId, { titleMaxLength });
  const blocks = await this.fetchBlockChildren(pageId);  // 递归获取所有 blocks
  return { ...pageInfo, blocks };
}
```

**递归获取 Block 子节点** (`notion.ts:238-285`):
```typescript
private async fetchBlockChildren(blockId: string) {
  const blocks: Block[] = [];
  let cursor: string | undefined;
  let hasMore = true;

  // 分页拉取
  while (hasMore) {
    const response = await this.fetchWithRetry(() =>
      this.client.blocks.children.list({
        block_id: blockId,
        start_cursor: cursor,
        page_size: this.pageSize,  // 100
      })
    );
    blocks.push(...(response.results as BlockObjectResponse[]));
    hasMore = response.has_more;
    cursor = response.next_cursor ?? undefined;
  }

  // 递归获取子 block（排除 child_page、child_database 等）
  await Promise.all(
    blocks.map(async (block) => {
      if (
        block.has_children &&
        !this.skipChildrenForBlock.includes(block.type)  // ["unsupported", "child_page", "child_database"]
      ) {
        block.children = await this.fetchBlockChildren(block.id);
      }
    })
  );

  return blocks;
}
```

#### 3.3.2 页面处理

**文件**：`plugins/notion/server/tasks/NotionAPIImportTask.ts:99-175`

```typescript
private async processPage({
  item,
  client,
}: {
  item: ImportTaskInput<IntegrationService.Notion>[number];
  client: NotionClient;
}): Promise<ParsePageOutput | null> {
  try {
    // 处理数据库类型
    if (item.type === PageType.Database) {
      const { pages, ...databaseInfo } = await client.fetchDatabase(
        item.externalId,
        { titleMaxLength }
      );

      return {
        ...databaseInfo,
        externalId: item.externalId,
        content: ProsemirrorHelper.getEmptyDocument() as ProsemirrorDoc,
        children: pages.map((page) => ({
          type: page.type,
          externalId: page.id,
        })),
      };
    }

    // 处理页面类型
    const { blocks, ...pageInfo } = await client.fetchPage(item.externalId, {
      titleMaxLength,
    });

    return {
      ...pageInfo,
      externalId: item.externalId,
      content: NotionConverter.page({ children: blocks } as NotionPage),
      children: this.parseChildPages(blocks),  // 提取子页面
    };
  } catch (error) {
    // 处理 Not found、Unauthorized 等可跳过错误
    if (error instanceof APIResponseError) {
      if (
        error.code === APIErrorCode.ObjectNotFound ||
        error.code === APIErrorCode.Unauthorized ||
        // ... 其他可跳过错误
      ) {
        Logger.warn(`Skipping Notion page ${item.externalId}`);
        return null;  // 跳过，不中断整个导入
      }
    }
    throw error;
  }
}
```

#### 3.3.3 子页面递归提取

**文件**：`plugins/notion/server/tasks/NotionAPIImportTask.ts:183-197`

```typescript
private parseChildPages(pageBlocks: Block[]): ChildPage[] {
  const childPages: ChildPage[] = [];

  pageBlocks.forEach((block) => {
    // 直接子页面
    if (block.type === "child_page") {
      childPages.push({ type: PageType.Page, externalId: block.id });
    } 
    // 直接子数据库
    else if (block.type === "child_database") {
      childPages.push({ type: PageType.Database, externalId: block.id });
    } 
    // 递归查找嵌套的子页面
    else if (block.children?.length) {
      childPages.push(...this.parseChildPages(block.children));
    }
  });

  return childPages;
}
```

#### 3.3.4 Notion Block → Prosemirror 转换

**文件**：`plugins/notion/server/utils/NotionConverter.ts`

**核心设计**：Notion 的 Block 结构与 Prosemirror 的节点结构相似，采用**直接映射**策略。

**主转换入口** (`NotionConverter.ts:54-59`):
```typescript
public static page(item: NotionPage): ProsemirrorDoc {
  return {
    type: "doc",
    content: this.mapChildren(item),
  };
}
```

**子节点映射** (`NotionConverter.ts:61-165`):

```typescript
private static mapChildren(item: Block | NotionPage) {
  const mapChild = (child: Block): ProsemirrorData | ProsemirrorData[] | undefined => {
    // 子页面和子数据库单独处理
    if (child.type === "child_page") {
      return this.child_page(child);  // 转换为 @mention
    }
    if (child.type === "child_database") {
      return this.child_database(child);  // 转换为 @mention
    }

    // 其他 Block 类型直接调用对应方法
    if (this[child.type]) {
      const response = this[child.type](child);

      // 处理可切换标题（toggleable heading）
      const canToggle = child[child.type].is_toggleable === true;
      if (canToggle) {
        return {
          type: "container_toggle",
          attrs: { id: randomUUID() },
          content: [response, ...this.mapChildren(child)],
        };
      }

      // 处理无子块容器的节点（如 paragraph）
      if (
        response &&
        this.nodesWithoutBlockChildren.includes(response.type) &&  // ["paragraph"]
        "children" in child
      ) {
        return [response, ...this.mapChildren(child)];  // 子块平铺
      }
      return response;
    }

    Logger.warn("Encountered unknown Notion block", child);
    return undefined;
  };

  // 列表包装逻辑：Notion 的列表项是扁平的，需要包装到列表容器中
  let wrappingList;
  const children = [] as ProsemirrorData[];

  for (const child of item.children) {
    const mapped = mapChild(child);
    if (!mapped) continue;

    // 有序列表
    if (child.type === "numbered_list_item") {
      if (!wrappingList) {
        wrappingList = { type: "ordered_list", content: [] };
      }
      wrappingList.content.push(...(isArray(mapped) ? mapped : [mapped]));
      continue;
    }

    // 无序列表
    if (child.type === "bulleted_list_item") {
      if (!wrappingList) {
        wrappingList = { type: "bullet_list", content: [] };
      }
      wrappingList.content.push(...(isArray(mapped) ? mapped : [mapped]));
      continue;
    }

    // 待办事项
    if (child.type === "to_do") {
      if (!wrappingList) {
        wrappingList = { type: "checkbox_list", content: [] };
      }
      wrappingList.content.push(...(isArray(mapped) ? mapped : [mapped]));
      continue;
    }

    // 非列表项：结束当前列表
    if (wrappingList) {
      children.push(wrappingList);
      wrappingList = undefined;
    }
    children.push(...(isArray(mapped) ? mapped : [mapped]));
  }

  if (wrappingList) {
    children.push(wrappingList);
  }

  return children;
}
```

**富文本转换** (`NotionConverter.ts:282-389`):

```typescript
private static rich_text = (item: RichTextItemResponse) => {
  // Notion annotation → Prosemirror mark 映射
  const annotationToMark: Record<
    keyof RichTextItemResponse["annotations"],
    string
  > = {
    bold: "strong",
    code: "code_inline",
    italic: "em",
    underline: "underline",
    strikethrough: "strikethrough",
    color: "highlight",
  };

  // 处理 @mention（页面引用）
  if (item.type === "mention") {
    if (item.mention.type === "page") {
      return {
        type: "mention",
        attrs: {
          type: MentionType.Document,
          label: item.plain_text,
          modelId: item.mention.page.id,  // 暂时用 Notion ID，后续替换
        },
      };
    }
    // ... 其他 mention 类型
  }

  // 处理公式
  if (item.type === "equation") {
    return {
      type: "math_inline",
      content: [{ type: "text", text: item.equation.expression }],
    };
  }

  // 普通文本
  if (item.text.content) {
    return {
      type: "text",
      text: item.text.content,
      marks: [
        ...mapAttrs(),  // 应用 annotations
        ...(item.text.link
          ? [{ type: "link", attrs: { href: item.text.link.url } }]
          : []),
      ].filter(Boolean),
    };
  }

  return undefined;
};
```

**支持的 Block 类型映射**：

| Notion Block | Outline Prosemirror | 说明 |
|-------------|---------------------|------|
| `paragraph` | `paragraph` | 普通段落 |
| `heading_1/2/3` | `heading` (level 1/2/3) | 标题 |
| `bulleted_list_item` | `list_item` in `bullet_list` | 无序列表 |
| `numbered_list_item` | `list_item` in `ordered_list` | 有序列表 |
| `to_do` | `checkbox_item` in `checkbox_list` | 待办事项 |
| `quote` | `blockquote` | 引用块 |
| `code` | `code_fence` | 代码块 |
| `callout` | `container_notice` | 提示块（带颜色） |
| `toggle` | `container_toggle` | 折叠块 |
| `table` | `table` | 表格 |
| `image` | `image` | 图片 |
| `video` | `video` 或 `embed` | 视频 |
| `file` | `attachment` | 附件 |
| `pdf` | `attachment` | PDF |
| `bookmark` | 链接文本 | 书签 |
| `divider` | `hr` | 分割线 |
| `equation` | `math_block` | 公式块 |
| `embed` | `embed` | 嵌入内容 |
| `child_page` | `mention` (Document) | 子页面引用 |
| `child_database` | `mention` (Document) | 子数据库引用 |
| `link_to_page` | `mention` (Document) | 页面链接 |
| `column_list` | 平铺子节点 | 列列表 |
| `column` | 平铺子节点 | 列 |
| `synced_block` | 平铺子节点 | 同步块 |
| `breadcrumb` | 忽略 | 面包屑 |
| `table_of_contents` | 忽略 | 目录 |

#### 3.3.5 附件处理

**文件**：`server/queues/tasks/APIImportTask.ts:253-388`

**附件上传流程**：

```typescript
private async uploadAttachments({
  doc,
  externalId,
  createdBy,
}: {
  doc: ProsemirrorDoc;
  externalId: string;
  createdBy: User;
}): Promise<ProsemirrorDoc> {
  const docNode = ProsemirrorHelper.toProsemirror(doc);
  
  // 1. 提取所有媒体节点
  const nodes = [
    ...ProsemirrorHelper.getImages(docNode),
    ...ProsemirrorHelper.getVideos(docNode),
    ...ProsemirrorHelper.getAttachments(docNode),
  ];

  if (!nodes.length) {
    return doc;
  }

  // 2. 去重 URL
  const attachmentsData = uniqBy(
    nodes.map((node) => {
      const url = String(
        node.type.name === "attachment" ? node.attrs.href : node.attrs.src
      );
      const name = String(
        node.type.name === "image" ? node.attrs.alt : node.attrs.title
      ).trim();
      return { url, name: name.length !== 0 ? name : node.type.name };
    }),
    "url"
  );

  const urlToAttachment: Record<string, Attachment> = {};

  // 3. 创建 Attachment 记录（先不下载文件）
  await sequelize.transaction(async (transaction) => {
    const dbPromises = attachmentsData.map(async (item) => {
      const modelId = randomUUID();
      const acl = AttachmentHelper.presetToAcl(
        AttachmentPreset.DocumentAttachment
      );
      const key = AttachmentHelper.getKey({
        id: modelId,
        name: item.name,
        userId: createdBy.id,
      });

      const attachment = await Attachment.create(
        {
          id: modelId,
          key,
          acl,
          size: 0,  // 大小暂时为 0，下载后更新
          expiresAt: AttachmentHelper.presetToExpiry(
            AttachmentPreset.DocumentAttachment
          ),
          contentType: "application/octet-stream",
          documentId: externalId,  // 暂时用 Notion ID
          teamId: createdBy.teamId,
          userId: createdBy.id,
        },
        { transaction }
      );

      urlToAttachment[item.url] = attachment;
    });

    return await Promise.all(dbPromises);
  });

  // 4. 调度异步下载任务（不阻塞导入流程）
  try {
    const uploadItems = Object.entries(urlToAttachment).map(
      ([url, attachment]) => ({ attachmentId: attachment.id, url })
    );
    await new UploadAttachmentsForImportTask().schedule(uploadItems);
  } catch (err) {
    // 下载失败不中断导入
    Logger.error(`upload attachment task failed for externalId ${externalId}`, err);
  }

  // 5. 替换 Prosemirror 中的外部 URL 为内部 redirect URL
  return this.replaceAttachmentUrls(docNode, urlToAttachment).toJSON();
}
```

**URL 替换** (`APIImportTask.ts:348-388`):
```typescript
private replaceAttachmentUrls(
  doc: Node,
  urlToAttachment: Record<string, Attachment>
): Node {
  const attachmentTypes = ["attachment", "image", "video"];

  const transformAttachmentNode = (node: Node): Node => {
    const json = node.toJSON() as ProsemirrorData;
    const attrs = json.attrs ?? {};

    if (node.type.name === "attachment") {
      // attachment 使用 href 属性
      const attachmentModel = urlToAttachment[attrs.href as string];
      attrs.href = attachmentModel.redirectUrl;
      attrs.id = attachmentModel.id;
    } else if (node.type.name === "image" || node.type.name === "video") {
      // image/video 使用 src 属性
      attrs.src = urlToAttachment[attrs.src as string].redirectUrl;
    }

    json.attrs = attrs;
    return Node.fromJSON(schema, json);
  };

  // 递归遍历替换
  const transformFragment = (fragment: Fragment): Fragment => {
    const nodes: Node[] = [];
    fragment.forEach((node) => {
      nodes.push(
        attachmentTypes.includes(node.type.name)
          ? transformAttachmentNode(node)
          : node.copy(transformFragment(node.content))
      );
    });
    return Fragment.fromArray(nodes);
  };

  return doc.copy(transformFragment(doc.content));
}
```

#### 3.3.6 任务调度

**文件**：`server/queues/tasks/APIImportTask.ts:195-224`

```typescript
private async onCompletion(importTask: ImportTask<T>) {
  // 查找是否还有待处理的任务
  const where: WhereOptions<ImportTask<T>> = {
    state: ImportTaskState.Created,
    importId: importTask.importId,
  };

  const nextImportTask = await ImportTask.findOne<ImportTask<T>>({
    where,
    order: [["createdAt", "ASC"]],
  });

  // 有任务则继续调度
  if (nextImportTask) {
    return await this.scheduleNextTask(nextImportTask);
  }

  // 所有任务完成，标记为 Processed，触发 imports.processed 事件
  await sequelize.transaction(async (transaction) => {
    const associatedImport = importTask.import;
    associatedImport.state = ImportState.Processed;
    await associatedImport.saveWithCtx(
      createContext({
        user: associatedImport.createdBy,
        transaction,
      }),
      undefined,
      { name: "processed" }  // 触发事件
    );
  });
}
```

### 3.4 阶段 3：数据持久化详解

#### 3.4.1 持久化入口

**文件**：`server/queues/processors/ImportsProcessor.ts:158-213`

```typescript
private async onProcessed(importModel: Import<T>, transaction: Transaction) {
  try {
    // 创建 Collection 和 Document
    const { collections } = await this.createCollectionsAndDocuments({
      importModel,
      transaction,
    });

    // 更新 Collection 的文档结构
    for (const collection of collections) {
      await Document.unscoped().findAllInBatches<Document>(
        {
          where: { parentDocumentId: null, collectionId: collection.id },
          // ...
        },
        async (documents) => {
          for (const document of documents) {
            await collection.addDocumentToStructure(document, undefined, {
              save: false,
              silent: true,
              transaction,
              insertOrder: "append",
            });
          }
        }
      );
      await collection.save({ silent: true, transaction });
    }

    // 标记导入完成
    importModel.state = ImportState.Completed;
    await importModel.saveWithCtx(
      createContext({ user: importModel.createdBy, transaction })
    );
  } catch (err) {
    // ... 错误处理
  }
}
```

#### 3.4.2 创建 Collection 和 Document

**文件**：`server/queues/processors/ImportsProcessor.ts:265-472`

```typescript
private async createCollectionsAndDocuments({
  importModel,
  transaction,
}: {
  importModel: Import<T>;
  transaction: Transaction;
}): Promise<{ collections: Collection[] }> {
  const createdCollections: Collection[] = [];
  // 外部 ID → 内部 ID 映射
  const idMap: Record<string, string> = {};
  // 根页面（将创建为 Collection）
  const importInput = keyBy(importModel.input, "externalId");
  const ctx = createContext({ user: importModel.createdBy, transaction });

  // 分批处理 ImportTask
  await ImportTask.findAllInBatches<ImportTask<T>>(
    {
      where: { importId: importModel.id },
      order: [["createdAt", "ASC"], ["id", "ASC"]],
      batchLimit: 5,  // 输出数据可能很大，分批处理
      transaction,
    },
    async (importTasks) => {
      for (const importTask of importTasks) {
        const outputMap = keyBy(importTask.output ?? [], "externalId");

        for (const input of importTask.input) {
          const externalId = input.externalId;
          const internalId = await this.getInternalId(externalId, idMap);
          const parentExternalId = input.parentExternalId;
          const parentInternalId = parentExternalId
            ? await this.getInternalId(parentExternalId, idMap)
            : undefined;
          const collectionExternalId = input.collectionExternalId;
          const collectionInternalId = collectionExternalId
            ? await this.getInternalId(collectionExternalId, idMap)
            : undefined;

          const output = outputMap[externalId];
          if (!output) continue;  // 跳过无输出的项

          // 查找该页面的附件
          const attachments = await Attachment.findAll({
            attributes: ["id", "size"],
            where: { documentId: externalId },
            transaction,
          });

          // 更新 @mention 和附件
          const transformedContent = await this.updateMentionsAndAttachments({
            content: output.content,
            attachments,
            importInput,
            idMap,
            actorId: importModel.createdById,
            teamId: importModel.teamId,
          });

          // ========== 根页面 → 创建为 Collection ==========
          const collectionItem = importInput[externalId];
          if (collectionItem) {
            const description = await DocumentHelper.toMarkdown(
              transformedContent,
              { includeTitle: false }
            );

            const sourceMetadata: SourceMetadata = {
              externalId: externalId,
              externalName: output.title,
              createdByName: output.author,
            };

            const collection = Collection.build({
              id: internalId,
              name: output.title,
              icon: output.emoji ?? "collection",
              color: output.emoji ? undefined : randomElement(colorPalette),
              content: transformedContent,
              description: truncate(description, {
                length: CollectionValidation.maxDescriptionLength,
              }),
              createdById: importModel.createdById,
              teamId: importModel.createdBy.teamId,
              apiImportId: importModel.id,
              index: collectionIdx,
              sort: Collection.DEFAULT_SORT,
              permission: collectionItem.permission,
              sourceMetadata,
              createdAt: output.createdAt ?? now,
              updatedAt: output.updatedAt ?? now,
            });

            await collection.saveWithCtx(ctx, { silent: true }, { name: "create" });
            createdCollections.push(collection);

            // 根页面的附件不需要 documentId
            await Attachment.update(
              { documentId: null },
              { where: { documentId: externalId }, silent: true, transaction }
            );

            continue;
          }

          // ========== 子页面 → 创建为 Document ==========
          const isRootDocument =
            !parentExternalId || !!importInput[parentExternalId];

          const defaults = {
            title: output.title,
            icon: output.emoji,
            content: transformedContent,
            text: await DocumentHelper.toMarkdown(transformedContent, {
              includeTitle: false,
            }),
            collectionId: collectionInternalId,
            parentDocumentId: isRootDocument ? undefined : parentInternalId,
            createdById: importModel.createdById,
            lastModifiedById: importModel.createdById,
            teamId: importModel.createdBy.teamId,
            apiImportId: importModel.id,
            sourceMetadata: {
              externalId,
              externalName: output.title,
              createdByName: output.author,
            },
            createdAt: output.createdAt ?? now,
            updatedAt: output.updatedAt ?? now,
            publishedAt: output.updatedAt ?? output.createdAt ?? now,
          };

          await Document.findOrCreateWithCtx(
            ctx,
            {
              where: { id: internalId },
              defaults,
              silent: true,
            },
            {
              name: "create",
              data: { title: output.title, source: "import" },
            }
          );

          // 更新附件的 documentId 为内部 ID
          await Attachment.update(
            { documentId: internalId },
            { where: { documentId: externalId }, silent: true, transaction }
          );
        }
      }
    }
  );

  return { collections: createdCollections };
}
```

#### 3.4.3 更新 @mention 和附件

**文件**：`server/queues/processors/ImportsProcessor.ts:484-559`

```typescript
private async updateMentionsAndAttachments({
  content,
  attachments,
  idMap,
  importInput,
  actorId,
  teamId,
}: {
  content: ProsemirrorDoc;
  attachments: Attachment[];
  idMap: Record<string, string>;
  importInput: Record<string, ImportInput<any>[number]>;
  actorId: string;
  teamId: string;
}): Promise<ProsemirrorDoc> {
  if (!content.content.length) {
    return content;
  }

  const attachmentsMap = keyBy(attachments, "id");
  const doc = ProsemirrorHelper.toProsemirror(content);

  // 转换 @mention
  const transformMentionNode = async (node: Node): Promise<Node> => {
    const json = node.toJSON() as ProsemirrorData;
    const attrs = json.attrs ?? {};

    attrs.id = randomUUID();
    attrs.actorId = actorId;

    // 将 Notion 外部 ID 转换为 Outline 内部 ID
    const externalId = attrs.modelId as string;
    attrs.modelId = await this.getInternalId(externalId, idMap, teamId);

    // 区分是引用 Collection 还是 Document
    const isCollectionMention = !!importInput[externalId];
    attrs.type = isCollectionMention
      ? MentionType.Collection
      : MentionType.Document;

    json.attrs = attrs;
    return Node.fromJSON(schema, json);
  };

  // 更新附件 size
  const transformAttachmentNode = (node: Node): Node => {
    const json = node.toJSON() as ProsemirrorData;
    const attrs = json.attrs ?? {};

    // 附件下载后会更新 size，这里回填
    attrs.size = attachmentsMap[attrs.id as string]?.size;

    json.attrs = attrs;
    return Node.fromJSON(schema, json);
  };

  // 递归遍历转换
  const transformFragment = async (fragment: Fragment): Promise<Fragment> => {
    const nodePromises: Promise<Node>[] = [];

    fragment.forEach((node) => {
      if (node.type.name === "mention") {
        nodePromises.push(transformMentionNode(node));
      } else if (node.type.name === "attachment") {
        nodePromises.push(Promise.resolve(transformAttachmentNode(node)));
      } else {
        nodePromises.push(
          transformFragment(node.content).then((transformedContent) =>
            node.copy(transformedContent)
          )
        );
      }
    });

    const nodes = await Promise.all(nodePromises);
    return Fragment.fromArray(nodes);
  };

  return doc.copy(await transformFragment(doc.content)).toJSON();
}
```

#### 3.4.4 ID 映射策略

**文件**：`server/queues/processors/ImportsProcessor.ts:570-597`

```typescript
private async getInternalId(
  externalId: string,
  idMap: Record<string, string>,
  teamId?: string
) {
  let internalId = idMap[externalId];

  if (!internalId && teamId) {
    // 检查是否已存在（用于重新导入场景）
    const existingId = (
      await Document.findOne({
        attributes: ["id"],
        where: {
          teamId,
          sourceMetadata: {
            externalId,  // 通过 sourceMetadata 匹配
          },
        },
      })
    )?.id;

    if (existingId) {
      return existingId;
    }
  }

  // 生成新 ID
  idMap[externalId] = internalId ?? randomUUID();
  return idMap[externalId];
}
```

---

## 4. 两条路径对比

| 特性 | 文件上传导入（Markdown/Docx） | Notion API 导入 |
|------|-------------------------------|-----------------|
| **数据来源** | 本地文件 | Notion API |
| **触发方式** | 同步 API 调用 | 事件驱动 + 异步任务 |
| **架构模式** | Command 模式 | Processor + Task 模式 |
| **递归处理** | 不支持（单文件） | 自动递归子页面/数据库 |
| **附件处理** | 同步处理 | 异步下载任务 |
| **存储结构** | 单个 Document | Collection + Document 层次结构 |
| **@mention 处理** | 无 | 外部 ID → 内部 ID 转换 |
| **错误处理** | 同步抛出 | 可跳过错误（如页面无权限） |
| **速率限制** | 无 | Notion API 限制（每秒 3 次） |
| **重试机制** | 无 | 超时/速率限制自动重试 |

---

## 5. 关键设计决策

### 5.1 为什么 Notion 导入采用异步任务队列？

1. **API 速率限制**：Notion API 有严格的速率限制（每秒 3 次），大量页面导入需要时间
2. **递归拉取**：子页面、嵌套 block 可能需要多次 API 调用
3. **附件下载**：图片、视频等附件下载可能耗时很长
4. **用户体验**：异步处理允许用户继续操作，后台进度可追踪

### 5.2 为什么使用两层 ID 映射？

1. **第一层（任务阶段）**：Notion externalId → 临时 ID（用于创建 Attachment）
2. **第二层（持久化阶段）**：Notion externalId → Outline internalId（用于 Document/Collection）

**原因**：
- 任务阶段不知道最终的 internalId（可能复用已存在的文档）
- 附件需要提前创建记录以替换 URL
- @mention 需要在所有页面创建完成后才能解析

### 5.3 为什么附件下载是异步的？

1. **不阻塞主流程**：附件下载失败不应导致整个导入失败
2. **可重试**：下载任务可以单独重试
3. **并发控制**：可以限制同时下载的附件数量

### 5.4 为什么 Docx 采用中间格式（HTML）？

1. **成熟的转换库**：`mammoth` 是成熟的 Docx → HTML 转换库
2. **HTML 是通用中间格式**：HTML 到 Prosemirror 的转换已存在（用于其他场景）
3. **减少维护成本**：不需要直接维护 Docx → Prosemirror 的复杂映射

### 5.5 为什么 Notion 采用直接映射？

1. **结构相似**：Notion Block 与 Prosemirror Node 都是树形结构
2. **特性丰富**：Notion 有很多独特特性（callout、toggle、database 等），直接映射能更好保留
3. **API 原生**：Notion API 返回的就是 Block 结构，无需中间转换

---

## 6. 数据库模型

### 6.1 相关表

| 表名 | 用途 | 关键字段 |
|------|------|----------|
| `imports` | 导入记录 | `state`, `service`, `input`, `documentCount` |
| `import_tasks` | 导入任务 | `state`, `input`, `output`, `importId` |
| `collections` | 集合（根页面导入为集合） | `apiImportId`, `sourceMetadata` |
| `documents` | 文档 | `apiImportId`, `sourceMetadata`, `parentDocumentId` |
| `attachments` | 附件 | `documentId`, `redirectUrl` |

### 6.2 状态流转

**Import 状态**：
```
Created → InProgress → Processed → Completed
                ↓
             Errored / Canceled
```

**ImportTask 状态**：
```
Created → InProgress → Completed
                ↓
             Errored / Canceled
```

---

## 7. 错误处理

### 7.1 可跳过的错误（Notion 导入）

在 `NotionAPIImportTask.processPage()` 中，以下错误会被跳过而不中断整个导入：

1. **`ObjectNotFound`**：页面/数据库不存在
2. **`Unauthorized`**：无权限访问
3. **特定错误消息**：
   - "Database retrievals do not support linked databases"
   - "does not contain any data sources accessible by this API bot"
   - "Databases with multiple data sources are not supported in this API version"

### 7.2 自动重试

`NotionClient.fetchWithRetry()` 会自动重试：
1. **超时错误**：`RequestTimeoutError`
2. **速率限制**：`APIErrorCode.RateLimited`（读取 `Retry-After` header）

重试策略：
- 最多 3 次
- 指数退避延迟

### 7.3 附件下载失败

附件下载失败**不中断**导入流程，只是记录日志。

---

## 8. 扩展与维护

### 8.1 添加新的 API 集成导入

参考 Notion 导入的架构，需要实现：

1. **`XxxImportsProcessor`**：继承 `ImportsProcessor`
   - `canProcess()`：判断是否处理该服务
   - `buildTasksInput()`：构建初始任务输入
   - `scheduleTask()`：调度第一个任务

2. **`XxxAPIImportTask`**：继承 `APIImportTask`
   - `process()`：从 API 拉取数据并转换
   - `scheduleNextTask()`：调度下一个任务

3. **`XxxClient`**：封装外部 API 调用
4. **`XxxConverter`**：将外部数据格式转换为 Prosemirror

### 8.2 添加新的文件导入格式

在 `DocumentConverter` 中添加：

1. 在 `convertToHtml()` 或 `convertToMarkdown()` 中添加类型判断
2. 实现对应的转换方法（如 `newFormatToHtml()`）
3. 确保转换结果可以被现有的 HTML/Markdown 解析器处理

---

## 9. 总结

Outline 的文档导入系统设计了**两条独立但相似的路径**：

### 文件上传导入（Markdown、Docx 等）
- **简单直接**：同步处理，适合单文件
- **中间格式策略**：Docx → HTML → Prosemirror
- **即时反馈**：用户立即可知结果

### Notion API 导入
- **复杂但强大**：异步任务队列，支持递归和层次结构
- **直接映射策略**：Notion Block → Prosemirror Node
- **容错设计**：可跳过错误页面，附件异步下载
- **ID 映射**：两层映射处理 @mention 和父子关系

两条路径最终都汇聚到 **Prosemirror** 这一统一内部格式，保证了后续编辑、搜索、导出等功能的一致性。