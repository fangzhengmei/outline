# Outline 本地会话与 SSO 登录合并分析

## 文档说明

本文档严格区分：
- **✅ 事实**：代码库中可直接验证的实现逻辑
- **🤔 推断**：基于现有架构的合理推测，需进一步验证

---

## 1. 执行路径总览

### 1.1 完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              入口：用户点击 SSO 登录按钮                            │
│                         (用户可能已登录，也可能未登录)                              │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 1: 发起 OAuth 请求 - GET /auth/google (或其他 SSO 路由)                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ✅ 事实：路由注册 (server/routes/auth/index.ts:22-33)                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ router.use(                                                              │  │
│  │   "/",                                                                   │  │
│  │   authMiddleware({ optional: true }),  // 注意：optional=true！         │  │
│  │   resolvedRouter.routes()                                               │  │
│  │ );                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ✅ 事实：StateStore 保存当前会话 (server/utils/passport.ts:50-57)             │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ const accessToken = ctx.cookies.get("accessToken");                     │  │
│  │ const state = buildState({                                               │  │
│  │   host,                                                                   │  │
│  │   token,                                                                  │  │
│  │   client,                                                                 │  │
│  │   codeVerifier,                                                           │  │
│  │   accessToken,  // ← 关键：保存当前 accessToken 到 state cookie         │  │
│  │ });                                                                       │  │
│  │                                                                           │  │
│  │ ctx.cookies.set(this.key, state, {                                       │  │
│  │   expires: addMinutes(new Date(), 10),  // 10 分钟过期                 │  │
│  │   domain: getCookieDomain(ctx.hostname, env.isCloudHosted),             │  │
│  │ });                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  关键发现：                                                                      │
│  - authMiddleware 使用 optional=true，所以未登录用户也能访问 SSO 路由           │
│  - 如果用户已登录（有 accessToken cookie），accessToken 会被保存到 state        │
│  - state 被加密存储在 cookie 中，有效期 10 分钟                                 │
│                                                                                 │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼ (用户被重定向到 IdP 完成登录，然后回调)
                                │
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 2: SSO 回调处理 - GET/POST /auth/google.callback                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ✅ 事实：从 state 恢复用户 (各 SSO 插件，如 oidcRouter.ts:126-127)             │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ const user =                                                             │  │
│  │   context.state?.auth?.user ??              // 来自 authMiddleware      │  │
│  │   (await getUserFromOAuthState(context));  // 来自 state cookie         │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ✅ 事实：getUserFromOAuthState 实现 (server/utils/passport.ts:190-202)        │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ export async function getUserFromOAuthState(ctx: Context) {            │  │
│  │   const token = getAccessTokenFromOAuthState(ctx);                      │  │
│  │   if (!token) {                                                          │  │
│  │     return undefined;  // 未登录状态                                      │  │
│  │   }                                                                       │  │
│  │                                                                           │  │
│  │   try {                                                                   │  │
│  │     const { user } = await getUserForJWT(token);  // 验证 JWT          │  │
│  │     return user;                                                          │  │
│  │   } catch (_err) {                                                        │  │
│  │     return undefined;  // JWT 无效（过期、被篡改等）                      │  │
│  │   }                                                                       │  │
│  │ }                                                                         │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  关键发现：                                                                      │
│  - 恢复用户有两个来源：                                                          │
│    1. context.state.auth.user：来自 authMiddleware（但 optional 可能为空）     │
│    2. getUserFromOAuthState：从 state cookie 解析 accessToken                  │
│  - JWT 验证失败（过期、被篡改）会返回 undefined，视为未登录                     │
│                                                                                 │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 3: 调用 accountProvisioner 进行账号处理                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  代码位置：server/commands/accountProvisioner.ts:87-271                        │
│                                                                                 │
│  ✅ 事实：已登录用户的特殊处理分支 (accountProvisioner.ts:99-125)               │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ const actor = ctx.state.auth?.user;  // 从 state 恢复的"原用户"         │  │
│  │                                                                           │  │
│  │ // 条件判断：必须同时满足 3 个条件                                        │  │
│  │ if (actor &&                              // 1. 用户已登录               │  │
│  │     actor.teamId === teamParams.teamId && // 2. 同一团队               │  │
│  │     actor.isAdmin) {                      // 3. 是管理员               │  │
│  │                                                                           │  │
│  │   const team = actor.team;                                                │  │
│  │                                                                           │  │
│  │   // 查找或创建【团队级别】的认证提供者配置                                 │  │
│  │   const authenticationProvider = await AuthenticationProvider.findOne({  │  │
│  │     where: {                                                              │  │
│  │       ...authenticationProviderParams,  // { name, providerId }         │  │
│  │       teamId: team.id,                                                    │  │
│  │     },                                                                    │  │
│  │   });                                                                     │  │
│  │                                                                           │  │
│  │   if (!authenticationProvider) {                                          │  │
│  │     // 创建【团队级别】的认证提供者                                        │  │
│  │     await team.$create<AuthenticationProvider>(                          │  │
│  │       "authenticationProvider",                                           │  │
│  │       authenticationProviderParams                                        │  │
│  │     );                                                                    │  │
│  │   }                                                                       │  │
│  │                                                                           │  │
│  │   // 直接返回【原用户】，不进行任何账号合并！                               │  │
│  │   return {                                                                │  │
│  │     user: actor,         // ← 返回的是原登录用户                         │  │
│  │     team,                                                                 │  │
│  │     isNewUser: false,                                                     │  │
│  │     isNewTeam: false,                                                     │  │
│  │   };                                                                      │  │
│  │ }                                                                         │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ⚠️  极其重要的发现：                                                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │  1. 此分支【仅管理员】触发：条件 actor.isAdmin 必须为 true               │  │
│  │  2. 此分支的行为是【团队级别配置】：创建 AuthenticationProvider 记录      │  │
│  │  3. 此分支【不创建】UserAuthentication 记录（用户级别绑定）              │  │
│  │  4. 此分支【直接返回】原用户 actor，忽略 SSO 返回的用户信息             │  │
│  │  5. 此分支【不走】userProvisioner 的邮箱匹配等合并逻辑                   │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ✅ 事实：普通用户（非管理员）的处理路径 (accountProvisioner.ts:127-270)        │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ // 普通用户：不满足 admin 条件，走正常流程                               │  │
│  │                                                                           │  │
│  │ // Step A: 团队解析                                                      │  │
│  │ try {                                                                     │  │
│  │   result = await teamProvisioner(ctx, { ... });                         │  │
│  │ } catch (err) {                                                          │  │
│  │   // emailMatchOnly 回退模式（详见之前的报告）                           │  │
│  │ }                                                                         │  │
│  │                                                                           │  │
│  │ // Step B: 用户解析与合并                                                │  │
│  │ result = await userProvisioner(ctx, {                                    │  │
│  │   name: userParams.name,          // SSO 返回的用户名                    │  │
│  │   email: userParams.email,        // SSO 返回的邮箱                      │  │
│  │   // ...                                                                  │  │
│  │   authentication: emailMatchOnly                                          │  │
│  │     ? undefined                                                            │  │
│  │     : { authenticationProviderId, ...authenticationParams },            │  │
│  │ });                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  关键发现：                                                                      │
│  - 普通用户不走 admin 特殊分支                                                  │
│  - userProvisioner 【不使用】ctx.state.auth.user (actor)                      │
│  - userProvisioner 【仅使用】SSO 返回的 email 和 providerId 进行匹配         │
│                                                                                 │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 4: signIn - 会话切换与返回结果                                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ✅ 事实：signIn 实现 (server/utils/authentication.ts:31-178)                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ export async function signIn(                                            │  │
│  │   ctx: Context | APIContext,                                             │  │
│  │   service: string,                                                        │  │
│  │   { user, team, client, isNewTeam }: AuthenticationResult               │  │
│  │ ) {                                                                       │  │
│  │   // ... 检查挂起状态 ...                                                 │  │
│  │                                                                           │  │
│  │   // 更新用户登录时间                                                     │  │
│  │   await user.updateSignedIn(ctx);  // ← 注意：这里的 user 是           │  │
│  │                                        // accountProvisioner 返回的用户  │  │
│  │                                                                           │  │
│  │   // 记录登录事件                                                         │  │
│  │   await Event.createFromContext(ctx, {                                   │  │
│  │     name: "users.signin",                                                 │  │
│  │     userId: user.id,              // ← 返回用户的 ID                     │  │
│  │     authType: AuthenticationType.APP,                                     │  │
│  │     data: { name: user.name, service },                                   │  │
│  │   }, { actorId: user.id, teamId: team.id });                             │  │
│  │                                                                           │  │
│  │   // 设置新的 accessToken cookie                                          │  │
│  │   if (env.isCloudHosted && team.subdomain) {                             │  │
│  │     // ... 重定向到子域名，使用 transferToken                            │  │
│  │   } else {                                                                │  │
│  │     ctx.cookies.set(                                                      │  │
│  │       "accessToken",                                                      │  │
│  │       user.getJwtToken(expires, service),  // ← 新用户的 JWT           │  │
│  │       { sameSite: "lax", expires }                                        │  │
│  │     );                                                                    │  │
│  │   }                                                                       │  │
│  │                                                                           │  │
│  │   // 重定向                                                               │  │
│  │   ctx.redirect(...);                                                      │  │
│  │ }                                                                         │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  关键发现：                                                                      │
│  - signIn 完全不关心"原登录用户"是谁                                           │
│  - signIn 只使用 accountProvisioner 返回的 user                               │
│  - signIn 会设置【新的】accessToken cookie，覆盖旧的                           │
│  - 这意味着：会话会被切换到 accountProvisioner 返回的用户                      │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 分场景详细分析

