# Outline API Token 与 OAuth 授权链路深度分析 (R3)

## 关键修正摘要

本报告基于对代码的**逐行核查**，修正了前两版中的关键错误：

| 表述 (R1/R2) | 实际情况 (R3) | 影响 |
|-------------|---------------|------|
| `/oauth/revoke` 经过 `auth()` 中间件 | **不经过 `auth()` 中间件** | 认证链路完全不同 |
| `/oauth/revoke` 通过 `canAccess` 特例允许 | **直接查询数据库，不验证请求者身份** | 安全模型理解错误 |
| CSRF: "POST + Cookie" | **POST + Cookie + (非 `/api/` 或 `/api/` 写操作)** | 保护范围理解错误 |
| `/api` 写操作 = 所有 POST | **写操作 = `canAccess(path, ["read"])` 返回 `false`** | 具体哪些端点需要保护 |

---

## 一、`/oauth/revoke` 真实认证链路深度分析

### 1.1 路由定义对比：关键差异

**代码位置**: `server/routes/oauth/index.ts`

```typescript
// ========== /authorize 路由 ==========
router.post(
  "/authorize",
  rateLimiter(RateLimiterStrategy.OneHundredPerHour),
  auth(),  // ⭐ 使用 auth() 中间件
  async (ctx) => {
    const { user } = ctx.state.auth;  // 从 auth() 中间件获取用户
    // ...
  }
);

// ========== /revoke 路由 ==========
router.post(
  "/revoke",
  rateLimiter(RateLimiterStrategy.OneHundredPerHour),
  validate(T.TokenRevokeSchema),
  transaction(),
  // ⚠️ 注意：没有 auth() 中间件！
  async (ctx: APIContext<T.TokenRevokeReq>) => {
    const { token } = ctx.input.body;  // 直接从 body 取 token

    // 直接查询数据库，不经过 auth 中间件的验证
    if (OAuthAuthentication.match(token)) {
      const accessToken = await OAuthAuthentication.findByAccessToken(token);
      await accessToken?.destroyWithCtx(ctx);
    }

    if (OAuthAuthentication.matchRefreshToken(token)) {
      const refreshToken = await OAuthAuthentication.findByRefreshToken(token);
      await refreshToken?.destroyWithCtx(ctx);
    }

    // RFC 7009 §2.2: 无效 token 不返回错误
    ctx.body = { success: true };
  }
);
```

### 1.2 完整认证链路对比图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    /oauth/authorize vs /oauth/revoke                      │
└──────────────────────────────────────────────────────────────────────────┘

