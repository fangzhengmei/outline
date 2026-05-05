# Outline API Token 与 OAuth 授权实现分析

## 一、概述

Outline 实现了两套权限认证机制：

1. **API Token** - 用户直接创建的 API 密钥，用于个人/脚本访问
2. **OAuth 2.0** - 标准的第三方应用授权框架，支持授权码流程

两套机制共享相同的 **Scope 权限范围控制模型**，确保权限验证逻辑的一致性。

---

## 二、API Token 实现分析

### 2.1 数据模型 (`server/models/ApiKey.ts`)

```typescript
@Table({ tableName: "apiKeys" })
class ApiKey extends ParanoidModel {
  static prefix = "ol_api_";  // 密钥前缀

  name: string;           // 密钥名称
  scope: string[] | null; // 权限范围数组
  secret: string;         // @deprecated 明文密钥
  value: string | null;   // 虚拟字段：创建时返回明文
  hash: string;           // SHA-256 哈希值
  last4: string;          // 最后 4 位（用于显示）
  expiresAt: Date | null; // 过期时间
  lastActiveAt: Date | null; // 最后活跃时间
  userId: string;         // 所属用户
}
```

**核心设计要点**：

| 特性 | 实现方式 |
|------|----------|
| 密钥格式 | `ol_api_` + 38 位随机字符串 |
| 存储安全 | 使用 SHA-256 哈希存储，不保存明文 |
| 向后兼容 | 同时支持 `secret` (旧) 和 `hash` (新) 字段查询 |
| 过期机制 | 支持可选的过期时间 |

### 2.2 Token 创建流程

**路由**: `POST /api/apiKeys.create` (`server/routes/api/apiKeys/apiKeys.ts`)

**创建流程**：

```
1. 认证检查 (auth 中间件)
   └─ 要求 type: AuthenticationType.APP (不能用 API key 创建 API key)
   └─ 要求 role: UserRole.Member

2. 权限检查 (authorize)
   └─ authorize(user, "createApiKey", user.team)

3. Scope 规范化
   └─ 自动添加 /api/ 前缀（如果没有的话）
   └─ 支持三种格式: 全局 scope、命名空间 scope、路由 scope

4. 创建记录
   └─ @BeforeValidate 钩子生成随机密钥
   └─ hash = SHA256(value)
   └─ 返回明文 value (仅创建时可见)
```

**Scope 规范化逻辑**：

