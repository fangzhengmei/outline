# Outline API Token 与 OAuth 授权链路深度分析 (R2)

## 一、核心链路总览

Outline 实现了两套独立但共享权限模型的认证机制：

| 特性 | API Token | OAuth 2.0 |
|------|-----------|-----------|
| 使用者 | 用户/脚本 | 第三方应用 |
| 创建方式 | 用户在设置中创建 | OAuth 授权码流程 |
| Token 格式 | `ol_api_` + 38 字符 | `ol_at_` (Access), `ol_rt_` (Refresh) |
| Scope 存储 | `string[] \| null` | `string[]` (始终为数组) |
| 空 Scope 权限 | 完全访问 (`null`) | 无任何访问 (`[]`) |

---

## 二、OAuth 授权入口请求方式详解

### 2.1 两阶段请求流程

OAuth 授权入口采用 **GET + POST 两阶段** 设计：

```
┌──────────────┐              ┌──────────────┐              ┌──────────────┐
│  第三方应用   │              │   浏览器     │              │ Outline 服务 │
└──────┬───────┘              └──────┬───────┘              └──────┬───────┘
       │                             │                             │
       │ 1. 构造授权 URL             │                             │
       │────────────────────────────>│                             │
       │ GET /oauth/authorize?       │                             │
       │   client_id=xxx&            │                             │
       │   redirect_uri=xxx&         │                             │
       │   response_type=code&       │                             │
       │   state=xxx&                │                             │
       │   scope=read                │                             │
       │                             │                             │
       │                             │ 2. 前端路由捕获，显示授权页面  │
       │                             │────────────────────────────>│
       │                             │                             │
       │                             │ 3. 用户点击 "Authorize"      │
       │                             │<────────────────────────────│
       │                             │                             │
       │                             │ 4. 表单 POST 提交            │
       │                             │────────────────────────────>│
       │                             │ POST /oauth/authorize        │
       │                             │ Body: {                      │
       │                             │   client_id, redirect_uri,  │
       │                             │   response_type, state,     │
       │                             │   scope, code_challenge...  │
       │                             │ }                           │
       │                             │                             │
       │                             │ 5. 返回 302 重定向           │
       │                             │<────────────────────────────│
       │                             │ Location: redirect_uri?code │
       │                             │                             │
       │ 6. 携带 code 重定向         │                             │
       │<────────────────────────────│                             │
       │                             │                             │
```

### 2.2 第一阶段：GET 请求（前端路由）

**触发方式**: 第三方应用构造链接，用户点击或浏览器重定向

**参数解析** (`app/scenes/Login/OAuthAuthorize.tsx:86-94`)：

```typescript
const params = useQuery();
const {
  client_id: clientId,
  redirect_uri: redirectUri,
  response_type: responseType,
  code_challenge: codeChallenge,
  code_challenge_method: rawCodeChallengeMethod,
  state,
  scope,
} = Object.fromEntries(params);
```

**必填参数校验** (`OAuthAuthorize.tsx:129-134`)：

```typescript
const missingParams = [
  !clientId && "client_id",
  !redirectUri && "redirect_uri",
  !responseType && "response_type",
  !state && "state",  // ⭐ state 是必填参数
].filter(Boolean);
```

### 2.3 第二阶段：POST 请求（后端处理）

**表单提交** (`OAuthAuthorize.tsx:257-290`)：

```tsx
<Form
  method="POST"
  action="/oauth/authorize"
  style={{ width: "100%" }}
  onSubmit={handleSubmit}
>
  <input type="hidden" name="client_id" value={clientId ?? ""} />
  <input type="hidden" name="redirect_uri" value={redirectUri ?? ""} />
  <input type="hidden" name="response_type" value={responseType ?? ""} />
  <input type="hidden" name="state" value={state ?? ""} />
  <input type="hidden" name="scope" value={scopes.join(" ")} />
  {codeChallenge && (
    <input type="hidden" name="code_challenge" value={codeChallenge} />
  )}
  {codeChallengeMethod && (
    <input
      type="hidden"
      name="code_challenge_method"
      value={codeChallengeMethod}
    />
  )}
  {/* 提交按钮 */}
</Form>
```

**后端路由处理** (`server/routes/oauth/index.ts:41-88`)：