### 场景 A：管理员已登录，发起 SSO 登录（同团队）

#### 前置条件
| 条件 | 值 |
|-----|---|
| 原登录用户 | User A (admin@company.com, isAdmin=true, teamId=team-123) |
| 团队 | Team 123 (已配置 Google 登录) |
| SSO 登录使用的账号 | User B (another-admin@company.com, 也是 Google 账号) |

#### 执行路径

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 1: 发起 SSO 请求                                                          │
│  - accessToken (User A 的) 被保存到 state cookie                              │
│  - 用户被重定向到 Google                                                         │
│  - 用户在 Google 选择 User B 的账号                                             │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 2: SSO 回调                                                               │
│  - 从 state cookie 恢复 User A (actor = User A)                                │
│  - 从 Google 回调获取 User B 的信息:                                            │
│    - email: another-admin@company.com                                           │
│    - providerId: google-user-id-b                                               │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 3: accountProvisioner 判断                                                │
│                                                                                 │
│  ✅ 事实：条件判断 (accountProvisioner.ts:103)                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ actor &&                   // true (User A 已恢复)                       │  │
│  │ actor.teamId === teamParams.teamId && // true (同一团队)                 │  │
│  │ actor.isAdmin              // true (User A 是管理员)                     │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  结果：【Admin 分支被触发】                                                     │
│                                                                                 │
│  ✅ 事实：Admin 分支行为 (accountProvisioner.ts:104-124)                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ 1. 查找或创建 AuthenticationProvider（团队级别配置）                     │  │
│  │    - 假设 Team 123 已有 Google 配置，跳过创建                           │  │
│  │                                                                           │  │
│  │ 2. 直接返回：                                                             │  │
│  │    {                                                                      │  │
│  │      user: actor,        // ← 返回 User A，不是 User B！                │  │
│  │      team: Team 123,                                                     │  │
│  │      isNewUser: false,                                                    │  │
│  │      isNewTeam: false,                                                    │  │
│  │    }                                                                      │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ⚠️  极其重要：                                                                 │
│  - SSO 返回的 User B 的信息被【完全忽略】                                      │
│  - 没有进行邮箱匹配（User A.email ≠ User B.email）                             │
│  - 没有创建 UserAuthentication 记录                                            │
│  - 返回的是【原登录用户 User A】                                               │
│                                                                                 │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 4: signIn                                                                 │
│  - 使用 User A 登录                                                             │
│  - 刷新 User A 的 accessToken                                                   │
│  - 记录 User A 的登录事件                                                        │
│                                                                                 │
│  最终结果：                                                                      │
│  - 用户仍以 User A 身份登录                                                     │
│  - User B 的 SSO 账号【没有被绑定】到任何用户                                  │
│  - 如果下次用 User B 以未登录状态登录，会创建新用户或进行邮箱匹配             │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 事实 vs 推断

