# Outline 文档导入功能分析

## 1. 概述

Outline 支持从多种格式导入文档，包括 Markdown、Notion 和 docx 等。整个导入系统采用了分层架构，将不同格式的解析、内容转换和数据存储分离，实现了高内聚低耦合的设计。

### 核心组件

- **`documentImporter.ts`**：文档导入的主入口，协调整个导入流程
- **`DocumentConverter.ts`**：通用文档转换器，处理 Markdown、docx、HTML、CSV 等格式
- **`NotionConverter.ts`**：专用 Notion 格式转换器
- **`ProsemirrorHelper.ts`**：Prosemirror 相关的辅助工具
- **`Prosemirror 编辑器`**：内部使用的富文本编辑器格式

## 2. 整体架构

### 导入流程

1. **格式识别**：根据文件扩展名和 MIME 类型识别导入格式
2. **格式解析**：使用对应的解析器解析原始内容
3. **内容转换**：将解析后的内容转换为统一的 Prosemirror 格式
4. **后处理**：提取标题、图标，处理图片等
5. **状态转换**：将 Prosemirror 数据转换为 Y.Doc 状态（用于实时协作）
6. **存储**：将处理后的数据存储到数据库

### 数据流

```
原始文件 → 格式解析器 → 中间格式 → Prosemirror → Y.Doc → 数据库存储
```

## 3. 各格式详细实现

### 3.1 Markdown 导入

#### 解析流程

Markdown 是 Outline 原生支持的格式，解析流程相对直接：

1. **文件读取**：将 Markdown 文件内容读取为字符串
2. **Frontmatter 处理**：识别并处理 YAML frontmatter
3. **Markdown 解析**：转换为 Prosemirror 格式

#### 关键代码