```typescript
router.post(
  "/authorize",
  rateLimiter(RateLimiterStrategy.OneHundredPerHour),
  auth(),  // ⭐ 用户必须已登录（cookie 认证）
  async (ctx) => {
    const { user } = ctx.state.auth;
    const clientId = ctx.request.body.client_id;
    // ...
    
    const authorizationCode = await oauth.authorize(request, response, {
      allowEmptyState: false,  // ⭐ 强制 state 参数
      authorizationCodeLifetime: OAuthAuthorizationCode.authorizationCodeLifetime,
      authenticateHandler: {
        handle: async () => user,
      },
    });
    
    // 302 重定向到 redirect_uri
    if (response.status === 302 && response.headers?.location) {
      ctx.redirect(location);
      return;
    }
  }
);
```

### 2.4 路由方式关键结论

| 阶段 | HTTP 方法 | 处理位置 | 主要作用 |
|------|-----------|----------|----------|
| 1 | GET | 前端 React 路由 | 显示授权确认页面 |
| 2 | POST | 后端 Koa 路由 | 生成授权码并返回 |

**重要提醒**: 标准 OAuth 2.0 的 `authorization_endpoint` 通常同时支持 GET 和 POST，但 Outline 的实现中：
- 前端只构造 GET 链接供用户点击
- 后端只处理 POST 请求（通过表单提交）
- GET 请求由前端 React 应用捕获，不直接进入后端 OAuth 处理

---

## 三、空 Scope 时的默认权限边界

### 3.1 两种 "空 Scope" 的本质区别

Outline 中存在两种含义不同的 "空 Scope"：

| 场景 | 数据类型 | 权限含义 | 适用对象 |
|------|----------|----------|----------|
| **未设置 Scope** | `scope: null` | 完全访问权限 | API Key |
| **空 Scope 数组** | `scope: []` | 无任何访问权限 | OAuth Token |

### 3.2 API Key 的 `scope: null` → 完全访问

**代码位置**: `server/models/ApiKey.ts:174`

```typescript
canAccess = (path: string) => {
  if (!this.scope) {  // scope 为 null 或 undefined
    return true;  // ⭐ 完全访问权限！
  }
  
  // MCP 特例
  if (path.startsWith("/mcp")) {
    return this.scope.length > 0;
  }
  
  return AuthenticationHelper.canAccess(path, this.scope);
};
```

**创建时的 Scope 处理** (`server/routes/api/apiKeys/apiKeys.ts:36-44`)：

```typescript
const apiKey = await ApiKey.createWithCtx(ctx, {
  name,
  userId: user.id,
  expiresAt,
  scope: scope?.map((s) =>  // ⭐ 如果 scope 参数未传，则为 undefined
    s.startsWith("/api/") || s.includes(":") || globalScopes.has(s)
      ? s
      : `/api/${s.replace(/^\//, "")}`
  ),
});
```

**关键结论**:
- 创建 API Key 时，如果请求中没有 `scope` 字段 → `scope` 字段保存为 `null`
- `scope: null` 的 API Key 拥有 **完全访问权限**（相当于 `"*"` scope）

### 3.3 OAuth Token 的 `scope: []` → 无访问权限

#### 3.3.1 前端的默认 Scope 处理

**代码位置**: `app/scenes/Login/OAuthAuthorize.tsx:56-71`

```typescript
function inputScopes(scope?: string): string[] {
  const defaultScopes = ["read", "write"];

  // Some clients don't send the scope parameter if it's empty, 
  // so we default to "read write".
  if (!scope) {
    return defaultScopes;  // ⭐ 前端默认: read + write
  }

  // 处理 Claude 发送的无效 scope
  if (scope === "claudeai") {
    return defaultScopes;
  }

  return scope.split(" ").filter(Boolean);
}
```

#### 3.3.2 后端的 Scope 验证

**代码位置**: `server/utils/oauth/OAuthInterface.ts:390-421`

```typescript
async validateScope(user, client, scope) {
  if (!scope?.length) {
    return [];  // ⭐ 空 scope 返回空数组
  }

  const scopes = Array.isArray(scope) ? scope : [scope];
  const validAccessScopes = Object.values(Scope); // ["read", "write", "create"]

  return scopes.every((s: string) => {
    // 验证逻辑...
  }) ? scopes : false;
}
```

**测试验证** (`OAuthInterface.test.ts:291-303`)：

```typescript
it("should return empty array for empty scope", async () => {
  const result = await OAuthInterface.validateScope(user, client, []);
  expect(result).toEqual([]);
});