| 项目 | 状态 | 说明 |
|-----|------|------|
| Admin 分支条件 | ✅ 事实 | 代码明确检查 `actor.isAdmin` |
| 返回原用户 | ✅ 事实 | `return { user: actor, ... }` |
| 忽略 SSO 用户信息 | ✅ 事实 | SSO 的 email 和 providerId 没有被使用 |
| 不创建 UserAuthentication | ✅ 事实 | Admin 分支中没有 `userProvisioner` 调用 |
| Admin 分支的设计意图 | 🤔 推断 | 可能是为了让管理员"连接"新的 SSO 提供者到团队，而不是绑定到个人账号 |

---

### 场景 B：普通用户已登录，发起 SSO 登录（同一用户）

#### 前置条件
| 条件 | 值 |
|-----|---|
| 原登录用户 | User A (user@company.com, isAdmin=false, teamId=team-123) |
| 团队 | Team 123 |
| SSO 登录使用的账号 | User A (user@company.com, Google 账号) |
| SSO 返回 | email: user@company.com, providerId: google-user-id-a |

#### 执行路径

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 1: 发起 SSO 请求                                                          │
│  - User A 的 accessToken 被保存到 state                                        │
│  - 用户在 Google 选择同一账号                                                   │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 2: SSO 回调                                                               │
│  - 从 state 恢复 User A (actor = User A)                                       │
│  - SSO 返回 email=user@company.com, providerId=google-user-id-a               │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 3: accountProvisioner 判断                                                │
│                                                                                 │
│  ✅ 事实：条件判断 (accountProvisioner.ts:103)                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ actor &&                   // true (User A 已恢复)                       │  │
│  │ actor.teamId === teamParams.teamId && // true (同一团队)                 │  │
│  │ actor.isAdmin              // FALSE (User A 是普通用户)                  │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  结果：【Admin 分支不触发】，走正常流程                                        │
│                                                                                 │
│  ✅ 事实：正常流程调用                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ result = await teamProvisioner(ctx, { ... });  // 团队解析             │  │
│  │ result = await userProvisioner(ctx, {      // 用户解析与合并            │  │
│  │   name: ...,                        // SSO 返回的名称                     │  │
│  │   email: "user@company.com",        // SSO 返回的邮箱                    │  │
│  │   authentication: {                                                        │  │
│  │     providerId: "google-user-id-a",  // SSO 返回的用户 ID                │  │
│  │     ...                                                                    │  │
│  │   },                                                                       │  │
│  │ });                                                                        │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ✅ 事实：userProvisioner 行为 (userProvisioner.ts:59-259)                    │
│                                                                                 │
│  【Step 3a: providerId 精确匹配】                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ SELECT * FROM user_authentications                                       │  │
│  │ INNER JOIN users ON users.id = user_authentications.userId              │  │
│  │ WHERE user_authentications.providerId = 'google-user-id-a'             │  │
│  │   AND users.teamId = 'team-123'                                          │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  假设：User A 之前没有通过 Google 登录过，所以无匹配                           │
│                                                                                 │
│  【Step 3b: email 模糊匹配】                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ SELECT * FROM users                                                       │  │
│  │ WHERE email ILIKE 'user@company.com'  -- 不区分大小写                   │  │
│  │   AND teamId = 'team-123'                                                │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  结果：找到 User A！                                                            │
│                                                                                 │
│  【Step 3c: 合并逻辑】                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ // 在事务中执行                                                           │  │
│  │ await existingUser.update({               // 更新用户信息                 │  │
│  │   name,                                     // SSO 返回的名称            │  │
│  │   avatarUrl,                                // SSO 返回的头像            │  │
│  │   lastActiveAt: new Date(),                 // 标记为活跃               │  │
│  │   lastActiveIp: ctx.ip,                                                    │  │
│  │ }, { transaction });                                                        │  │
│  │                                                                           │  │
│  │ // 为现有用户创建【新的】UserAuthentication 记录                          │  │
│  │ await existingUser.$create<UserAuthentication>(                           │  │
│  │   "authentication",                                                         │  │
│  │   {                                                                         │  │
│  │     authenticationProviderId: ...,         // Google 提供者 ID          │  │
│  │     providerId: "google-user-id-a",        // Google 用户 ID            │  │
│  │     accessToken: ...,                      // 加密存储                   │  │
│  │     refreshToken: ...,                                                     │  │
│  │     expiresAt: ...,                                                         │  │
│  │   },                                                                        │  │
│  │   { transaction }                                                           │  │
│  │ );                                                                          │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  返回结果：                                                                      │
│  {                                                                             │
│    user: User A,           // 同一用户                                        │
│    authentication: 新的 UserAuthentication 记录,  // 新增绑定                │
│    isNewUser: false,        // 不是新用户                                     │
│  }                                                                             │
│                                                                                 │
│  关键发现：                                                                      │
│  - userProvisioner 【不关心】ctx.state.auth.user (actor = User A)           │
│  - 它只根据 SSO 返回的 email 和 providerId 进行匹配                            │
│  - 碰巧这里 email 匹配到了 User A                                              │
│  - 为 User A 创建了新的 Google 认证记录                                        │
│                                                                                 │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 4: signIn                                                                 │
│  - 使用 User A 登录（与原登录用户相同）                                         │
│  - 刷新 accessToken                                                             │
│  - 记录登录事件                                                                  │
│                                                                                 │
│  最终结果：                                                                      │
│  - 用户仍以 User A 身份登录                                                     │
│  - User A 现在【多了一个】Google 认证方式                                      │
│  - 下次可以用密码或 Google 登录                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 事实 vs 推断