**Frontmatter 处理** (`DocumentConverter.ts:444-469`):
```typescript
private static processFrontmatter(content: string): string {
  // Frontmatter must start at the beginning of the document
  const frontmatterRegex = /^---\n([\s\S]*?)\n---(?:\n|$)/;
  const match = content.match(frontmatterRegex);

  if (!match) {
    return content;
  }

  const frontmatterContent = match[1];
  const remainingContent = content.slice(match[0].length);

  // Validate that the frontmatter is valid YAML
  try {
    yaml.load(frontmatterContent);
  } catch {
    // If it's not valid YAML, return content unchanged
    return content;
  }

  // Convert frontmatter to a YAML codeblock
  const codeBlockDelimiter = "```";
  const yamlCodeblock = `${codeBlockDelimiter}yaml\n${frontmatterContent}\n${codeBlockDelimiter}\n\n`;

  return yamlCodeblock + remainingContent;
}
```

**Markdown 转换为 Prosemirror**：
Markdown 通过 `ProsemirrorHelper.toProsemirror(markdown)` 方法转换为 Prosemirror 格式。

#### 支持的 Markdown 特性

- 标准 Markdown 语法（标题、列表、链接、图片等）
- YAML frontmatter（转换为 YAML 代码块）
- 代码块（支持语法高亮）
- 表格

### 3.2 Docx 导入

#### 解析流程

Docx 格式的解析采用了 "中间格式" 策略，先将 docx 转换为 HTML，再从 HTML 转换为 Prosemirror：

1. **Mammoth 转换**：使用 `mammoth` 库将 docx 文件转换为 HTML
2. **HTML 预处理**：清理 HTML 内容，处理特殊元素
3. **HTML 解析**：使用 JSDOM 解析 HTML
4. **Prosemirror 转换**：将 DOM 转换为 Prosemirror 格式

#### 关键代码

**Docx 转 HTML** (`DocumentConverter.ts:260-270`):
```typescript
private static async docxToHtml(content: Buffer | string): Promise<string> {
  if (content instanceof Buffer) {
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

**HTML 转 Prosemirror** (`DocumentConverter.ts:85-111`):
```typescript
public static htmlToProsemirror(content: Buffer | string): Node {
  if (typeof content !== "string") {
    content = content.toString("utf8");
  }

  const dom = new JSDOM(content);
  const document = dom.window.document;

  // Remove problematic elements before parsing
  const elementsToRemove = document.querySelectorAll(
    "script, style, title, head, meta, link"
  );
  elementsToRemove.forEach((el) => el.remove());

  // Preprocess the DOM to handle edge cases
  this.preprocessHtmlForImport(document);

  // Patch global environment for Prosemirror DOMParser
  const cleanup = ProsemirrorHelper.patchGlobalEnv(dom.window);

  try {
    const domParser = ProsemirrorDOMParser.fromSchema(schema);
    return domParser.parse(document.body);
  } finally {
    cleanup();
  }
}
```

**HTML 预处理** (`DocumentConverter.ts:119-175`):
预处理主要处理图片元素，包括：
- 过滤表情符号图片
- 移除 Jira 图标
- 处理 Confluence 图片尺寸
- 从 data URI 图片提取尺寸

#### 特殊处理：Confluence Word 导出

Outline 还支持一种特殊的 "Word" 格式，即 Confluence 导出的 Word 文件。这种文件实际上是多部分邮件消息，使用 `mailparser` 解析：

```typescript
private static async confluenceToHtml(
  content: Buffer | string
): Promise<string> {
  if (typeof content !== "string") {
    content = content.toString("utf8");
  }

  // We're only supporting the output from Confluence here, regular Word documents should call
  // into the docxToHtml importer. See: https://jira.atlassian.com/browse/CONFSERVER-38237
  if (!content.includes("Content-Type: multipart/related")) {
    throw FileImportError("Unsupported Word file");
  }

  // Confluence "Word" documents are actually just multi-part email messages, so we can use
  // mailparser to parse the content.
  const parsed = await simpleParser(content);
  if (!parsed.html) {
    throw FileImportError("Unsupported Word file (No content found)");
  }

  let html = parsed.html;

  // Replace the content-location with a data URI for each attachment.
  for (const attachment of parsed.attachments) {
    const contentLocation =
      (attachment.headers.get("content-location") as string | undefined) ??
      "";

    const id = contentLocation.split("/").pop();
    if (!id) {
      continue;
    }

    html = html.replace(
      new RegExp(escapeRegExp(id), "g"),
      `data:image/png;base64,${attachment.content.toString("base64")}`
    );
  }

  return html;
}
```

#### 支持的 Docx 特性

- 文本格式（加粗、斜体、下划线等）
- 标题层级
- 列表（有序、无序）
- 表格
- 图片（转换为 data URI）
- 链接
- 引用块

### 3.3 Notion 导入

#### 解析流程

Notion 导入采用了直接映射的策略，因为 Notion 的 Block 结构与 Prosemirror 的节点结构有相似之处：

1. **API 数据获取**：通过 Notion API 获取页面和块数据
2. **块映射**：将 Notion 的每种 Block 类型映射到对应的 Prosemirror 节点
3. **富文本处理**：转换 Notion 的富文本格式（annotations）
4. **嵌套处理**：处理嵌套块结构
5. **列表包装**：将 Notion 的扁平列表项包装为适当的列表结构

#### 关键代码

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
  const mapChild = (
    child: Block
  ): ProsemirrorData | ProsemirrorData[] | undefined => {
    if (child.type === "child_page") {
      return this.child_page(child);
    }
    if (child.type === "child_database") {
      return this.child_database(child);
    }

    // @ts-expect-error Not all blocks have an interface
    if (this[child.type]) {
      // @ts-expect-error Not all blocks have an interface
      const response = this[child.type](child);

      // @ts-expect-error Not all blocks have an interface
      const canToggle = child[child.type].is_toggleable === true;

      if (canToggle) {
        return {
          type: "container_toggle",
          attrs: {
            id: randomUUID(),
          },
          content: [response, ...this.mapChildren(child)],
        };
      }

      if (
        response &&
        this.nodesWithoutBlockChildren.includes(response.type) &&
        "children" in child
      ) {
        return [response, ...this.mapChildren(child)];
      }
      return response;
    }

    Logger.warn("Encountered unknown Notion block", child);
    return undefined;
  };

  let wrappingList;
  const children = [] as ProsemirrorData[];

  if (!item.children) {
    return [];
  }

  for (const child of item.children) {
    const mapped = mapChild(child);
    if (!mapped) {
      continue;
    }

    // Ensure lists are wrapped correctly – we require a wrapping element
    // whereas Notion does not
    // TODO: Handle mixed list
    if (child.type === "numbered_list_item") {
      if (!wrappingList) {
        wrappingList = {
          type: "ordered_list",
          content: [] as ProsemirrorData[],
        };
      }

      wrappingList.content.push(...(isArray(mapped) ? mapped : [mapped]));
      continue;
    }
    if (child.type === "bulleted_list_item") {
      if (!wrappingList) {
        wrappingList = {
          type: "bullet_list",
          content: [] as ProsemirrorData[],
        };
      }

      wrappingList.content.push(...(isArray(mapped) ? mapped : [mapped]));
      continue;
    }
    if (child.type === "to_do") {
      if (!wrappingList) {
        wrappingList = {
          type: "checkbox_list",
          content: [] as ProsemirrorData[],
        };
      }

      wrappingList.content.push(...(isArray(mapped) ? mapped : [mapped]));
      continue;
    }
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

  const mapAttrs = () =>
    Object.entries(item.annotations)
      .filter(([key]) => key !== "color")
      .filter(([, enabled]) => enabled)
      .map(([key]) => ({
        type: annotationToMark[key as keyof typeof annotationToMark],
      }));

  if (item.type === "mention") {
    if (item.mention.type === "page") {
      return {
        type: "mention",
        attrs: {
          type: MentionType.Document,
          label: item.plain_text,
          modelId: item.mention.page.id,
        },
      };
    }
    // ... 其他 mention 类型处理
  }

  if (item.type === "equation") {
    return item.equation.expression
      ? {
          type: "math_inline",
          content: [
            {
              type: "text",
              text: item.equation.expression,
            },
          ],
        }
      : undefined;
  }

  if (item.text.content) {
    return {
      type: "text",
      text: item.text.content,
      marks: [
        ...mapAttrs(),
        ...(item.text.link
          ? [{ type: "link", attrs: { href: item.text.link.url } }]
          : []),
      ].filter(Boolean),
    };
  }

  return undefined;
};
```

#### 支持的 Notion Block 类型

| Notion Block 类型 | Outline 对应节点 | 说明 |
|------------------|-----------------|------|
| `paragraph` | `paragraph` | 普通段落 |
| `heading_1/2/3` | `heading` (level 1/2/3) | 标题 |
| `bulleted_list_item` | `list_item` in `bullet_list` | 无序列表项 |
| `numbered_list_item` | `list_item` in `ordered_list` | 有序列表项 |
| `to_do` | `checkbox_item` in `checkbox_list` | 待办事项 |
| `quote` | `blockquote` | 引用块 |
| `code` | `code_fence` | 代码块 |
| `divider` | `hr` | 分割线 |
| `image` | `image` | 图片 |
| `file` | `attachment` | 附件 |
| `pdf` | `attachment` | PDF 附件 |
| `video` | `video` 或 `embed` | 视频 |
| `bookmark` | 链接文本 | 书签链接 |
| `callout` | `container_notice` | 提示块 |
| `toggle` | `container_toggle` | 折叠块 |
| `table` | `table` | 表格 |
| `equation` | `math_block` | 公式块 |
| `embed` | `embed` | 嵌入内容 |
| `child_page` | `mention` (Document 类型) | 子页面引用 |
| `child_database` | `mention` (Document 类型) | 子数据库引用 |
| `link_to_page` | `mention` (Document 类型) | 页面链接 |
| `column_list` | 平铺子节点 | 列列表（平铺处理） |
| `column` | 平铺子节点 | 列（平铺处理） |
| `synced_block` | 平铺子节点 | 同步块（平铺处理） |
| `link_preview` | 链接文本 | 链接预览 |
| `breadcrumb` | 忽略 | 面包屑（忽略） |
| `table_of_contents` | 忽略 | 目录（忽略） |

#### 特殊处理

**列表包装**：
Notion 的列表项是扁平的，而 Outline 需要包装在列表容器中。`mapChildren` 方法会自动将连续的同类型列表项包装到对应的列表容器中。

**可切换标题**：
Notion 的标题可以设置为可切换（toggleable），这种情况下会被包装为 `container_toggle` 节点。

**无子块容器的节点**：
某些节点（如 `paragraph`）不能包含块级子节点，它们的子节点会被平铺到父级。

## 4. 内容后处理

无论哪种格式，转换为 Prosemirror 后都会进行统一的后处理：

### 4.1 标题提取

从文档的第一个 H1 标题提取文档标题：

```typescript
// Extract title from first H1 heading
let title = "";
const headings = ProsemirrorHelper.getHeadings(doc);
if (headings.length > 0 && headings[0].level === 1) {
  title = headings[0].title;
  doc = ProsemirrorHelper.removeFirstHeading(doc);
}
```

### 4.2 图标提取

从文档开头提取 emoji 作为文档图标：

```typescript
// Extract emoji from start of document
const { emoji: icon, doc: docWithoutEmoji } =
  ProsemirrorHelper.extractEmojiFromStart(doc);
doc = docWithoutEmoji;
```

### 4.3 图片处理

将外部图片替换为附件：

```typescript
// Replace external images with attachments
const processedDoc = await ProsemirrorHelper.replaceImagesWithAttachments(
  ctx,
  doc,
  user
);
```

## 5. 数据存储

### 5.1 存储格式

导入的文档最终存储为两种格式：

1. **文本格式**：Prosemirror 序列化为 Markdown 文本
2. **状态格式**：Y.Doc 状态（用于实时协作）

### 5.2 转换流程

```typescript
// Serialize final text and handle empty documents
let text = serializer.serialize(processedDoc).trim();
// Empty paragraphs serialize to escaped newlines/backslashes, treat as empty
if (/^[\\\s]*$/.test(text)) {
  text = "";
}

// Truncate title and validate size
title = truncate(title, { length: DocumentValidation.maxTitleLength });
const state = convertToState(processedDoc.toJSON() as ProsemirrorData, title);

return { text, state, title, icon };
```

### 5.3 Y.Doc 转换

```typescript
function convertToState(content: ProsemirrorData, title: string): Buffer {
  const ydoc = ProsemirrorHelper.toYDoc(content);
  const state = ProsemirrorHelper.toState(ydoc);

  if (state.length > DocumentValidation.maxStateLength) {
    throw InvalidRequestError(
      `The document "${title}" is too large to import, please reduce the length and try again`
    );
  }

  return state;
}
```

### 5.4 数据库存储

导入结果包含以下字段，最终存储到 `documents` 表：

- `title`：文档标题
- `text`：文档内容（Markdown 格式）
- `state`：Y.Doc 状态（Buffer 格式）
- `icon`：文档图标（emoji）

## 6. 格式对比与技术选型

### 6.1 各格式转换策略

| 格式 | 转换策略 | 依赖库 | 优点 | 缺点 |
|------|---------|--------|------|------|
| **Markdown** | 直接解析为 Prosemirror | 内部解析器 | 简单高效，原生支持 | 格式有限 |
| **Docx** | 中间格式（Docx → HTML → Prosemirror） | mammoth, jsdom | 利用成熟的转换库，HTML 是通用中间格式 | 两次转换可能损失精度 |
| **Notion** | 直接映射（Block → Prosemirror Node） | @notionhq/client | 结构相似，映射直接，保留丰富特性 | 需处理每种 Block 类型 |

### 6.2 设计决策分析

1. **统一中间格式**：
   - 选择 Prosemirror 作为统一的内部格式，所有导入格式都转换为 Prosemirror
   - 优点：简化后续处理，统一的编辑体验
   - 缺点：需要为每种格式编写转换器

2. **Docx 采用 HTML 中间格式**：
   - 选择先转 HTML 再转 Prosemirror，而不是直接转 Prosemirror
   - 原因：mammoth 库成熟稳定，HTML 到 Prosemirror 的转换已存在（用于其他场景）

3. **Notion 采用直接映射**：
   - Notion 的 Block 结构与 Prosemirror 的节点结构相似
   - 直接映射可以保留更多 Notion 的特性（如 callout、toggle 等）

4. **Y.Doc 状态存储**：
   - 使用 Y.js 进行实时协作，因此需要将 Prosemirror 转换为 Y.Doc
   - 同时保留 Markdown 文本格式，用于搜索、导出等场景

## 7. 错误处理与验证

### 7.1 文件类型验证

```typescript
private static async convertToMarkdown(
  content: Buffer | string,
  fileName: string,
  mimeType: string
): Promise<string> {
  // ... 类型判断逻辑
  default:
    throw FileImportError(`File type ${mimeType} not supported`);
}
```

### 7.2 文档大小验证

```typescript
function convertToState(content: ProsemirrorData, title: string): Buffer {
  const ydoc = ProsemirrorHelper.toYDoc(content);
  const state = ProsemirrorHelper.toState(ydoc);

  if (state.length > DocumentValidation.maxStateLength) {
    throw InvalidRequestError(
      `The document "${title}" is too large to import, please reduce the length and try again`
    );
  }

  return state;
}
```

### 7.3 标题长度限制

```typescript
title = truncate(title, { length: DocumentValidation.maxTitleLength });
```

## 8. 扩展与维护

### 8.1 添加新格式支持

要添加新的导入格式，需要：

1. 在 `DocumentConverter.convertToHtml()` 或 `DocumentConverter.convertToMarkdown()` 中添加类型判断
2. 实现对应的转换方法（如 `newFormatToHtml()` 或 `newFormatToMarkdown()`）
3. 确保转换结果可以被现有的 HTML 或 Markdown 解析器处理

### 8.2 维护考虑

- **Mammoth 库更新**：Docx 转换依赖 mammoth 库，需关注其更新
- **Notion API 变化**：Notion 导入依赖 Notion API，API 变化可能需要调整 `NotionConverter`
- **Prosemirror 架构**：内部格式基于 Prosemirror，其 schema 变化会影响所有转换器

## 9. 总结

Outline 的文档导入系统设计合理，采用了分层架构和统一中间格式的策略：

1. **格式识别**：根据 MIME 类型和文件扩展名自动识别
2. **多策略转换**：
   - Markdown：直接解析
   - Docx：中间格式转换（Docx → HTML → Prosemirror）
   - Notion：直接映射（Block → Prosemirror Node）
3. **统一后处理**：标题提取、图标提取、图片处理
4. **双格式存储**：Markdown 文本 + Y.Doc 状态

这种设计既保证了各种格式的准确转换，又提供了统一的内部处理和存储机制，为用户提供了流畅的导入体验。