╔══════════════════════════════════════════════════════════════════════════╗
║                        /oauth/authorize (使用 auth())                      ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                             ║
║  POST /oauth/authorize                                                      ║
║       │                                                                     ║
║       ▼                                                                     ║
║  ┌─────────────────────────────────────────────────────────────────────┐  ║
║  │                    auth() 中间件 (必须)                               │  ║
║  │  ┌───────────────────────────────────────────────────────────────┐  │  ║
║  │  │ 1. parseAuthentication()                                       │  │  ║
║  │  │    - 检查 Authorization header                                  │  │  ║
║  │  │    - 检查 body.token                                           │  │  ║
║  │  │    - 检查 query.token                                          │  │  ║
║  │  │    - 检查 cookie.accessToken ← ⭐ 用户已登录，走这个分支      │  │  ║
║  │  │                                                               │  │  ║
║  │  │    transport = "cookie"                                        │  │  ║
║  │  └───────────────────────────────────────────────────────────────┘  │  ║
║  │                              │                                        │  ║
║  │                              ▼                                        │  ║
║  │  ┌───────────────────────────────────────────────────────────────┐  │  ║
║  │  │ 2. validateAuthentication()                                    │  │  ║
║  │  │    - transport === "cookie" → 使用 JWT 验证                   │  │  ║
║  │  │    - 解析 JWT，获取 user                                      │  │  ║
║  │  │    - 检查用户是否被禁用                                       │  │  ║
║  │  │    - 检查 role/type 限制                                      │  │  ║
║  │  │                                                               │  │  ║
║  │  │    ctx.state.auth = { user, token, type, service, scope }    │  │  ║
║  │  └───────────────────────────────────────────────────────────────┘  │  ║
║  └─────────────────────────────────────────────────────────────────────┘  ║
║       │                                                                     ║
║       ▼                                                                     ║
║  路由处理函数:                                                              ║
║  - 从 ctx.state.auth 获取 user                                             ║
║  - authorize(user, "read", client) → 权限检查                             ║
║  - 生成授权码                                                               ║
║                                                                             ║
╚══════════════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════════════╗
║                        /oauth/revoke (不使用 auth())                        ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                             ║
║  POST /oauth/revoke                                                         ║
║  Content-Type: application/json                                             ║
║  Body: { "token": "ol_at_xxx" }                                             ║
║       │                                                                     ║
║       ▼                                                                     ║
║  ┌─────────────────────────────────────────────────────────────────────┐  ║
║  │              verifyCSRFToken() 中间件 (全局应用)                      │  ║
║  │  ┌───────────────────────────────────────────────────────────────┐  │  ║
║  │  │ shouldProtectRequest():                                        │  │  ║
║  │  │ 1. method = POST → 可能需要保护                                │  │  ║
║  │  │ 2. parseAuthentication():                                      │  │  ║
║  │  │    - 检查 Authorization header → 无                            │  │  ║
│  │  │    - 检查 body.token → ⭐ 有！                                  │  │  ║
│  │  │    - transport = "body"                                         │  │  ║
│  │  │ 3. transport !== "cookie" → ⭐ 不需要 CSRF 保护                 │  │  ║
│  │  └───────────────────────────────────────────────────────────────┘  │  ║
│  └─────────────────────────────────────────────────────────────────────┘  ║
║       │                                                                     ║
║       ▼                                                                     ║
║  路由处理函数 (直接执行，不经过 auth()):                                    ║
║  ┌─────────────────────────────────────────────────────────────────────┐  ║
║  │  const { token } = ctx.input.body;  // 直接从 body 取              │  ║
║  │                                                                     │  ║
║  │  // 格式检查 (只检查前缀)                                           │  ║
║  │  if (OAuthAuthentication.match(token)) {                            │  ║
║  │    // match(): token.startsWith("ol_at_")                          │  ║
║  │                                                                     │  ║
║  │    // 直接查询数据库                                                │  ║
║  │    const accessToken = await OAuthAuthentication.findByAccessToken(token); │  ║
║  │    // 查询: WHERE accessTokenHash = SHA256(token)                  │  ║
│  │                                                                     │  ║
│  │    // 直接删除，不验证"请求者身份"                                   │  ║
│  │    await accessToken?.destroyWithCtx(ctx);                          │  ║
│  │  }                                                                  │  ║
│  │                                                                     │  ║
│  │  // 同样处理 refresh token                                          │  ║
│  │  if (OAuthAuthentication.matchRefreshToken(token)) {                │  ║
│  │    // ...                                                           │  ║
│  │  }                                                                  │  ║
│  └─────────────────────────────────────────────────────────────────────┘  ║
║                                                                             ║
╚══════════════════════════════════════════════════════════════════════════╝
```

### 1.3 Token 传递方式与认证优先级

**代码位置**: `server/middlewares/authentication.ts:84-133`

```typescript
export function parseAuthentication(ctx: AppContext): AuthInput {
  const authorizationHeader = ctx.request.get("authorization");

  // 优先级 1: Authorization header (Bearer token)
  if (authorizationHeader) {
    const parts = authorizationHeader.split(" ");
    if (parts.length === 2 && /^Bearer$/i.test(parts[0])) {
      return {
        token: parts[1],
        transport: "header",  // ⭐
      };
    }
  }
  // 优先级 2: Body 中的 token 字段
  else if (
    ctx.request.body &&
    typeof ctx.request.body === "object" &&
    "token" in ctx.request.body
  ) {
    return {
      token: String(ctx.request.body.token),
      transport: "body",  // ⭐ /oauth/revoke 走这个分支
    };
  }
  // 优先级 3: Query 中的 token
  else if (ctx.request.query?.token) {
    return {
      token: String(ctx.request.query.token),
      transport: "query",  // ⭐
    };
  }
  // 优先级 4: Cookie 中的 accessToken
  else {
    const accessToken = ctx.cookies.get("accessToken");
    if (accessToken) {
      return {
        token: accessToken,
        transport: "cookie",  // ⭐ 应用内登录走这个分支
      };
    }
  }

  return {
    token: undefined,
    transport: undefined,
  };
}
```

### 1.4 `/oauth/revoke` 安全设计分析

#### 为什么不需要验证请求者身份？

根据 **RFC 7009 (OAuth 2.0 Token Revocation)** 的设计理念：

> "The token revocation endpoint can be used by clients to invalidate an access token or refresh token. Since the tokens themselves are secrets, possession of the token is considered sufficient authorization for revocation."

**Outline 的实现逻辑**：

| 安全考虑 | 实现方式 |
|----------|----------|
| Token 本身是机密 | `ol_at_` + 64 位随机字符 (16^64 种可能) |
| 暴力破解不可行 | 64 位随机字符 + 速率限制 (`OneHundredPerHour`) |
| 跨域攻击防护 | 即使攻击者构造 CSRF 表单，也无法获取有效的 token 值 |
| 泄露后紧急撤销 | 用户/应用能快速撤销泄露的 token，无需额外认证 |

#### 可能的攻击面分析

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      /oauth/revoke 攻击面分析                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  攻击场景 1: 攻击者知道某个 token，尝试撤销它                              │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ 攻击者: POST /oauth/revoke { token: "ol_at_xxx" }                  │ │
│  │                                                                    │ │
│  │ 结果: ✅ token 被撤销                                               │ │
│  │                                                                    │ │
│  │ 分析: 这是预期行为。如果攻击者已经获取了 token，说明 token 已经     │ │
│  │      泄露，此时最安全的做法是让它失效。                             │ │
│  │      (合法持有者也可以随时撤销自己的 token)                          │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  攻击场景 2: CSRF 攻击 (攻击者构造跨站表单)                               │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ 攻击者的网站:                                                        │ │
│  │ <form action="https://outline.com/oauth/revoke" method="POST">   │ │
│  │   <input type="hidden" name="token" value="???" />                │ │
│  │ </form>                                                             │ │
│  │ <script>document.forms[0].submit();</script>                       │ │
│  │                                                                    │ │
│  │ 问题: 攻击者如何获取有效的 token 值？                                │ │
│  │                                                                    │ │
│  │ 结果: ❌ 攻击失败                                                   │ │
│  │                                                                    │ │
│  │ 分析: 1. 攻击者无法读取跨域 Cookie (Same Origin Policy)            │ │
│  │      2. 攻击者无法猜测 64 位随机 token                             │ │
│  │      3. 即使请求发送了，token 无效也不会报错 (RFC 7009)            │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  攻击场景 3: 恶意撤销用户的 token                                         │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ 前提: 攻击者已经获取了用户的 token                                   │ │
│  │                                                                    │ │
│  │ 结果: ⚠️ token 被撤销                                               │ │
│  │                                                                    │ │
│  │ 分析: 这实际上是一种"止损"机制。如果攻击者已经获取了 token，         │ │
│  │      他们可以直接使用 token 访问 API，撤销只是让它失效。             │ │
│  │      真正的问题是 token 如何泄露的，而不是撤销机制本身。             │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.5 `/oauth/revoke` 权限对比表 (修正版)

| 特性 | R1/R2 表述 | R3 实际情况 |
|------|-----------|-------------|
| 是否经过 `auth()` 中间件 | ✅ 是 | ❌ **否** |
| token 传递方式 | Authorization header | **`body.token` 字段** |
| 认证方式 | OAuthAuthentication.canAccess() 特例 | **直接查询数据库，不验证请求者身份** |
| CSRF 保护 | 需要 | **不需要** (transport = "body") |
| 无效 token 处理 | 返回错误 | **返回成功** (RFC 7009 §2.2) |

---

## 二、CSRF 实际适用条件深度分析

### 2.1 CSRF 中间件全局应用

**代码位置**: `server/routes/oauth/index.ts:292-297`

```typescript
app.use(requestTracer());
app.use(oauthErrorHandler());
app.use(bodyParser());
app.use(apiContext());
app.use(verifyCSRFToken());  // ⭐ 全局应用
app.use(router.routes());
```

**关键**：`verifyCSRFToken()` 是全局应用的，但 `shouldProtectRequest()` 函数决定是否实际执行保护。

### 2.2 `shouldProtectRequest()` 完整逻辑

**代码位置**: `server/middlewares/csrf.ts:45-71`

```typescript
const shouldProtectRequest = (ctx: AppContext): boolean => {
  // 条件 1: 方法必须是"可能修改状态"的方法
  if (["GET", "HEAD", "OPTIONS"].includes(ctx.method)) {
    return false;  // 只读方法，不需要保护
  }

  // 条件 2: 认证方式必须是 cookie
  const { transport } = parseAuthentication(ctx);
  if (transport !== "cookie") {
    return false;  // 非 cookie 认证，不受 CSRF 攻击
  }

  // 条件 3: 如果是 /api/ 路由，必须是"写操作"
  if (ctx.originalUrl.startsWith("/api/")) {
    const canAccessWithReadOnly = AuthenticationHelper.canAccess(ctx.path, [
      Scope.Read,  // 用 "read" scope 测试
    ]);

    // 如果用只读 scope 就能访问，说明是只读操作，不需要 CSRF
    if (canAccessWithReadOnly) {
      return false;
    }
  }

  // 所有条件都满足：需要 CSRF 保护
  return true;
};
```

### 2.3 三层保护逻辑图解

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    CSRF 保护判断逻辑 (三层)                                │
└──────────────────────────────────────────────────────────────────────────┘

                    请求到达
                        │
                        ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 第一层: 方法检查                                                            │
│ ┌──────────────────────────────────────────────────────────────────────┐ │
│ │ 方法是 GET / HEAD / OPTIONS ?                                          │ │
│ │                                                                       │ │
│ │ YES ──► 不需要 CSRF 保护 ──► 直接放行                                  │ │
│ │                                                                       │ │
│ │ NO ──► 继续检查                                                        │ │
│ └──────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
                        │
                        ▼ (POST / PUT / DELETE)
┌──────────────────────────────────────────────────────────────────────────┐
│ 第二层: 认证方式检查                                                        │
│ ┌──────────────────────────────────────────────────────────────────────┐ │
│ │ parseAuthentication() 返回的 transport 是什么？                       │ │
│ │                                                                       │ │
│ │ transport = "header"  ──► API Key / OAuth Token ──► 不需要保护      │ │
│ │ transport = "body"    ──► /oauth/revoke 等 ──► 不需要保护          │ │
│ │ transport = "query"   ──► 查询参数传递 ──► 不需要保护               │ │
│ │ transport = "cookie"  ──► 继续检查                                   │ │
│ │ transport = undefined ──► 未认证 ──► 不需要保护                     │ │
│ └──────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
                        │
                        ▼ (transport = "cookie")
┌──────────────────────────────────────────────────────────────────────────┐
│ 第三层: /api/ 路由的只读操作检查                                           │
│ ┌──────────────────────────────────────────────────────────────────────┐ │
│ │ 路径是否以 /api/ 开头？                                                │ │
│ │                                                                       │ │
│ │ NO ──► 非 API 路由 ──► 需要 CSRF 保护                                 │ │
│ │       (如 /oauth/authorize)                                          │ │
│ │                                                                       │ │
│ │ YES ──► 检查: canAccess(path, ["read"]) ?                            │ │
│ │                                                                       │ │
│ │       canAccess = true  ──► 只读操作 ──► 不需要保护                  │ │
│ │       canAccess = false ──► 写操作 ──► 需要保护                      │ │
│ └──────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.4 如何定义"写操作"

**核心逻辑**: `AuthenticationHelper.canAccess(path, ["read"])` 返回 `false` 即为写操作。

**代码位置**: `shared/helpers/AuthenticationHelper.ts:12-68`

```typescript
// 方法到 scope 的映射
private static methodToScope = {
  create: Scope.Create,    // "create"
  config: Scope.Read,       // "read"
  list: Scope.Read,         // "read"
  info: Scope.Read,         // "read"
  search: Scope.Read,       // "read"
  documents: Scope.Read,    // "read"
  drafts: Scope.Read,       // "read"
  viewed: Scope.Read,       // "read"
  export: Scope.Read,       // "read"
  // ⚠️ 未列出的方法默认为 Scope.Write ("write")
};