| 项目 | 状态 | 说明 |
|-----|------|------|
| 普通用户不走 Admin 分支 | ✅ 事实 | `actor.isAdmin` 为 false |
| userProvisioner 不使用 actor | ✅ 事实 | 函数签名中没有使用 `ctx.state.auth.user` |
| 邮箱匹配到同一用户 | ✅ 事实 | `Op.iLike` 不区分大小写匹配 |
| 创建新的 UserAuthentication | ✅ 事实 | `existingUser.$create("authentication", ...)` |
| 原会话被切换到"新"用户 | ✅ 事实 | signIn 设置新的 accessToken（但这里是同一用户） |

---

### 场景 C：普通用户已登录，发起 SSO 登录（不同用户）

#### 前置条件
| 条件 | 值 |
|-----|---|
| 原登录用户 | User A (user-a@company.com, isAdmin=false, teamId=team-123) |
| 团队 | Team 123 |
| SSO 登录使用的账号 | User B (user-b@company.com, Google 账号) |
| SSO 返回 | email: user-b@company.com, providerId: google-user-id-b |
| 假设 | User B 之前已存在于 Team 123（通过邀请或其他方式） |

#### 执行路径

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 1: 发起 SSO 请求                                                          │
│  - User A 的 accessToken 被保存到 state                                        │
│  - 用户在 Google 选择 User B 的账号                                            │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 2: SSO 回调                                                               │
│  - 从 state 恢复 User A (actor = User A)                                       │
│  - SSO 返回 email=user-b@company.com, providerId=google-user-id-b            │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 3: accountProvisioner + userProvisioner                                  │
│                                                                                 │
│  Admin 分支不触发（User A 不是管理员）                                          │
│                                                                                 │
│  userProvisioner 匹配逻辑：                                                    │
│  1. providerId 匹配：查找 google-user-id-b                                    │
│     - 假设 User B 之前没有通过 Google 登录，无匹配                            │
│                                                                                 │
│  2. email 匹配：查找 user-b@company.com                                       │
│     - 找到 User B！（之前通过邀请创建）                                        │
│                                                                                 │
│  3. 合并逻辑：                                                                  │
│     - 更新 User B 的信息（name, avatarUrl）                                   │
│     - 为 User B 创建新的 UserAuthentication 记录                              │
│                                                                                 │
│  返回结果：                                                                      │
│  {                                                                             │
│    user: User B,           // ← 注意：返回的是 User B，不是 User A！        │
│    authentication: 新的 UserAuthentication 记录,                               │
│    isNewUser: false,                                                            │
│  }                                                                             │
│                                                                                 │
│  ⚠️  极其重要：                                                                 │
│  - actor = User A（原登录用户）                                                │
│  - 但返回的 user = User B（SSO 登录的用户）                                    │
│  - 系统【没有检查】这两个用户是否相同                                          │
│  - 系统【没有阻止】这种"切换"行为                                              │
│                                                                                 │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 4: signIn                                                                 │
│  - signIn 接收的 user = User B                                                  │
│  - 它【不关心】原登录用户是谁                                                   │
│  - 它会：                                                                       │
│    1. 更新 User B 的 lastSignedInAt                                            │
│    2. 记录 User B 的登录事件                                                   │
│    3. 设置【User B 的】accessToken cookie（覆盖 User A 的）                    │
│    4. 重定向到首页                                                              │
│                                                                                 │
│  最终结果：                                                                      │
│  - 会话从 User A【切换】到 User B                                              │
│  - User B 现在绑定了 Google 认证                                               │
│  - User A 被"踢下线"（accessToken 被覆盖）                                     │
│  - 整个过程【没有任何提示或确认】                                              │
│                                                                                 │
│  🤔  这是预期行为吗？                                                           │
│  - 从代码看：是的，这就是实现方式                                               │
│  - 从产品角度：可能是预期的（用户明确选择了不同的账号）                         │
│  - 从安全角度：需要确认这是否符合安全要求                                      │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 事实 vs 推断