```typescript
scope: scope?.map((s) =>
  s.startsWith("/api/") || s.includes(":") || globalScopes.has(s)
    ? s
    : `/api/${s.replace(/^\//, "")}`
)
```

### 2.3 Token 验证流程

**中间件**: `server/middlewares/authentication.ts`

**验证步骤**：

```typescript
// 1. 识别 token 类型
if (ApiKey.match(token)) {
  // 格式检查: ol_api_ 前缀 + 38 位字符

  // 2. 安全限制: 禁止通过 cookie 传输 API key
  if (transport === "cookie") {
    throw AuthenticationError("API key must not be passed in the cookie");
  }

  // 3. 数据库查询 (同时检查 secret 和 hash)
  apiKey = await ApiKey.findByToken(token);
  // SQL: WHERE secret = ? OR hash = SHA256(?)

  // 4. 过期检查
  if (apiKey.expiresAt && apiKey.expiresAt < new Date()) {
    throw AuthenticationError("API key is expired");
  }

  // 5. Scope 权限检查 ⭐
  if (!apiKey.canAccess(ctx.originalUrl)) {
    throw AuthorizationError("API key does not have access to this resource");
  }

  // 6. 更新活跃时间
  await apiKey.updateActiveAt();
}
```

### 2.4 API Key 操作路由

| 路由 | 方法 | 说明 | 权限要求 |
|------|------|------|----------|
| `apiKeys.create` | POST | 创建 API Key | APP 认证 + Member |
| `apiKeys.list` | POST | 列出 API Keys | Member |
| `apiKeys.delete` | POST | 删除 API Key | APP 认证 + Member |

---

## 三、OAuth 2.0 授权实现分析

### 3.1 核心模型

#### OAuthClient (`server/models/oauth/OAuthClient.ts`)

第三方应用客户端信息：

```typescript
@Table({ tableName: "oauth_clients" })
class OAuthClient extends ParanoidModel {
  static clientSecretPrefix = "ol_sk_";
  static registrationAccessTokenPrefix = "ol_rat_";

  name: string;                    // 应用名称
  description: string | null;       // 应用描述
  developerName: string | null;     // 开发者名称
  developerUrl: string | null;      // 开发者网址
  avatarUrl: string | null;         // 应用图标
  clientId: string;                 // 公开客户端 ID
  clientType: "public" | "confidential"; // 客户端类型
  clientSecret: string;             // 客户端密钥 (加密存储)
  redirectUris: string[];           // 允许的重定向 URI
  published: boolean;                // 是否发布
  registrationAccessTokenHash: string | null; // DCR 管理令牌哈希
  teamId: string;                   // 所属团队
  createdById: string | null;       // 创建者 (null 表示 DCR 动态注册)
}
```

#### OAuthAuthorizationCode (`server/models/oauth/OAuthAuthorizationCode.ts`)

授权码（一次性使用）：

```typescript
@Table({ tableName: "oauth_authorization_codes" })
class OAuthAuthorizationCode extends IdModel {
  static authorizationCodePrefix = "ol_ac_";
  static authorizationCodeLifetime = env.OAUTH_PROVIDER_AUTHORIZATION_CODE_LIFETIME;

  authorizationCodeHash: string;     // 授权码哈希
  codeChallenge?: string;            // PKCE 挑战值
  codeChallengeMethod?: string;      // PKCE 方法 (S256/plain)
  grantId: string | null;            // 授权会话 ID (用于 token 轮换和撤销)
  scope: string[];                    // 权限范围
  redirectUri: string;                // 重定向 URI
  expiresAt: Date;                    // 过期时间
  oauthClientId: string;              // 所属客户端
  userId: string;                     // 授权用户
}
```

#### OAuthAuthentication (`server/models/oauth/OAuthAuthentication.ts`)

Access Token 和 Refresh Token 存储：

```typescript
@Table({ tableName: "oauth_authentications" })
class OAuthAuthentication extends ParanoidModel {
  static accessTokenPrefix = "ol_at_";
  static refreshTokenPrefix = "ol_rt_";
  static accessTokenLifetime = env.OAUTH_PROVIDER_ACCESS_TOKEN_LIFETIME;
  static refreshTokenLifetime = env.OAUTH_PROVIDER_REFRESH_TOKEN_LIFETIME;

  accessTokenHash: string;           // Access Token 哈希
  accessTokenExpiresAt: Date;         // Access Token 过期时间
  refreshTokenHash: string;           // Refresh Token 哈希
  refreshTokenExpiresAt: Date;        // Refresh Token 过期时间
  grantId: string | null;             // 授权会话 ID
  scope: string[];                     // 权限范围
  lastActiveAt: Date;                  // 最后活跃时间
  oauthClientId: string;               // 所属客户端
  userId: string;                      // 所属用户
}
```

### 3.2 OAuth 服务配置

**路由入口**: `server/routes/oauth/index.ts`

使用第三方库 `@node-oauth/oauth2-server` 实现标准 OAuth 2.0 协议：

```typescript
const oauth = new OAuth2Server({
  model: OAuthInterface,  // 自定义数据模型实现
  requireClientAuthentication: {
    refresh_token: false, // 公共客户端刷新 token 无需 client_secret
  },
  alwaysIssueNewRefreshToken: true,  // 每次刷新都发放新 token (RFC 6819)
});
```

**支持的 Grant Types**:
- `authorization_code` - 授权码流程
- `refresh_token` - 刷新令牌

### 3.3 OAuthInterface 核心实现

**文件**: `server/utils/oauth/OAuthInterface.ts`

这是 `@node-oauth/oauth2-server` 库的 Model 接口实现，核心方法：

| 方法 | 作用 |
|------|------|
| `generateAccessToken()` | 生成 `ol_at_` 前缀的 Access Token |
| `generateRefreshToken()` | 生成 `ol_rt_` 前缀的 Refresh Token |
| `generateAuthorizationCode()` | 生成 `ol_ac_` 前缀的授权码 |
| `getAccessToken()` | 通过哈希查找 Access Token |
| `getRefreshToken()` | 通过哈希查找 Refresh Token (含重用检测) |
| `getAuthorizationCode()` | 通过哈希查找授权码 |
| `getClient()` | 验证客户端 ID 和密钥 |
| `saveToken()` | 保存 Access/Refresh Token |
| `saveAuthorizationCode()` | 保存授权码 |
| `revokeToken()` | 撤销 Refresh Token |
| `revokeAuthorizationCode()` | 撤销授权码 |
| `validateRedirectUri()` | 验证重定向 URI |
| `validateScope()` | 验证请求的 Scope 是否有效 |

**安全特性**：

1. **Token 哈希存储**: 所有 token 都使用 SHA-256 哈希存储，不存明文
2. **Refresh Token 重用检测** (`getRefreshToken`):
   - 如果 refresh token 已被使用（软删除），撤销该 grantId 下的所有 token
   - 遵循 RFC 9700 安全建议
3. **时间常量比较**: 使用 `safeEqual` 比较 clientSecret，防止时序攻击

### 3.4 完整 OAuth 授权码流程

```
┌──────────┐          ┌─────────────┐          ┌──────────────┐
│  第三方应用 │          │  Outline 前端 │          │  Outline 后端  │
└────┬─────┘          └──────┬──────┘          └──────┬───────┘
     │                        │                        │
     │  1. 重定向到授权页面    │                        │
     │───────────────────────>│                        │
     │  GET /oauth/authorize? │                        │
     │  client_id=xxx&        │                        │
     │  redirect_uri=xxx&     │                        │
     │  scope=read&            │                        │
     │  state=xxx              │                        │
     │                        │                        │
     │                        │  2. 用户登录/授权        │
     │                        │  (auth 中间件)          │
     │                        │───────────────────────>│
     │                        │                        │
     │                        │  3. 生成授权码           │
     │                        │  oauth.authorize()      │
     │                        │───────────────────────>│
     │                        │                        │
     │  4. 重定向带回 code     │                        │
     │<───────────────────────│                        │
     │  302 redirect_uri?     │                        │
     │  code=ol_ac_xxx&       │                        │
     │  state=xxx              │                        │
     │                        │                        │
     │  5. 用 code 换 token    │                        │
     │───────────────────────────────────────────────>│
     │  POST /oauth/token     │                        │
     │  grant_type=authorization_code │                │
     │  code=xxx&              │                        │
     │  client_id=xxx&         │                        │
     │  client_secret=xxx&     │                        │
     │  redirect_uri=xxx       │                        │
     │                        │                        │
     │  6. 返回 token           │                        │
     │<───────────────────────────────────────────────│
     │  {                      │                        │
     │    access_token: ol_at_xxx, │                  │
     │    refresh_token: ol_rt_xxx,│                  │
     │    expires_in: 3600,   │                        │
     │    scope: "read"        │                        │
     │  }                      │                        │
     │                        │                        │
     │  7. 使用 Access Token   │                        │
     │───────────────────────────────────────────────>│
     │  Authorization: Bearer  │                        │
     │  ol_at_xxx              │                        │
```

### 3.5 OAuth 端点详解

#### POST /oauth/authorize - 获取授权码

```typescript
router.post("/authorize",
  rateLimiter(RateLimiterStrategy.OneHundredPerHour),
  auth(),  // 用户必须已登录
  async (ctx) => {
    const { user } = ctx.state.auth;
    const clientId = ctx.request.body.client_id;
    
    // 1. 查找客户端
    const client = await OAuthClient.findByClientId(clientId);
    
    // 2. 授权检查 (用户对客户端的 read 权限)
    authorize(user, "read", client);
    
    // 3. 调用 OAuth2Server 生成授权码
    const authorizationCode = await oauth.authorize(request, response, {
      allowEmptyState: false,  // 强制 state 防止 CSRF
      authorizationCodeLifetime: ...,
      authenticateHandler: {
        handle: async () => user,  // 传递当前用户
      },
    });
  }
);
```

#### POST /oauth/token - 交换令牌

```typescript
router.post("/token",
  validate(T.TokenSchema),
  rateLimiter(RateLimiterStrategy.OneHundredPerHour),
  async (ctx) => {
    const grantType = ctx.input.body.grant_type;
    
    // 机密客户端刷新 token 时必须提供 client_secret
    if (grantType === "refresh_token" && !clientSecret) {
      const client = await OAuthClient.findByClientId(clientId);
      if (client.clientType === "confidential") {
        throw ValidationError("Missing client_secret for confidential client");
      }
    }
    
    const token = await oauth.token(request, response, {
      accessTokenLifetime: OAuthAuthentication.accessTokenLifetime,
      refreshTokenLifetime: OAuthAuthentication.refreshTokenLifetime,
    });
    
    ctx.body = {
      access_token: token.accessToken,
      refresh_token: token.refreshToken,
      expires_in: ...,
      token_type: "Bearer",
      scope: token.scope?.join(" "),
    };
  }
);
```

#### POST /oauth/revoke - 撤销令牌

```typescript
router.post("/revoke",
  validate(T.TokenRevokeSchema),
  transaction(),
  async (ctx) => {
    const { token } = ctx.input.body;
    
    // 尝试撤销 Access Token
    if (OAuthAuthentication.match(token)) {
      const accessToken = await OAuthAuthentication.findByAccessToken(token);
      await accessToken?.destroyWithCtx(ctx);
    }
    
    // 尝试撤销 Refresh Token
    if (OAuthAuthentication.matchRefreshToken(token)) {
      const refreshToken = await OAuthAuthentication.findByRefreshToken(token);
      await refreshToken?.destroyWithCtx(ctx);
    }
    
    // RFC 7009 §2.2: 无效 token 不返回错误
    ctx.body = { success: true };
  }
);
```

### 3.6 动态客户端注册 (DCR)

**RFC 7591, RFC 7592 实现**

```typescript
// POST /oauth/register - 注册新客户端
router.post("/register",
  validate(T.RegisterSchema),
  rateLimiter(RateLimiterStrategy.FivePerHour),
  async (ctx) => {
    // 功能开关检查
    if (env.OAUTH_DISABLE_DCR) throw NotFoundError();
    if (!team.getPreference(TeamPreference.MCP)) throw NotFoundError();
    
    // 根据认证方式决定客户端类型
    const clientType = token_endpoint_auth_method === "client_secret_post"
      ? "confidential" 
      : "public";
    
    const client = await OAuthClient.createWithCtx(ctx, {
      name: client_name,
      redirectUris: redirect_uris,
      clientType,
      teamId: team.id,
      createdById: null,  // 标记为 DCR 客户端
      published: false,
    });
    
    // 返回 registrationAccessToken (仅创建时可见)
    ctx.status = 201;
    ctx.body = presentDCRClient(team.url, client, {
      includeRegistrationAccessToken: true,
      includeCredentials: true,
    });
  }
);
```

**DCR 管理端点**：
- `GET /oauth/register/:clientId` - 查询客户端信息 (需要 registrationAccessToken)
- `PUT /oauth/register/:clientId` - 更新客户端 (自动轮换 registrationAccessToken)
- `DELETE /oauth/register/:clientId` - 删除客户端

---

## 四、Scope 权限范围控制机制

### 4.1 Scope 类型定义

**文件**: `shared/types.ts`

```typescript
export enum Scope {
  Read = "read",
  Write = "write",
  Create = "create",
}
```

### 4.2 三种 Scope 格式

`AuthenticationHelper.canAccess()` 支持三种格式的 scope (`shared/helpers/AuthenticationHelper.ts`)：

#### 格式 1: 全局 Scope (`scope`)

适用于所有资源的通用权限：

| Scope | 权限说明 | 可访问的方法 |
|-------|----------|--------------|
| `read` | 只读 | list, info, search, documents, drafts, viewed, export, config |
| `write` | 读写 | 所有方法 (包含 read 权限) |
| `create` | 仅创建 | create 方法 |

**示例**：
```javascript
canAccess("/api/documents.info", ["read"]);    // true
canAccess("/api/documents.create", ["read"]);  // false
canAccess("/api/documents.update", ["write"]); // true
```

#### 格式 2: 命名空间 Scope (`namespace:scope`)

限定特定资源类型的权限：

**格式**: `{namespace}:{scope}`

- `namespace`: API 命名空间 (如 `documents`, `users`, `collections`)
- `scope`: `read`, `write`, `create`

**示例**：
```javascript
canAccess("/api/documents.info", ["documents:read"]);    // true
canAccess("/api/users.info", ["documents:read"]);         // false (不同命名空间)
canAccess("/api/documents.create", ["documents:create"]); // true
canAccess("/api/documents.update", ["documents:write"]);  // true
```

#### 格式 3: 路由 Scope (`/api/namespace.method`)

精确控制单个 API 端点：

**格式**: `/api/{namespace}.{method}`

支持通配符：
- `*` 可用于 namespace 或 method 位置

**示例**：
```javascript
// 精确匹配
canAccess("/api/documents.info", ["/api/documents.info"]); // true
canAccess("/api/documents.create", ["/api/documents.info"]); // false

// 通配符: 某个 namespace 下的所有方法
canAccess("/api/documents.info", ["/api/documents.*"]); // true
canAccess("/api/documents.create", ["/api/documents.*"]); // true
canAccess("/api/users.info", ["/api/documents.*"]); // false

// 通配符: 所有 namespace 的某个方法
canAccess("/api/documents.info", ["/api/*.info"]); // true
canAccess("/api/users.info", ["/api/*.info"]); // true
canAccess("/api/documents.create", ["/api/*.info"]); // false
```

### 4.3 方法到 Scope 的映射

**`AuthenticationHelper.methodToScope`** 定义了默认映射规则：

```typescript
private static methodToScope = {
  create: Scope.Create,
  config: Scope.Read,
  list: Scope.Read,
  info: Scope.Read,
  search: Scope.Read,
  documents: Scope.Read,
  drafts: Scope.Read,
  viewed: Scope.Read,
  export: Scope.Read,
  // 未列出的方法默认为 Scope.Write
};
```

**规则**：
- 表中列出的方法对应特定 scope
- **未列出的方法默认需要 `write` 权限** (如 `update`, `delete` 等)

### 4.4 特殊处理

#### MCP 端点 (`/mcp/*`)

```typescript
// ApiKey.canAccess() 和 OAuthAuthentication.canAccess()
if (path.startsWith("/mcp")) {
  return this.scope.length > 0;  // 只要有 scope 就允许访问
  // 细粒度权限在 tool 级别检查
}
```

#### OAuth 撤销端点

```typescript
// OAuthAuthentication.canAccess()
if (path === "/oauth/revoke") {
  return true;  // 始终允许撤销自己的 token
}
```

#### 通配符 Scope (`*`)

```typescript
if (scopes.includes("*")) {
  return true;  // 完全访问权限
}
```

### 4.5 Scope 验证流程

```typescript
// 伪代码: AuthenticationHelper.canAccess(path, scopes)
function canAccess(path, scopes) {
  // 1. 检查通配符
  if (scopes.includes("*")) return true;
  
  // 2. 去除查询字符串
  path = path.split("?")[0];
  
  // 3. 解析请求路径
  // /api/documents.info -> namespace="documents", method="info"
  const resource = path.split("/").pop();
  const [namespace, method] = resource.split(".");
  
  // 4. 遍历所有 scope 检查是否有匹配
  return scopes.some(scope => {
    // 解析 scope 格式
    const [scopeNamespace, scopeMethod] = scope.match(/[:.]/g)
      ? scope.replace("/api/", "").split(/[:.]/g)
      : ["*", scope];  // 全局 scope
    
    // 路由格式: /api/documents.info
    if (scope.startsWith("/api/")) {
      return (namespace === scopeNamespace || scopeNamespace === "*")
          && (method === scopeMethod || scopeMethod === "*");
    }
    
    // 命名空间格式: documents:read 或 全局格式: read
    return (namespace === scopeNamespace || scopeNamespace === "*")
        && (scopeMethod === "write"  // write 包含所有权限
            || methodToScope[method] === scopeMethod);
  });
}
```

### 4.6 Scope 验证入口

**API Key 验证**: `server/models/ApiKey.ts:174`
```typescript
canAccess = (path: string) => {
  if (!this.scope) return true;  // null 表示无限制
  
  if (path.startsWith("/mcp")) return this.scope.length > 0;
  
  return AuthenticationHelper.canAccess(path, this.scope);
};
```

**OAuth Token 验证**: `server/models/oauth/OAuthAuthentication.ts:143`
```typescript
canAccess = (path: string) => {
  if (path === "/oauth/revoke") return true;
  
  if (path.startsWith("/mcp")) return this.scope.length > 0;
  
  return AuthenticationHelper.canAccess(path, this.scope);
};
```

---

## 五、认证中间件统一处理

**文件**: `server/middlewares/authentication.ts`

三种认证方式的统一处理流程：

```
请求到达
    │
    ▼
parseAuthentication() 解析 token
│
├─── Header: Authorization: Bearer <token>
├─── Body: { token: <token> }
├─── Query: ?token=<token>
└─── Cookie: accessToken
    │
    ▼
validateAuthentication() 根据 token 类型分发
│
├─── OAuth Token (ol_at_ 前缀)
│    │
│    ├── 1. 必须通过 header 传输
│    ├── 2. 数据库查找 (accessTokenHash)
│    ├── 3. 过期检查
│    ├── 4. canAccess() 权限检查
│    └── 5. 更新 lastActiveAt
│
├─── API Key (ol_api_ 前缀或 38 位字符)
│    │
│    ├── 1. 禁止通过 cookie 传输
│    ├── 2. 数据库查找 (secret 或 hash)
│    ├── 3. 过期检查
│    ├── 4. canAccess() 权限检查
│    └── 5. 更新 lastActiveAt
│
└─── JWT (应用内登录)
     │
     └── getUserForJWT() 解析
```

**认证类型枚举**: `server/types.ts`
```typescript
export enum AuthenticationType {
  APP = "app",         // JWT (应用内登录)
  API = "api",         // API Key
  OAUTH = "oauth",     // OAuth Access Token
}
```

---

## 六、安全特性总结

| 安全特性 | 实现位置 | 说明 |
|----------|----------|------|
| Token 哈希存储 | 所有模型 | SHA-256 哈希，不留明文 |
| 传输限制 | authentication.ts | API Key 禁止 cookie 传输，OAuth 要求 header |
| 过期机制 | 所有 token 模型 | Access Token、Refresh Token、授权码都有过期时间 |
| PKCE 支持 | OAuthAuthorizationCode | `codeChallenge` 和 `codeChallengeMethod` 字段 |
| Refresh Token 轮换 | routes/oauth/index.ts | `alwaysIssueNewRefreshToken: true` |
| Refresh Token 重用检测 | OAuthInterface.getRefreshToken() | 检测到重用立即撤销整个 grant |
| State 参数强制 | oauth.authorize() | `allowEmptyState: false` 防止 CSRF |
| 重定向 URI 白名单 | OAuthInterface.validateRedirectUri() | 必须精确匹配预注册的 redirectUris |
| 时序攻击防护 | OAuthInterface.getClient() | 使用 `safeEqual` 比较密钥 |
| 速率限制 | 所有路由 | `rateLimiter` 中间件 |
| 软删除 | ParanoidModel | 保留审计日志，支持 token 重用检测 |

---

## 七、关键配置项

| 环境变量 | 说明 | 来源 |
|----------|------|------|
| `OAUTH_PROVIDER_AUTHORIZATION_CODE_LIFETIME` | 授权码有效期 (秒) | 授权码过期 |
| `OAUTH_PROVIDER_ACCESS_TOKEN_LIFETIME` | Access Token 有效期 (秒) | Token 过期 |
| `OAUTH_PROVIDER_REFRESH_TOKEN_LIFETIME` | Refresh Token 有效期 (秒) | Token 过期 |
| `OAUTH_DISABLE_DCR` | 禁用动态客户端注册 | DCR 开关 |

---

## 八、数据模型关系图

```
┌─────────────────┐       ┌──────────────────────┐
│     User        │       │      Team            │
└────────┬────────┘       └──────────┬───────────┘
         │                             │
         │ 1:N                         │ 1:N
         ▼                             ▼
┌─────────────────┐       ┌──────────────────────┐
│    ApiKey       │       │    OAuthClient       │
│                 │       │                      │
│ - name          │       │ - clientId           │
│ - scope[]       │       │ - clientSecret (加密)│
│ - hash          │       │ - redirectUris[]     │
│ - last4         │       │ - clientType         │
│ - expiresAt     │       │ - teamId             │
│ - userId        │       │ - createdById        │
└─────────────────┘       └──────────┬───────────┘
                                      │
                                      │ 1:N
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
           ┌──────────────┐ ┌──────────────────┐ ┌───────────┐
           │OAuthAuthCode │ │OAuthAuthentication│ │ DCR Token │
           │              │ │                  │ │ (管理用)  │
           │ - codeHash   │ │ - accessTokenHash│ └───────────┘
           │ - codeChallenge││ - refreshTokenHash│
           │ - scope[]    │ │ - scope[]          │
           │ - grantId    │ │ - grantId          │
           │ - expiresAt  │ │ - expiresAt        │
           │ - clientId   │ │ - clientId         │
           │ - userId     │ │ - userId           │
           └──────────────┘ └──────────────────┘
```

---

## 九、参考文件位置

### 核心模型

| 模型 | 文件路径 |
|------|----------|
| API Key | `server/models/ApiKey.ts` |
| OAuth 客户端 | `server/models/oauth/OAuthClient.ts` |
| OAuth 授权码 | `server/models/oauth/OAuthAuthorizationCode.ts` |
| OAuth Token | `server/models/oauth/OAuthAuthentication.ts` |

### 路由

| 功能 | 文件路径 |
|------|----------|
| API Key 管理 | `server/routes/api/apiKeys/apiKeys.ts` |
| OAuth 2.0 端点 | `server/routes/oauth/index.ts` |
| OAuth 客户端管理 | `server/routes/api/oauthClients/oauthClients.ts` |

### 核心逻辑

| 功能 | 文件路径 |
|------|----------|
| 认证中间件 | `server/middlewares/authentication.ts` |
| Scope 验证 | `shared/helpers/AuthenticationHelper.ts` |
| OAuth 协议实现 | `server/utils/oauth/OAuthInterface.ts` |
| Scope 枚举定义 | `shared/types.ts` |

### 测试文件

| 功能 | 文件路径 |
|------|----------|
| Scope 验证测试 | `shared/helpers/AuthenticationHelper.test.ts` |
| API Key 测试 | `server/models/ApiKey.test.ts` |
| OAuth Token 测试 | `server/models/oauth/OAuthAuthentication.test.ts` |