it("should return empty array for empty scope", async () => {
  const result = await OAuthInterface.validateScope(user, client, undefined);
  expect(result).toEqual([]);
});
```

#### 3.3.3 空数组 Scope 的实际权限

**代码位置**: `shared/helpers/AuthenticationHelper.ts:36-68`

```typescript
public static canAccess = (path: string, scopes: string[]) => {
  // 通配符检查
  if (scopes.includes("*")) {
    return true;
  }
  
  // ... 路径解析 ...
  
  // ⭐ 空数组的 .some() 永远返回 false
  return scopes.some((scope) => {
    // 匹配逻辑...
  });
};
```

**OAuthAuthentication 的 canAccess** (`server/models/oauth/OAuthAuthentication.ts:143`)：

```typescript
canAccess = (path: string) => {
  // 特例 1: /oauth/revoke 始终允许
  if (path === "/oauth/revoke") {
    return true;
  }
  
  // 特例 2: /mcp 只要有任何 scope 就允许
  if (path.startsWith("/mcp")) {
    return this.scope.length > 0;  // ⭐ 空数组返回 false
  }
  
  // 常规检查
  return AuthenticationHelper.canAccess(path, this.scope);
};
```

### 3.4 空 Scope 权限边界总结

```
┌─────────────────────────────────────────────────────────────────┐
│                     空 Scope 权限对比                             │
├──────────────────────┬────────────────────┬─────────────────────┤
│                      │   API Key          │   OAuth Token       │
│                      │  (scope: null)     │   (scope: [])      │
├──────────────────────┼────────────────────┼─────────────────────┤
│ 常规 API 端点         │     ✅ 允许        │      ❌ 拒绝        │
│ /api/documents.info  │     ✅             │      ❌             │
│ /api/users.create    │     ✅             │      ❌             │
├──────────────────────┼────────────────────┼─────────────────────┤
│ /oauth/revoke 特例    │     ✅ 允许        │      ✅ 允许        │
│ (始终可访问)          │                    │                     │
├──────────────────────┼────────────────────┼─────────────────────┤
│ /mcp/* 特例           │     ✅ 允许        │      ❌ 拒绝        │
│ (需要 scope.length>0)│ (scope 为 null)    │ (空数组 length=0)   │
└──────────────────────┴────────────────────┴─────────────────────┘
```

---

## 四、特殊端点的权限逻辑

### 4.1 `/oauth/revoke` 端点

**适用场景**: 撤销 OAuth Access Token 或 Refresh Token

**权限特性**: **无需任何 Scope 即可访问**

#### 4.1.1 认证中间件中的特殊处理

**代码位置**: `server/models/oauth/OAuthAuthentication.ts:143`

```typescript
canAccess = (path: string) => {
  // Special case for the revoke endpoint, which is always allowed
  if (path === "/oauth/revoke") {
    return true;  // ⭐ 始终返回 true
  }
  // ... 其他检查
};
```

#### 4.1.2 端点实现

**代码位置**: `server/routes/oauth/index.ts:151-176`

```typescript
router.post(
  "/revoke",
  rateLimiter(RateLimiterStrategy.OneHundredPerHour),
  validate(T.TokenRevokeSchema),
  transaction(),
  async (ctx: APIContext<T.TokenRevokeReq>) => {
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

#### 4.1.3 设计原因

根据 **RFC 7009 (OAuth 2.0 Token Revocation)**：

> "If the server is unable to find a token or the token is already invalid, the server responds with a success status code since the desired goal of revoking the token has already been achieved."

此外，**撤销自己的 token 不应该要求额外权限**：
- 用户应该能够随时撤销已授权应用的访问
- 这是一种安全机制，防止 token 泄露后无法撤销

### 4.2 `/mcp/*` 端点（Model Context Protocol）

**适用场景**: MCP 服务端点，用于 AI 工具集成

**权限特性**: **只要有任何非空 Scope 即可访问**

#### 4.2.1 两处相同的实现

**API Key** (`server/models/ApiKey.ts:180-186`)：

```typescript
canAccess = (path: string) => {
  if (!this.scope) return true;
  
  // MCP endpoint access is allowed if the key has any valid scope.
  // Fine-grained scope enforcement happens at the tool level.
  if (path.startsWith("/mcp")) {
    return this.scope.length > 0;  // ⭐ 只要有 scope 就允许
  }
  
  return AuthenticationHelper.canAccess(path, this.scope);
};
```

**OAuth Token** (`server/models/oauth/OAuthAuthentication.ts:149-153`)：

```typescript
canAccess = (path: string) => {
  if (path === "/oauth/revoke") return true;
  
  // MCP endpoint access is allowed if the token has any valid scope.
  // Fine-grained scope enforcement happens at the tool level.
  if (path.startsWith("/mcp")) {
    return this.scope.length > 0;  // ⭐ 相同逻辑
  }
  
  return AuthenticationHelper.canAccess(path, this.scope);
};
```

#### 4.2.2 设计原因

代码注释明确说明了设计意图：

> "MCP endpoint access is allowed if the key has any valid scope. Fine-grained scope enforcement happens at the tool level."

**两层权限模型**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    MCP 权限模型                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  第一层: API Gateway 检查 (canAccess)                           │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  只要 token 有任何非空 scope → 允许进入 /mcp 端点         │  │
│  │  scope: ["read"] → ✅ 允许                                │  │
│  │  scope: ["documents:read"] → ✅ 允许                      │  │
│  │  scope: [] → ❌ 拒绝 (空数组)                             │  │
│  └─────────────────────────────────────────────────────────┘  │
│                              ↓                                  │
│  第二层: Tool 级别检查 (各 tools/*.ts)                          │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  每个 MCP tool 单独检查权限                                │  │
│  │  例: documents.list 工具需要 "read" 或 "documents:read"  │  │
│  │  例: documents.create 工具需要 "write" 或 "documents:*"  │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 4.2.3 Tool 级别的 Scope 检查示例

**代码位置**: `server/tools/documents.ts` (示意)

```typescript
// 实际检查逻辑
if (!AuthenticationHelper.canAccess("/api/documents.list", tokenScope)) {
  throw AuthorizationError("Insufficient scope");
}
```

#### 4.2.4 MCP 端点发现

**代码位置**: `server/routes/index.ts:115-174`

```typescript
// OAuth 授权服务器发现
router.get(
  ["/.well-known/oauth-authorization-server", "/.well-known/oauth-authorization-server/mcp"],
  async (ctx) => {
    ctx.body = {
      issuer: origin,
      authorization_endpoint: `${origin}/oauth/authorize`,
      token_endpoint: `${origin}/oauth/token`,
      revocation_endpoint: `${origin}/oauth/revoke`,
      registration_endpoint: `${origin}/oauth/register`, // DCR
      scopes_supported: ["read", "write"],
      // ...
    };
  }
);

// OAuth 受保护资源发现 (MCP)
router.get(
  ["/.well-known/oauth-protected-resource", "/.well-known/oauth-protected-resource/mcp"],
  async (ctx) => {
    ctx.body = {
      resource: `${origin}/mcp`,
      authorization_servers: [origin],
      scopes_supported: ["read", "write"],
      bearer_methods_supported: ["header"],
    };
  }
);
```

### 4.3 特殊端点权限对比表

| 端点 | 认证方式 | 权限条件 | 设计依据 |
|------|----------|----------|----------|
| `/oauth/revoke` | OAuth Token | **始终允许** | RFC 7009，安全撤销机制 |
| `/mcp/*` | API Key / OAuth | **scope.length > 0** | 两层权限模型，细粒度在 Tool 层 |
| `/api/*` (常规) | API Key / OAuth | AuthenticationHelper.canAccess() | 标准 Scope 检查 |

---

## 五、State 参数与 CSRF 校验关系

### 5.1 两层 CSRF 防护机制

Outline 在 OAuth 授权流程中实现了 **两层 CSRF 防护**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    OAuth 授权 CSRF 防护架构                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  第一层: OAuth 2.0 标准的 state 参数                             │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  • 第三方应用生成随机 state                                │  │
│  │  • 授权后通过 redirect_uri 带回                            │  │
│  │  • 应用验证 state 是否匹配                                 │  │
│  │  • 防止: 攻击者诱骗用户授权给攻击者的应用                   │  │
│  └─────────────────────────────────────────────────────────┘  │
│                              ↓                                  │
│  第二层: 应用层 CSRF Token (Double-Submit Cookie)              │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  • Outline 自己的 CSRF 防护                                │  │
│  │  • Cookie 存储 + Header/Form 提交                         │  │
│  │  • HMAC 签名验证                                           │  │
│  │  • 防止: 跨站表单提交请求                                  │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 第一层：OAuth State 参数（强制）

#### 5.2.1 前端强制校验

**代码位置**: `app/scenes/Login/OAuthAuthorize.tsx:129-134`

```typescript
const missingParams = [
  !clientId && "client_id",
  !redirectUri && "redirect_uri",
  !responseType && "response_type",
  !state && "state",  // ⭐ state 是必填参数
].filter(Boolean);

if (missingParams.length || clientError) {
  // 显示错误页面
  return (
    <Text as="p" type="secondary">
      {t("Required OAuth parameters are missing")}
      <Pre>
        {missingParams.map((param: string) => (
          <span key={param}>{param}<br /></span>
        ))}
      </Pre>
    </Text>
  );
}
```

#### 5.2.2 后端强制校验

**代码位置**: `server/routes/oauth/index.ts:64-74`

```typescript
const authorizationCode = await oauth.authorize(request, response, {
  allowEmptyState: false,  // ⭐ 强制要求 state 参数
  authorizationCodeLifetime: OAuthAuthorizationCode.authorizationCodeLifetime,
  authenticateHandler: {
    handle: async () => user,
  },
});
```

#### 5.2.3 State 参数的 CSRF 防护原理

```
┌──────────────────────────────────────────────────────────────────┐
│                    OAuth State 防止 CSRF 原理                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  正常流程:                                                        │
│  ┌──────────┐                                                    │
│  │ 第三方应用 │ ──1. 生成随机 state=abc123──→                    │
│  └────┬─────┘                                                    │
│       │                                                          │
│       │ 2. 重定向用户到:                                          │
│       │    /oauth/authorize?client_id=xxx&state=abc123          │
│       │                                                          │
│       │ 3. 用户授权后，重定向回:                                  │
│       │    redirect_uri?code=xxx&state=abc123                   │
│       │                                                          │
│       │ 4. 应用验证 state 匹配 → ✅ 用 code 换 token              │
│       ↓                                                          │
│  ┌──────────┐                                                    │
│  │ 第三方应用 │                                                    │
│  └──────────┘                                                    │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CSRF 攻击被阻止:                                                 │
│  ┌──────────┐                                                    │
│  │  攻击者   │ ──1. 准备自己的应用 client_id=evil                 │
│  └────┬─────┘                                                    │
│       │                                                          │
│       │ 2. 构造攻击页面:                                          │
│       │    <img src="https://outline.com/oauth/authorize?       │
│       │          client_id=evil&state=xyz" />                    │
│       │                                                          │
│       │ 3. 诱骗已登录用户访问                                     │
│       │                                                          │
│       │ 4. 如果没有 state 校验:                                   │
│       │    用户授权后 → code 发送到攻击者的 redirect_uri          │
│       │    攻击者用 code 换 token → 接管用户账户                  │
│       │                                                          │
│       │ 5. 有 state 校验时:                                       │
│       │    攻击者的 state=xyz 与第三方应用保存的 state=abc123     │
│       │    不匹配 → 应用拒绝接受 code → ❌ 攻击失败               │
│       ↓                                                          │
│  ┌──────────┐                                                    │
│  │  攻击者   │ → 攻击失败                                        │
│  └──────────┘                                                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 5.3 第二层：应用层 CSRF Token

#### 5.3.1 CSRF 中间件的保护条件

**代码位置**: `server/middlewares/csrf.ts:45-71`

```typescript
const shouldProtectRequest = (ctx: AppContext): boolean => {
  // 1. 非修改性方法 (GET, HEAD, OPTIONS) 不需要保护
  if (["GET", "HEAD", "OPTIONS"].includes(ctx.method)) {
    return false;
  }

  // 2. 非 cookie 认证不需要保护
  // (API Key / OAuth Token 通过 Header 传递，不受 CSRF 影响)
  const { transport } = parseAuthentication(ctx);
  if (transport !== "cookie") {
    return false;
  }

  // 3. API 路由的只读操作不需要保护
  if (ctx.originalUrl.startsWith("/api/")) {
    const canAccessWithReadOnly = AuthenticationHelper.canAccess(ctx.path, [
      Scope.Read,
    ]);
    if (canAccessWithReadOnly) {
      return false;
    }
  }

  // 其他情况需要 CSRF 保护
  return true;
};
```

#### 5.3.2 OAuth `/oauth/authorize` POST 的保护分析

```
请求: POST /oauth/authorize
      ├─ method: POST → 不是 GET/HEAD/OPTIONS
      ├─ transport: cookie → 用户已登录，使用 cookie 认证
      └─ path: /oauth/authorize → 不是 /api/ 开头
      
结论: ✅ 需要 CSRF 保护
```

#### 5.3.3 CSRF Token 验证流程

**代码位置**: `server/middlewares/csrf.ts:73-108`

```typescript
return async function verifyCSRFTokenMiddleware(ctx: AppContext, next: Next) {
  if (!shouldProtectRequest(ctx)) {
    await next();
    return;
  }

  // 1. 从 Cookie 获取 CSRF Token
  const cookieVal = ctx.cookies.get(CSRF.cookieName);
  if (!cookieVal) {
    throw CSRFError("CSRF token missing from cookie");
  }

  // 2. 从 Header 或 Form 字段获取 CSRF Token
  const inputVal =
    ctx.get(CSRF.headerName) || ctx.request.body?.[CSRF.fieldName];

  if (!inputVal) {
    throw CSRFError("CSRF token missing from request");
  }

  // 3. 验证两个 Token 都是有效的 HMAC 签名
  const { valid: cookieValid } = unbundleToken(cookieVal, env.SECRET_KEY);
  const { valid: inputValid } = unbundleToken(inputVal, env.SECRET_KEY);

  if (!cookieValid || !inputValid) {
    throw CSRFError("CSRF token invalid or malformed");
  }

  // 4. 验证两个 Token 匹配 (Double-Submit Cookie 模式)
  if (cookieVal !== inputVal) {
    throw CSRFError("CSRF token mismatch");
  }

  await next();
};
```

#### 5.3.4 CSRF Token 的生成

**代码位置**: `server/middlewares/csrf.ts:19-36`

```typescript
export function attachCSRFToken() {
  return async function attachCSRFTokenMiddleware(ctx: AppContext, next: Next) {
    // 只在安全方法上附加 Token
    if (["GET", "HEAD", "OPTIONS"].includes(ctx.method)) {
      const raw = generateRawToken(16);
      const bundled = bundleToken(raw, env.SECRET_KEY); // HMAC 签名

      // 设置 Cookie (非 HttpOnly，让前端 JS 可以读取)
      ctx.cookies.set(CSRF.cookieName, bundled, {
        httpOnly: false,  // ⭐ 前端可读取
        sameSite: "lax",
        domain: getCookieDomain(ctx.request.hostname, env.isCloudHosted),
      });
    }

    await next();
  };
}
```

### 5.4 两层防护的协同关系

| 防护层 | 保护对象 | 验证时机 | 防止的攻击 |
|--------|----------|----------|------------|
| **OAuth State** | 授权码 `code` | 第三方应用接收 `code` 时 | 攻击者获取用户授权的 `code` |
| **CSRF Token** | 表单提交 | 后端处理 `POST /oauth/authorize` 时 | 跨站请求伪造用户授权操作 |

### 5.5 为什么需要两层防护？

#### 场景 1: 没有 OAuth State

```
攻击者可以:
1. 创建恶意网站，包含:
   <img src="https://outline.com/oauth/authorize?client_id=evil-app" />
   
2. 诱骗已登录用户访问

3. 用户浏览器自动发送 cookie，Outline 认为是用户操作

4. 用户授权后，code 发送到 evil-app 的 redirect_uri

5. 攻击者用 code 换取 token → 接管用户账户

结果: ❌ 攻击成功
```

#### 场景 2: 没有应用层 CSRF Token

```
攻击者可以:
1. 创建恶意网站，包含自动提交的表单:
   <form action="https://outline.com/oauth/authorize" method="POST">
     <input type="hidden" name="client_id" value="evil-app" />
     <input type="hidden" name="state" value="attacker-state" />
     ...
   </form>
   <script>document.forms[0].submit();</script>
   
2. 诱骗已登录用户访问

3. 表单自动提交 POST 请求

4. 如果没有 CSRF Token 验证 → 请求被处理

5. 但此时 OAuth State 仍会阻止攻击:
   - 攻击者的 state 与用户应用保存的 state 不匹配
   - 应用拒绝接受 code

结果: ⚠️ 第一层防护仍然有效
```

#### 场景 3: 两层都有（实际情况）

```
攻击者需要同时突破:
1. CSRF Token → 需要获取用户的 CSRF Cookie (同源策略阻止)
2. OAuth State → 需要预测或获取用户应用的 state

结果: ✅ 攻击极难成功
```

### 5.6 CSRF 与 OAuth State 关键结论

| 问题 | 答案 |
|------|------|
| `allowEmptyState: false` 是什么？ | 强制要求 OAuth state 参数，防止授权码 CSRF |
| 为什么需要应用层 CSRF Token？ | 防止跨站表单提交 POST 请求 |
| 两层防护是冗余的吗？ | 不是。OAuth State 保护 `code` 的接收方，CSRF Token 保护授权操作本身 |
| 哪些请求需要 CSRF 保护？ | POST/PUT/DELETE + Cookie 认证的请求 |
| API 请求需要 CSRF 吗？ | 不需要，因为 API Key/OAuth Token 通过 Header 传递，不受同源策略限制 |

---

## 六、完整授权链路流程图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    OAuth 2.0 授权码完整流程                                │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  第三方应用   │         │   用户浏览器  │         │  Outline 服务 │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       │ 1. 生成随机 state       │                        │
       │    (如: "abc123")      │                        │
       │                        │                        │
       │ 2. 构造授权 URL         │                        │
       │───────────────────────>│                        │
       │ GET /oauth/authorize?  │                        │
       │   client_id=xxx&       │                        │
       │   redirect_uri=xxx&    │                        │
       │   response_type=code&  │                        │
       │   state=abc123&        │                        │
       │   scope=read            │                        │
       │                        │                        │
       │                        │ 3. 前端 React 路由捕获   │
       │                        │    显示授权确认页面      │
       │                        │    ⭐ 必填参数检查:       │
       │                        │    - client_id           │
       │                        │    - redirect_uri        │
       │                        │    - response_type       │
       │                        │    - state (强制)        │
       │                        │                        │
       │                        │ 4. 用户点击 "Authorize"  │
       │                        │    表单 POST 提交        │
       │                        │───────────────────────>│
       │                        │ POST /oauth/authorize   │
       │                        │ Content-Type: form-data  │
       │                        │ Body: {                  │
       │                        │   client_id,             │
       │                        │   redirect_uri,          │
       │                        │   response_type,         │
       │                        │   state=abc123,         │
       │                        │   scope=read,            │
       │                        │   csrf_token=xxx,        │ ⭐ CSRF Token
       │                        │   code_challenge...      │ ⭐ PKCE (可选)
       │                        │ }                        │
       │                        │                        │
       │                        │                        │ 5. 后端中间件检查
       │                        │                        │    ⭐ auth(): 验证 cookie
       │                        │                        │    ⭐ verifyCSRFToken():
       │                        │                        │      - 方法: POST (需保护)
       │                        │                        │      - 认证: cookie (需保护)
       │                        │                        │      - 验证 CSRF Token
       │                        │                        │
       │                        │                        │ 6. OAuth 库处理
       │                        │                        │    ⭐ allowEmptyState: false
       │                        │                        │       (强制 state 参数)
       │                        │                        │
       │                        │                        │ 7. 生成授权码
       │                        │                        │    - 插入 oauth_authorization_codes
       │                        │                        │    - code: "ol_ac_xxx" (哈希存储)
       │                        │                        │    - scope: ["read"]
       │                        │                        │    - state 透传 (不存储)
       │                        │                        │
       │                        │ 8. 302 重定向            │
       │                        │<───────────────────────│
       │                        │ Location:               │
       │                        │   redirect_uri?         │
       │                        │   code=ol_ac_xxx&       │
       │                        │   state=abc123          │
       │                        │                        │
       │ 9. 验证 state 匹配      │                        │
       │<───────────────────────│                        │
       │                        │                        │
       │ 10. 用 code 换 token   │                        │
       │──────────────────────────────────────────────>│
       │ POST /oauth/token      │                        │
       │ grant_type=authorization_code │                │
       │ code=ol_ac_xxx&        │                        │
       │ client_id=xxx&          │                        │
       │ client_secret=xxx&      │                        │ (机密客户端需要)
       │ redirect_uri=xxx&       │                        │
       │ code_verifier=xxx       │                        │ ⭐ PKCE (如果有)
       │                        │                        │
       │                        │                        │ 11. 验证授权码
       │                        │                        │     - 哈希匹配
       │                        │                        │     - 未过期
       │                        │                        │     - redirect_uri 匹配
       │                        │                        │     - PKCE 验证 (如果有)
       │                        │                        │
       │                        │                        │ 12. 生成 Token
       │                        │                        │     - access_token: "ol_at_xxx"
       │                        │                        │     - refresh_token: "ol_rt_xxx"
       │                        │                        │     - 插入 oauth_authentications
       │                        │                        │     - 删除授权码 (一次性)
       │                        │                        │
       │ 13. 返回 Token          │                        │
       │<───────────────────────────────────────────────│
       │ {                       │                        │
       │   access_token: ol_at_xxx, │                   │
       │   refresh_token: ol_rt_xxx, │                  │
       │   expires_in: 3600,    │                        │
       │   token_type: "Bearer", │                        │
       │   scope: "read"         │                        │
       │ }                       │                        │
       │                        │                        │
       │ 14. 使用 Access Token   │                        │
       │──────────────────────────────────────────────>│
       │ Authorization: Bearer   │                        │
       │ ol_at_xxx                │                        │
       │                        │                        │
       │                        │                        │ 15. 认证中间件检查
       │                        │                        │     ⭐ OAuthAuthentication.match()
       │                        │                        │     ⭐ 查找 accessTokenHash
       │                        │                        │     ⭐ 过期检查
       │                        │                        │     ⭐ canAccess() 检查:
       │                        │                        │       - 特例: /oauth/revoke 允许
       │                        │                        │       - 特例: /mcp/* 只要有 scope
       │                        │                        │       - 常规: AuthenticationHelper
       │                        │                        │
       │ 16. 返回 API 响应        │                        │
       │<───────────────────────────────────────────────│
       │                        │                        │
┌──────┴───────┐         ┌──────┴───────┐         ┌──────┴───────┐
│  第三方应用   │         │   用户浏览器  │         │  Outline 服务 │
└──────────────┘         └──────────────┘         └──────────────┘
```

---

## 七、关键代码位置索引

### 7.1 授权入口与请求方式

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 前端授权页面组件 | `app/scenes/Login/OAuthAuthorize.tsx` | 全文 |
| 前端 Scope 处理 | `app/scenes/Login/OAuthAuthorize.tsx` | 56-71 |
| 前端必填参数检查 | `app/scenes/Login/OAuthAuthorize.tsx` | 129-134 |
| 前端表单 POST 提交 | `app/scenes/Login/OAuthAuthorize.tsx` | 257-290 |
| 后端授权路由 | `server/routes/oauth/index.ts` | 41-88 |
| allowEmptyState 设置 | `server/routes/oauth/index.ts` | 66 |

### 7.2 空 Scope 权限

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| API Key canAccess (null 检查) | `server/models/ApiKey.ts` | 174-177 |
| OAuth Authentication canAccess | `server/models/oauth/OAuthAuthentication.ts` | 143-156 |
| AuthenticationHelper.canAccess | `shared/helpers/AuthenticationHelper.ts` | 36-68 |
| OAuthInterface.validateScope | `server/utils/oauth/OAuthInterface.ts` | 390-421 |
| OAuth Scope 验证测试 | `server/utils/oauth/OAuthInterface.test.ts` | 290-356 |

### 7.3 特殊端点

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| /oauth/revoke 路由 | `server/routes/oauth/index.ts` | 151-176 |
| /oauth/revoke 特例 (OAuth) | `server/models/oauth/OAuthAuthentication.ts` | 145-147 |
| /mcp 特例 (API Key) | `server/models/ApiKey.ts` | 180-186 |
| /mcp 特例 (OAuth) | `server/models/oauth/OAuthAuthentication.ts` | 149-153 |
| MCP 发现端点 | `server/routes/index.ts` | 115-174 |

### 7.4 CSRF 与 State

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| CSRF 中间件 | `server/middlewares/csrf.ts` | 全文 |
| shouldProtectRequest 逻辑 | `server/middlewares/csrf.ts` | 45-71 |
| CSRF Token 验证 | `server/middlewares/csrf.ts` | 73-108 |
| CSRF Token 生成 | `server/middlewares/csrf.ts` | 19-36 |
| OAuth 路由 CSRF 中间件应用 | `server/routes/oauth/index.ts` | 296 |

---

## 八、修正与补充要点总结

### 8.1 第一版报告的修正

| 原表述 | 修正后表述 | 依据 |
|--------|-----------|------|
| OAuth 授权入口是 POST | **GET + POST 两阶段** | 前端构造 GET 链接，表单提交 POST |
| 空 scope 默认无权限 | **API Key: null→完全访问; OAuth: []→无权限** | ApiKey.ts 检查 `!this.scope` |
| 未强调两层 CSRF | **OAuth State + 应用 CSRF Token 两层防护** | allowEmptyState: false + verifyCSRFToken 中间件 |

### 8.2 关键发现

1. **请求方式**: OAuth 授权采用 GET（显示页面）+ POST（后端处理）两阶段

2. **空 Scope 的歧义**:
   - `scope: null` (API Key) → 完全访问
   - `scope: []` (OAuth) → 无访问权限

3. **/oauth/revoke 的特殊性**:
   - 无需任何 scope 即可访问
   - 符合 RFC 7009 规范
   - 安全设计：用户应能随时撤销自己的 token

4. **/mcp 的两层权限模型**:
   - 第一层：Gateway 只要有非空 scope 就允许
   - 第二层：每个 Tool 单独检查精细权限

5. **CSRF 双重防护**:
   - **OAuth State**: 保护授权码不被误用
   - **应用 CSRF Token**: 保护授权操作不被伪造
   - 两者协同，缺一不可（虽然 OAuth State 是主要防线）