| 项目 | 状态 | 说明 |
|-----|------|------|
| userProvisioner 忽略 actor | ✅ 事实 | 代码中没有使用 `ctx.state.auth.user` 进行比较或限制 |
| 返回不同用户 | ✅ 事实 | 返回的是邮箱匹配的用户，不是原登录用户 |
| signIn 切换会话 | ✅ 事实 | 总是设置返回用户的 accessToken，覆盖旧的 |
| 无用户切换确认 | ✅ 事实 | 代码中没有任何比较或确认逻辑 |
| 产品意图 | 🤔 推断 | 可能是"用户明确选择了不同账号，应该切换过去" |
| 安全风险 | 🤔 推断 | 如果用户在公共电脑上忘记登出，他人可能用其他 SSO 账号切换 |

---

### 场景 D：普通用户已登录，发起 SSO 登录（完全新用户）

#### 前置条件
| 条件 | 值 |
|-----|---|
| 原登录用户 | User A (user-a@company.com, isAdmin=false, teamId=team-123) |
| 团队 | Team 123 |
| SSO 登录使用的账号 | User C (user-c@external.com, Google 账号) |
| SSO 返回 | email: user-c@external.com, providerId: google-user-id-c |
| 团队设置 | inviteRequired=false, 允许 user-c@external.com 域名 |

#### 执行路径

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 1-2: 同上，恢复 User A，获取 User C 的 SSO 信息                          │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 3: userProvisioner                                                        │
│                                                                                 │
│  1. providerId 匹配：google-user-id-b → 无匹配                                │
│  2. email 匹配：user-c@external.com → 无匹配                                   │
│                                                                                 │
│  3. 检查团队设置：                                                              │
│     - inviteRequired = false ✓                                                 │
│     - 域名白名单：user-c@external.com 被允许 ✓                                 │
│                                                                                 │
│  4. 创建【新用户】：                                                            │
│     - 创建 User C 记录                                                          │
│     - 创建 UserAuthentication 记录                                             │
│                                                                                 │
│  返回结果：                                                                      │
│  {                                                                             │
│    user: User C,           // ← 全新创建的用户                                 │
│    authentication: 新的 UserAuthentication 记录,                               │
│    isNewUser: true,         // 标记为新用户                                    │
│  }                                                                             │
│                                                                                 │
└───────────────────────────────┬─────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Step 4: signIn                                                                 │
│  - 会话从 User A 切换到【全新的 User C】                                       │
│  - User A 被踢下线                                                              │
│  - Team 123 多了一个新成员 User C                                              │
│                                                                                 │
│  ⚠️  安全考虑：                                                                 │
│  - 如果 Team 123 是敏感团队，这可能导致未授权访问                              │
│  - 但根据团队设置（inviteRequired=false，域名允许），这是预期行为             │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 关键决策点汇总

### 3.1 Admin 分支 vs 普通用户分支

| 决策点 | Admin 分支 (actor.isAdmin=true) | 普通用户分支 |
|-------|---------------------------------|-------------|
| 触发条件 | `actor && actor.teamId === teamParams.teamId && actor.isAdmin` | 不满足 Admin 条件 |
| 使用 actor | ✅ 使用：直接返回 `actor` | ❌ 忽略：不使用 `ctx.state.auth.user` |
| 使用 SSO 用户信息 | ❌ 忽略 | ✅ 使用：email 和 providerId 用于匹配 |
| 创建 AuthenticationProvider | ✅ 团队级别配置 | 由 teamProvisioner 处理 |
| 创建 UserAuthentication | ❌ 不创建 | ✅ 由 userProvisioner 创建 |
| 账号合并逻辑 | ❌ 跳过 | ✅ 执行：providerId → email → 创建新用户 |
| 返回用户 | `actor`（原登录用户） | SSO 信息匹配到的用户 |