// canAccess 核心判断
public static canAccess = (path: string, scopes: string[]) => {
  // 解析路径: /api/documents.create → namespace="documents", method="create"
  const resource = path.split("/").pop() ?? "";
  const [namespace, method] = resource.split(".");

  return scopes.some((scope) => {
    // 对于 scope = "read"，解析为 ["*", "read"]
    const [scopeNamespace, scopeMethod] = scope.match(/[:.]/g)
      ? scope.replace("/api/", "").split(/[:.]/g)
      : ["*", scope];

    // 非路由格式的 scope 判断
    return (
      (namespace === scopeNamespace || scopeNamespace === "*") &&
      (
        scopeMethod === Scope.Write ||  // "write" 包含所有权限
        this.methodToScope[method] === scopeMethod  // 精确匹配
      )
    );
  });
};
```

### 2.5 操作类型分类表

| API 端点 | method | methodToScope[method] | `canAccess(path, ["read"])` | 操作类型 | CSRF 保护 |
|----------|--------|----------------------|------------------------------|----------|-----------|
| `/api/documents.list` | `list` | `Scope.Read` | ✅ `true` | 只读 | ❌ 不需要 |
| `/api/documents.info` | `info` | `Scope.Read` | ✅ `true` | 只读 | ❌ 不需要 |
| `/api/documents.search` | `search` | `Scope.Read` | ✅ `true` | 只读 | ❌ 不需要 |
| `/api/documents.export` | `export` | `Scope.Read` | ✅ `true` | 只读 | ❌ 不需要 |
| `/api/documents.create` | `create` | `Scope.Create` | ❌ `false` | 写操作 | ✅ 需要 |
| `/api/documents.update` | `update` | `undefined` (默认 `Write`) | ❌ `false` | 写操作 | ✅ 需要 |
| `/api/documents.delete` | `delete` | `undefined` (默认 `Write`) | ❌ `false` | 写操作 | ✅ 需要 |
| `/api/users.create` | `create` | `Scope.Create` | ❌ `false` | 写操作 | ✅ 需要 |
| `/api/apiKeys.delete` | `delete` | `undefined` (默认 `Write`) | ❌ `false` | 写操作 | ✅ 需要 |

### 2.6 测试用例验证

**代码位置**: `shared/helpers/AuthenticationHelper.test.ts:94-134`

```typescript
describe("global access scopes", () => {
  it("read", async () => {
    const scopes = ["read"];
    
    // 只读操作 → true
    expect(canAccess("/api/documents.info", scopes)).toBe(true);
    expect(canAccess("/api/documents.list", scopes)).toBe(true);
    expect(canAccess("/api/users.info", scopes)).toBe(true);
    
    // 写操作 → false
    expect(canAccess("/api/documents.create", scopes)).toBe(false);
    expect(canAccess("/api/documents.update", scopes)).toBe(false);
    expect(canAccess("/api/users.create", scopes)).toBe(false);
  });
});
```

### 2.7 各端点 CSRF 保护情况汇总

| 端点 | 方法 | 认证方式 | 路径类型 | 操作类型 | CSRF 保护 |
|------|------|----------|----------|----------|-----------|
| `/oauth/authorize` | POST | Cookie | 非 `/api/` | - | ✅ **需要** |
| `/oauth/revoke` | POST | Body (`body.token`) | 非 `/api/` | - | ❌ **不需要** |
| `/oauth/token` | POST | 无 (client_credentials) | 非 `/api/` | - | ❌ **不需要** |
| `/api/documents.list` | POST | Cookie | `/api/` | 只读 | ❌ 不需要 |
| `/api/documents.info` | POST | Cookie | `/api/` | 只读 | ❌ 不需要 |
| `/api/documents.create` | POST | Cookie | `/api/` | 写操作 | ✅ **需要** |
| `/api/documents.update` | POST | Cookie | `/api/` | 写操作 | ✅ **需要** |
| `/api/documents.delete` | POST | Cookie | `/api/` | 写操作 | ✅ **需要** |

---

## 三、权限对比表修正版

### 3.1 空 Scope 权限对比

| 场景 | 数据类型 | API Key (`scope: null`) | OAuth Token (`scope: []`) |
|------|----------|-------------------------|---------------------------|
| 常规 API 端点 | | | |
| `/api/documents.info` | | ✅ 允许 (null → true) | ❌ 拒绝 (`[].some()` → false) |
| `/api/documents.create` | | ✅ 允许 | ❌ 拒绝 |
| `/api/users.update` | | ✅ 允许 | ❌ 拒绝 |
| 特殊端点 | | | |
| `/oauth/revoke` | | ✅ 允许 (无 auth 中间件) | ✅ 允许 (无 auth 中间件) |
| `/mcp/*` | | ✅ 允许 (null → true) | ❌ 拒绝 (`[].length === 0`) |

### 3.2 认证方式与 CSRF 关系

| 认证方式 | transport | CSRF 保护 | 原因 |
|----------|-----------|-----------|------|
| Cookie (应用内登录) | `"cookie"` | ✅ 需要 | 浏览器自动发送 cookie，易受 CSRF |
| Authorization Header | `"header"` | ❌ 不需要 | 跨域请求无法设置自定义 header |
| Body token 字段 | `"body"` | ❌ 不需要 | 攻击者无法获取有效 token 值 |
| Query token | `"query"` | ❌ 不需要 | 攻击者无法获取有效 token 值 |

### 3.3 各 OAuth 端点完整对比

| 端点 | `auth()` 中间件 | 认证方式 | CSRF 保护 | 主要功能 |
|------|-----------------|----------|-----------|----------|
| `/oauth/authorize` | ✅ 使用 | Cookie | ✅ 需要 | 用户授权，生成授权码 |
| `/oauth/token` | ❌ 不使用 | Client ID/Secret | ❌ 不需要 | 授权码换 token，刷新 token |
| `/oauth/revoke` | ❌ 不使用 | Body token | ❌ 不需要 | 撤销 token |
| `/oauth/register` | ❌ 不使用 | 无 (DCR) | ❌ 不需要 | 动态注册客户端 |

---

## 四、完整授权链路流程图 (修正版)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    OAuth 2.0 授权码完整流程 (修正版)                       │
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
       │                        │    OAuthAuthorize 组件   │
       │                        │                        │
       │                        │ 4. 必填参数检查          │
       │                        │    ⭐ state 是必填的     │
       │                        │                        │
       │                        │ 5. 用户点击 "Authorize"  │
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
       │                        │                        │ 6. CSRF 检查
       │                        │                        │    ═══════════
       │                        │                        │    方法: POST
       │                        │                        │    认证: Cookie (用户已登录)
       │                        │                        │    路径: 非 /api/ 开头
       │                        │                        │    ──► ⭐ 需要 CSRF 保护
       │                        │                        │
       │                        │                        │ 7. auth() 中间件
       │                        │                        │    ═══════════
       │                        │                        │    - 从 Cookie 解析 JWT
       │                        │                        │    - 验证用户身份
       │                        │                        │    - ctx.state.auth = { user, ... }
       │                        │                        │
       │                        │                        │ 8. OAuth 库处理
       │                        │                        │    ═══════════
       │                        │                        │    ⭐ allowEmptyState: false
       │                        │                        │       (强制 state 参数)
       │                        │                        │
       │                        │                        │ 9. 生成授权码
       │                        │                        │    - 插入 oauth_authorization_codes
       │                        │                        │    - state 透传 (不存储)
       │                        │                        │
       │                        │ 10. 302 重定向           │
       │                        │<───────────────────────│
       │                        │ Location:               │
       │                        │   redirect_uri?         │
       │                        │   code=ol_ac_xxx&       │
       │                        │   state=abc123          │
       │                        │                        │
       │ 11. 验证 state 匹配    │                        │
       │<───────────────────────│                        │
       │                        │                        │
       │ 12. 用 code 换 token   │                        │
       │──────────────────────────────────────────────>│
       │ POST /oauth/token      │                        │
       │ grant_type=authorization_code │                │
       │ code=ol_ac_xxx&        │                        │
       │ client_id=xxx&          │                        │
       │ client_secret=xxx&      │                        │ (机密客户端需要)
       │ redirect_uri=xxx&       │                        │
       │ code_verifier=xxx       │                        │ ⭐ PKCE (如果有)
       │                        │                        │
       │                        │                        │ 13. CSRF 检查
       │                        │                        │    ═══════════
       │                        │                        │    方法: POST
       │                        │                        │    认证: 无 (client_credentials)
       │                        │                        │    ──► ❌ 不需要 CSRF 保护
       │                        │                        │
       │                        │                        │ 14. 验证授权码
       │                        │                        │    - 哈希匹配
       │                        │                        │    - 未过期
       │                        │                        │    - redirect_uri 匹配
       │                        │                        │    - PKCE 验证 (如果有)
       │                        │                        │
       │                        │                        │ 15. 生成 Token
       │                        │                        │    - access_token: ol_at_xxx
       │                        │                        │    - refresh_token: ol_rt_xxx
       │                        │                        │    - 删除授权码 (一次性)
       │                        │                        │
       │ 16. 返回 Token          │                        │
       │<───────────────────────────────────────────────│
       │ {                       │                        │
       │   access_token: ol_at_xxx, │                   │
       │   refresh_token: ol_rt_xxx, │                  │
       │   expires_in: 3600,    │                        │
       │   token_type: "Bearer", │                        │
       │   scope: "read"         │                        │
       │ }                       │                        │
       │                        │                        │
       │ 17. 使用 Access Token   │                        │
       │──────────────────────────────────────────────>│
       │ Authorization: Bearer   │                        │
       │ ol_at_xxx                │                        │
       │                        │                        │
       │                        │                        │ 18. 认证中间件检查
       │                        │                        │    ═══════════
       │                        │                        │    - OAuthAuthentication.match()
       │                        │                        │    - 查找 accessTokenHash
       │                        │                        │    - 过期检查
       │                        │                        │    - canAccess() 检查
       │                        │                        │
       │ 19. 返回 API 响应        │                        │
       │<───────────────────────────────────────────────│
       │                        │                        │
       │                        │                        │
       │ 20. 撤销 Token (可选)    │                        │
       │──────────────────────────────────────────────>│
       │ POST /oauth/revoke      │                        │
       │ Content-Type: application/json │                 │
       │ Body: { token: "ol_at_xxx" } │                 │
       │                        │                        │
       │                        │                        │ 21. CSRF 检查
       │                        │                        │    ═══════════
       │                        │                        │    方法: POST
       │                        │                        │    认证: Body (body.token)
       │                        │                        │    transport = "body" !== "cookie"
       │                        │                        │    ──► ❌ 不需要 CSRF 保护
       │                        │                        │
       │                        │                        │ 22. 直接查询数据库
       │                        │                        │    ═══════════
       │                        │                        │    ⚠️ 不经过 auth() 中间件
       │                        │                        │
       │                        │                        │    - OAuthAuthentication.match(token)
       │                        │                        │      检查前缀: "ol_at_" 或 "ol_rt_"
       │                        │                        │
       │                        │                        │    - OAuthAuthentication.findByAccessToken(token)
       │                        │                        │      WHERE accessTokenHash = SHA256(token)
       │                        │                        │
       │                        │                        │    - destroyWithCtx(ctx)
       │                        │                        │      软删除，记录 deletedAt
       │                        │                        │
       │ 23. 返回成功 (RFC 7009) │                        │
       │<───────────────────────────────────────────────│
       │ { success: true }       │                        │
       │ (无效 token 也返回成功)  │                        │
       │                        │                        │
┌──────┴───────┐         ┌──────┴───────┐         ┌──────┴───────┐
│  第三方应用   │         │   用户浏览器  │         │  Outline 服务 │
└──────────────┘         └──────────────┘         └──────────────┘
```

---

## 五、关键代码位置索引 (修正版)

### 5.1 `/oauth/revoke` 相关

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| `/revoke` 路由定义 | `server/routes/oauth/index.ts` | 151-176 |
| `/authorize` 路由定义 (对比) | `server/routes/oauth/index.ts` | 41-88 |
| `parseAuthentication()` 优先级 | `server/middlewares/authentication.ts` | 84-133 |
| `TokenRevokeSchema` 定义 | `server/routes/oauth/schema.ts` | 19-26 |
| `/revoke` 测试用例 | `server/routes/oauth/index.test.ts` | 13-52 |
| `OAuthAuthentication.match()` | `server/models/oauth/OAuthAuthentication.ts` | 167-169 |
| `OAuthAuthentication.findByAccessToken()` | `server/models/oauth/OAuthAuthentication.ts` | 190-212 |

### 5.2 CSRF 相关

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| `verifyCSRFToken()` 中间件 | `server/middlewares/csrf.ts` | 41-108 |
| `shouldProtectRequest()` 逻辑 | `server/middlewares/csrf.ts` | 45-71 |
| `AuthenticationHelper.methodToScope` | `shared/helpers/AuthenticationHelper.ts` | 12-22 |
| `AuthenticationHelper.canAccess()` | `shared/helpers/AuthenticationHelper.ts` | 36-68 |
| CSRF 测试用例 (canAccess) | `shared/helpers/AuthenticationHelper.test.ts` | 94-134 |
| OAuth 路由应用 CSRF 中间件 | `server/routes/oauth/index.ts` | 296 |

### 5.3 认证中间件相关

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| `auth()` 中间件主函数 | `server/middlewares/authentication.ts` | 37-76 |
| `validateAuthentication()` | `server/middlewares/authentication.ts` | 135-278 |
| OAuth Token 认证流程 | `server/middlewares/authentication.ts` | 156-199 |
| API Key 认证流程 | `server/middlewares/authentication.ts` | 200-241 |

---

## 六、关键修正总结

### 6.1 前两版报告的主要错误

| 错误表述 | 正确表述 | 影响程度 |
|----------|----------|----------|
| `/oauth/revoke` 经过 `auth()` 中间件 | **不经过 `auth()` 中间件** | 🔴 严重 |
| `/oauth/revoke` 通过 `canAccess` 特例允许 | **直接查询数据库，不验证请求者身份** | 🔴 严重 |
| CSRF 保护条件: "POST + Cookie" | **POST + Cookie + (非 `/api/` 或 `/api/` 写操作)** | 🟡 中等 |
| `/api` 所有 POST 都需要 CSRF | **只有 `canAccess(path, ["read"])` 返回 `false` 的才需要** | 🟡 中等 |

### 6.2 核心安全模型理解

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         核心安全模型理解                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 关于认证                                                              │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │ /oauth/authorize ──► 用户必须已登录 (Cookie) ──► auth() 中间件  │ │
│     │ /oauth/token     ──► 客户端凭证 (client_id/secret) ──► 无 auth  │ │
│     │ /oauth/revoke    ──► 知道 token 即可撤销 ──► 无 auth             │ │
│     └─────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  2. 关于 CSRF                                                            │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │ CSRF 攻击的前提:                                                  │ │
│     │ - 浏览器自动发送 Cookie (Same Origin Policy 的"例外")            │ │
│     │ - 攻击者无法读取/修改跨域响应                                     │ │
│     │                                                                  │ │
│     │ 所以:                                                             │ │
│     │ - Cookie 认证 ──► 需要 CSRF 保护                                 │ │
│     │ - Header/Body/Query 认证 ──► 不需要 CSRF 保护                   │ │
│     │   (攻击者无法获取有效的 token 值)                                 │ │
│     └─────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  3. 关于 /api/ 路由的 CSRF                                               │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │ 为什么只读操作不需要 CSRF？                                       │ │
│     │                                                                  │ │
│     │ CSRF 的危害: 攻击者诱导用户执行"非预期的操作"                      │ │
│     │                                                                  │ │
│     │ - 只读操作 (list, info, search)                                  │ │
│     │   即使被 CSRF 触发，也不会修改数据                                │ │
│     │   攻击者无法读取响应 (跨域)                                       │ │
│     │   ──► 实际危害很小                                                │ │
│     │                                                                  │ │
│     │ - 写操作 (create, update, delete)                                │ │
│     │   被 CSRF 触发会修改/删除数据                                     │ │
│     │   ──► 需要保护                                                    │ │
│     └─────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  4. 关于 /oauth/revoke 的安全                                           │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │ RFC 7009 的设计哲学:                                              │ │
│     │                                                                  │ │
│     │ "Token 本身就是机密，持有 token 就有权撤销它"                      │ │
│     │                                                                  │ │
│     │ 原因:                                                            │ │
│     │ - Token 是 64 位随机字符串，无法猜测                              │ │
│     │ - 攻击者无法通过 CSRF 获取有效的 token 值                         │ │
│     │ - 如果 token 已经泄露，最安全的做法是让它失效                     │ │
│     │ - 合法用户应该能够快速撤销泄露的 token                             │ │
│     └─────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 6.3 快速查询表

| 问题 | 答案 |
|------|------|
| `/oauth/revoke` 是否需要 `auth()` 中间件？ | ❌ **不需要** |
| `/oauth/revoke` 的 token 如何传递？ | `body.token` 字段 |
| `/oauth/revoke` 是否验证请求者身份？ | ❌ **不验证**，只验证 token 本身 |
| `/oauth/revoke` 是否需要 CSRF 保护？ | ❌ **不需要** (`transport = "body"`) |
| `/oauth/authorize` 是否需要 CSRF 保护？ | ✅ **需要** (`transport = "cookie"`) |
| `/api/documents.list` 是否需要 CSRF？ | ❌ **不需要** (只读操作) |
| `/api/documents.create` 是否需要 CSRF？ | ✅ **需要** (写操作) |
| 什么是"写操作"？ | `canAccess(path, ["read"])` 返回 `false` |
| 哪些方法默认是"写操作"？ | `update`, `delete`, 及其他未在 `methodToScope` 中列出的方法 |