### 3.2 userProvisioner 合并优先级

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        userProvisioner 决策流程                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  输入:                                                                          │
│  - email: "user@company.com"          (来自 SSO)                               │
│  - providerId: "google-user-123"     (来自 SSO)                               │
│  - authentication: { accessToken, ... } (来自 SSO)                            │
│  - teamId: "team-123"                                                          │
│                                                                                 │
│  注意：【没有】actor 或原登录用户作为输入！                                      │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  优先级 1: providerId 精确匹配 (userProvisioner.ts:59-119)                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ 查找条件:                                                                 │  │
│  │ UserAuthentication.providerId = "google-user-123"                       │  │
│  │ AND User.teamId = "team-123"                                             │  │
│  │                                                                           │  │
│  │ 找到?                                                                    │  │
│  │ ├── 是:                                                                  │  │
│  │ │   - 更新 UserAuthentication (令牌、过期时间等)                        │  │
│  │ │   - 检查 authenticationProviderId 是否变更，自动迁移                   │  │
│  │ │   - 更新 User.email (以 SSO 返回的为准)                               │  │
│  │ │   - 返回: { user, authentication, isNewUser: false }                  │  │
│  │ │                                                                         │  │
│  │ └── 否:                                                                  │  │
│  │     - 继续优先级 2                                                        │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  优先级 2: email 模糊匹配 (userProvisioner.ts:122-200)                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ 查找条件:                                                                 │  │
│  │ User.email ILIKE "user@company.com"  (不区分大小写)                     │  │
│  │ AND User.teamId = "team-123"                                             │  │
│  │                                                                           │  │
│  │ 找到?                                                                    │  │
│  │ ├── 是:                                                                  │  │
│  │ │   - 更新 User (name, avatarUrl, lastActiveAt, lastActiveIp)          │  │
│  │ │   - 【创建新的】UserAuthentication 记录                               │  │
│  │ │   - 判断 isInvite: lastActiveAt === null                             │  │
│  │ │   - 返回: { user, authentication, isNewUser: isInvite }              │  │
│  │ │                                                                         │  │
│  │ │   关键点:                                                              │  │
│  │ │   - 一个用户可以有【多个】UserAuthentication 记录                     │  │
│  │ │   - 这就是"多 SSO 账号绑定同一用户"的实现方式                         │  │
│  │ │                                                                         │  │
│  │ └── 否:                                                                  │  │
│  │     - 继续优先级 3                                                        │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  优先级 3: 创建新用户或错误 (userProvisioner.ts:201-259)                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ 检查条件:                                                                 │  │
│  │                                                                           │  │
│  │ 1. emailMatchOnly 且无域名白名单?                                       │  │
│  │    IF !authentication AND !team.allowedDomains.length                   │  │
│  │    → 抛出 InvalidAuthenticationError                                    │  │
│  │                                                                           │  │
│  │ 2. 团队要求邀请?                                                         │  │
│  │    IF team.inviteRequired                                                │  │
│  │    → 抛出 InviteRequiredError                                           │  │
│  │                                                                           │  │
│  │ 3. 域名在白名单?                                                         │  │
│  │    IF team AND !team.isDomainAllowed(email)                             │  │
│  │    → 抛出 DomainNotAllowedError                                         │  │
│  │                                                                           │  │
│  │ 通过所有检查:                                                             │  │
│  │ - 创建新 User 记录                                                        │  │
│  │ - 创建新 UserAuthentication 记录（如有 authentication 参数）            │  │
│  │ - 返回: { user, authentication, isNewUser: true }                       │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 signIn 会话切换逻辑

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           signIn 函数行为                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  输入:                                                                          │
│  - user: 来自 accountProvisioner 的返回值                                      │
│  - team: 来自 accountProvisioner 的返回值                                      │
│  - service: SSO 提供者名称 ("google", "oidc" 等)                             │
│                                                                                 │
│  关键发现:                                                                      │
│  - signIn【不接受】actor 或原登录用户作为参数                                  │
│  - signIn【不知道】用户是否从另一个账号切换过来                                │
│  - signIn 只是简单地"让传入的 user 登录"                                       │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  执行步骤:                                                                      │
│                                                                                 │
│  1. 检查挂起状态                                                                │
│     - team.isSuspended → 重定向到 notice=team-suspended                      │
│     - user.isSuspended → 重定向到 notice=user-suspended                      │
│                                                                                 │
│  2. 更新登录时间                                                                │
│     - user.updateSignedIn(ctx)                                                 │
│     - 设置 lastSignedInAt, lastSignedInIp                                     │
│                                                                                 │
│  3. 记录登录事件                                                                │
│     - Event.createFromContext(ctx, { name: "users.signin", userId: user.id })│
│     - actorId = user.id (当前用户，不是原用户)                                │
│                                                                                 │
│  4. 设置新会话                                                                  │
│     ┌─────────────────────────────────────────────────────────────────────┐  │
│     │ 云托管 + 有子域名:                                                    │  │
│     │ - 创建 transferToken (有效期 1 分钟)                                 │  │
│     │ - 重定向到 ${team.url}/auth/redirect?token=${transferToken}        │  │
│     │ - 在目标子域名设置 accessToken                                       │  │
│     │                                                                       │  │
│     │ 自托管或无域名:                                                       │  │
│     │ - 直接设置 accessToken cookie                                        │  │
│     │ ctx.cookies.set(                                                      │  │
│     │   "accessToken",                                                      │  │
│     │   user.getJwtToken(expires, service),  // 新用户的 JWT              │  │
│     │   { sameSite: "lax", expires }                                        │  │
│     │ );                                                                    │  │
│     │                                                                       │  │
│     │ 关键点:                                                               │  │
│     │ - 新的 accessToken【覆盖】旧的                                        │  │
│     │ - 没有"合并"会话的概念，只有"切换"                                    │  │
│     └─────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  5. 重定向                                                                      │
│     - 有 defaultCollectionId → 该集合                                         │
│     - 有已读文档 → /home                                                        │
│     - 无已读文档 → 第一个集合的 /recent                                        │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 事实 vs 推断 对照表

### 4.1 已验证事实（代码可直接验证）

| 事实 | 代码位置 | 验证方式 |
|-----|---------|---------|
| SSO 路由使用 `authMiddleware({ optional: true })` | `server/routes/auth/index.ts:28` | 阅读代码 |
| StateStore 保存当前 accessToken 到 state cookie | `server/utils/passport.ts:50-57` | 阅读代码 |
| state cookie 有效期 10 分钟 | `server/utils/passport.ts:59-62` | 阅读代码 |
| getUserFromOAuthState 从 state 恢复用户 | `server/utils/passport.ts:190-202` | 阅读代码 |
| Admin 分支条件：`actor && actor.teamId === teamId && actor.isAdmin` | `server/commands/accountProvisioner.ts:103` | 阅读代码 |
| Admin 分支直接返回 actor，不使用 SSO 用户信息 | `server/commands/accountProvisioner.ts:119-124` | 阅读代码 |
| Admin 分支不创建 UserAuthentication | `server/commands/accountProvisioner.ts:103-125` | 阅读代码 |
| userProvisioner 函数签名不包含 actor | `server/commands/userProvisioner.ts:55-58` | 阅读代码 |
| userProvisioner 内部不使用 `ctx.state.auth.user` | `server/commands/userProvisioner.ts:59-259` | 阅读代码 |
| userProvisioner 支持多认证记录（一个用户多个 SSO） | `server/commands/userProvisioner.ts:167-173` | 阅读代码 |
| signIn 总是设置返回用户的 accessToken，覆盖旧的 | `server/utils/authentication.ts:137-140` | 阅读代码 |
| 邮箱登录时，如果用户有 SSO 绑定，重定向到 SSO | `plugins/email/server/auth/email.ts:64-73` | 阅读代码 |

### 4.2 架构推断（基于代码逻辑的合理推测）

| 推断 | 依据 | 置信度 |
|-----|------|-------|
| Admin 分支的设计意图是"团队级别配置 SSO"，而非"用户绑定 SSO" | Admin 分支创建 AuthenticationProvider（团队级别），不创建 UserAuthentication（用户级别） | 高 |
| 普通用户已登录时用不同 SSO 账号登录，会话会切换 | userProvisioner 忽略 actor，signIn 覆盖 accessToken | 高 |
| 会话切换没有任何确认提示 | 代码中没有比较 actor 和返回 user 的逻辑 | 高 |
| 用户可以通过 SSO 登录到任意邮箱匹配的账号 | email 匹配优先级 2，不需要原用户密码或确认 | 中 |
| Admin 分支中，SSO 用户信息被完全忽略 | Admin 分支直接返回 actor，SSO 的 email 和 providerId 没有被使用 | 高 |
| 管理员用其他 SSO 账号登录不会绑定到自己账号 | Admin 分支不创建 UserAuthentication | 高 |
| 管理员用其他 SSO 账号登录也不会创建新用户 | Admin 分支直接返回，不走 userProvisioner | 高 |

### 4.3 需要进一步验证的假设

| 假设 | 验证方法 | 风险 |
|-----|---------|------|
| 用户 A 已登录，用用户 B 的 SSO 账号登录，会切换到 B | 集成测试 | 需要确认是否为预期行为 |
| Admin 分支的"连接新认证提供者"只影响团队配置，不影响用户 | 集成测试 | 需要确认产品设计意图 |
| emailMatchOnly 模式下，普通用户也会走该分支 | 查看 teamProvisioner 何时抛出 invalid_authentication | 需要确认触发条件 |

---

## 5. 边界场景汇总

### 5.1 场景矩阵

| # | 原登录用户 | SSO 登录用户 | 用户类型 | 团队匹配 | 预期行为（基于代码） |
|---|-----------|-------------|---------|---------|---------------------|
| 1 | Admin A | Admin A (同邮箱) | Admin | 同一团队 | Admin 分支，返回 A，不创建绑定 |
| 2 | Admin A | User B (不同邮箱) | Admin | 同一团队 | Admin 分支，返回 A，忽略 B，不创建绑定 |
| 3 | User A (普通) | User A (同邮箱) | 普通 | 同一团队 | 邮箱匹配，为 A 创建新认证记录，会话保持 A |
| 4 | User A (普通) | User B (不同邮箱，B 已存在) | 普通 | 同一团队 | 邮箱匹配到 B，会话切换到 B，为 B 创建新认证 |
| 5 | User A (普通) | User C (全新用户) | 普通 | 同一团队 | 创建新用户 C，会话切换到 C，A 被踢下线 |
| 6 | Admin A (Team 1) | Admin A (同邮箱) | Admin | 不同团队 | Admin 分支条件不满足，走正常流程 |
| 7 | 未登录 | User A | 任意 | 任意 | 正常登录流程 |

### 5.2 关键场景详细说明

#### 场景 2：管理员用其他账号 SSO 登录

**代码行为**：
```typescript
// accountProvisioner.ts:103-125
if (actor && actor.teamId === teamParams.teamId && actor.isAdmin) {
  // 查找或创建团队级别的 AuthenticationProvider
  const authenticationProvider = await AuthenticationProvider.findOne({
    where: {
      ...authenticationProviderParams,  // SSO 返回的提供者信息
      teamId: team.id,
    },
  });

  if (!authenticationProvider) {
    await team.$create<AuthenticationProvider>(
      "authenticationProvider",
      authenticationProviderParams
    );
  }

  // 直接返回原用户，SSO 用户信息被忽略！
  return {
    user: actor,         // ← 原登录用户
    team,
    isNewUser: false,
    isNewTeam: false,
  };
}
```

**结果**：
- 团队可能多了一个 AuthenticationProvider 配置
- 但 SSO 账号【没有绑定】到任何用户
- 管理员仍以原账号登录

**潜在问题**：
- 管理员可能误以为"我用 B 账号登录了，应该绑定到一起"
- 但实际上什么绑定都没发生
- B 账号下次登录时，如果是未登录状态，会走正常流程（可能创建新用户或匹配其他账号）

#### 场景 4：普通用户切换到其他账号

**代码行为**：
- userProvisioner 完全忽略原登录用户
- 只根据 SSO 返回的 email 匹配
- 匹配到 B，返回 B
- signIn 设置 B 的 accessToken，覆盖 A 的

**结果**：
- 会话从 A 切换到 B
- A 被踢下线
- B 多了一个 SSO 认证方式

**安全考虑**：
- 这是"会话劫持"吗？
- 从代码看：不是，因为用户明确在 IdP 选择了 B 账号
- 从安全角度：需要确认这是否符合预期
- 建议：考虑添加确认提示"您即将切换到另一个账号，是否继续？"

---

## 6. 代码索引

### 6.1 关键文件

| 文件路径 | 职责 | 关键函数/逻辑 |
|---------|------|--------------|
| `server/routes/auth/index.ts` | 认证路由注册 | `authMiddleware({ optional: true })` |
| `server/utils/passport.ts` | OAuth 状态管理 | `StateStore.store()`, `getUserFromOAuthState()` |
| `server/commands/accountProvisioner.ts` | 账号供应主入口 | Admin 分支逻辑 (99-125 行) |
| `server/commands/userProvisioner.ts` | 用户合并逻辑 | 三级匹配优先级 |
| `server/utils/authentication.ts` | 登录会话管理 | `signIn()` 函数 |
| `plugins/email/server/auth/email.ts` | 邮箱登录 | SSO 用户重定向逻辑 (64-73 行) |

### 6.2 关键测试用例

| 测试用例 | 文件位置 | 覆盖场景 |
|---------|---------|---------|
| `should allow connecting a new authentication provider while logged in` | `accountProvisioner.test.ts:413-448` | 管理员已登录时连接新认证提供者 |

---

## 7. 建议与注意事项

### 7.1 产品设计建议

1. **Admin 分支行为澄清**
   - 当前行为容易让管理员困惑
   - 建议在 UI 上明确区分：
     - "为团队启用 Google 登录"（团队级别配置）
     - "将我的账号绑定到 Google"（用户级别绑定）

2. **账号切换确认**
   - 普通用户用不同账号 SSO 登录时，建议添加确认页面
   - 例如："您已登录为 User A，即将切换到 User B，是否继续？"

3. **Admin 分支的用户绑定**
   - 当前 Admin 分支不创建 UserAuthentication
   - 如果管理员想要绑定自己的账号，需要先登出再登录
   - 这可能是预期行为，但需要明确文档说明

### 7.2 安全建议

1. **会话切换审计**
   - 当前 `users.signin` 事件只记录登录用户
   - 建议添加字段记录"从哪个账号切换而来"（如果有）

2. **敏感操作重认证**
   - 对于敏感操作（如修改团队设置、删除文档）
   - 建议在会话切换后要求重新认证

3. **State Cookie 安全性**
   - state cookie 包含 accessToken，有效期 10 分钟
   - 确保使用 `secure` 和 `httpOnly` 标志（需要确认）

### 7.3 运维建议

1. **日志监控**
   - 监控 `users.signin` 事件
   - 关注短时间内同一 IP 登录不同账号的情况

2. **用户教育**
   - 告知用户："在 SSO 登录页面选择不同账号会切换当前会话"
   - 告知管理员："Admin 分支只配置团队 SSO，不绑定个人账号"

---

*报告生成时间: 2026-05-05*
*基于代码版本: 当前工作区*
*事实/推断区分: 已严格标注*
