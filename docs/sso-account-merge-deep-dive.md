# Outline SSO 账号合并深度分析：决策分支与边界场景

## 1. 认证系统架构总览

### 1.1 认证提供者类型

Outline 支持多种认证方式，按优先级排序（`AuthenticationHelper.ts:47-49`）：

| 优先级 | 认证方式 | ID | 类型 | 描述 |
|-------|---------|-----|------|------|
| 高 (负优先级) | SSO 协议 | google, azure, oidc, gitlab, github 等 | OAuth/OIDC | 外部身份提供商 |
| 低 (正优先级) | 本地认证 | email, passkeys | 无外部依赖 | 邮箱魔法链接 / WebAuthn |

### 1.2 核心数据模型关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Team (团队)                                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  AuthenticationProvider (认证提供者配置)                          │    │
│  │  - name: "google" | "azure" | "oidc" | "email" 等              │    │
│  │  - providerId: 外部标识（域名/租户ID）                            │    │
│  │  - enabled: 是否启用                                              │    │
│  │  - settings: 额外配置（如组同步）                                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  User (用户)                                                      │    │
│  │  - email: 邮箱（合并的关键标识）                                  │    │
│  │  - teamId: 所属团队                                              │    │
│  │  - lastActiveAt: 最后活动时间（null 表示邀请用户）              │    │
│  │  - invitedById: 邀请人ID（如有）                                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  UserAuthentication (用户认证记录)                               │    │
│  │  - userId: 关联用户                                              │    │
│  │  - authenticationProviderId: 关联认证提供者                      │    │
│  │  - providerId: 外部用户ID（sub/oid 等）                         │    │
│  │  - accessToken/refreshToken: 加密存储的令牌                     │    │
│  │  - 唯一约束: (userId, authenticationProviderId)                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. SAML 登录链路分析（基于现有架构推断）

### 2.1 SAML 现状

根据代码库分析：
- **社区版**：无 SAML 实现（仅翻译文件中有 `saml` 字符串）
- **企业版**：可能包含 SAML 支持（翻译文件位于 `plugins/enterprise/`）

### 2.2 基于现有 OAuth/OIDC 架构的 SAML 推断实现

如果要实现 SAML，其登录链路应与现有 OIDC 插件保持一致：

```
┌─────────────┐      ┌─────────────────┐      ┌─────────────────────┐
│   用户浏览器  │ ───► │  Outline 服务   │ ───► │  SAML Identity      │
│             │      │                 │      │  Provider (IdP)     │
└─────────────┘      └─────────────────┘      └─────────────────────┘
       ▲                      │                          │
       │                      ▼                          │
       │              ┌─────────────┐                   │
       │              │  passport-  │                   │
       │              │  saml 策略   │                   │
       │              └─────────────┘                   │
       │                      │                          │
       │                      ▼                          │
       │              ┌─────────────────┐               │
       │              │ accountProvisioner│ ◄───────────┘
       │              │  (账号合并逻辑)  │
       │              └─────────────────┘
       │                      │
       │                      ▼
       │              ┌─────────────────┐
       └──────────────│    signIn()     │
                      │  (设置会话Cookie)│
                      └─────────────────┘
```

### 2.3 SAML 插件预期结构（参照 OIDC）

```
plugins/saml/
├── plugin.json           # 插件元数据: { id: "saml", name: "SAML", priority: 10 }
├── server/
│   ├── index.ts          # 插件入口，注册 AuthProvider hook
│   ├── env.ts            # 环境变量配置
│   │                     #   - SAML_ENTRYPOINT: IdP 登录 URL
│   │                     #   - SAML_ISSUER: SP Entity ID
│   │                     #   - SAML_CALLBACK_URL: ACS URL
│   │                     #   - SAML_CERT: IdP 签名证书
│   │                     #   - SAML_PRIVATE_KEY: SP 私钥（可选）
│   └── auth/
│       ├── samlRouter.ts # 路由定义: /auth/saml, /auth/saml.callback
│       └── SAMLStrategy.ts # passport-saml 策略封装
```

### 2.4 SAML 响应处理预期逻辑

```typescript
// 参照 oidcRouter.ts 的结构，SAML 回调应类似：

// 1. 解析 SAML Response 中的断言
const profile = samlStrategy.validatePostResponse(req);

// 2. 提取关键字段（与 OIDC 对应）
const email = profile.email ?? profile.nameID;  // SAML 的 NameID
const name = profile.displayName ?? profile.cn;
const profileId = profile.nameID;                // 外部用户标识
const providerId = profile.issuer;               // IdP Entity ID

// 3. 调用账号供应器（与 OIDC 完全相同的合并逻辑）
const result = await accountProvisioner(ctx, {
  team: { teamId, name, domain, subdomain },
  user: { name, email, avatarUrl },
  authenticationProvider: {
    name: "saml",
    providerId: providerId,  // IdP Entity ID
  },
  authentication: {
    providerId: profileId,   // NameID / 用户唯一标识
    // SAML 通常没有 access_token，但可存储会话信息
  },
});
```

### 2.5 SAML 与 OIDC 字段映射

| 概念 | OIDC 字段 | SAML 断言属性 | 用途 |
|-----|----------|--------------|------|
| 用户邮箱 | `email` | `email`, `mail`, `emailAddress` | 账号合并关键标识 |
| 用户唯一ID | `sub` | `nameID` (Subject) | providerId 精确匹配 |
| 用户名 | `name`, `preferred_username` | `displayName`, `cn`, `givenName` + `sn` | 用户显示名 |
| IdP 标识 | `iss` (from id_token) | `Issuer` | authenticationProvider.providerId |
| 头像 | `picture` | (通常无，需额外配置) | 用户头像 |

---

## 3. 本地账号与 SSO 绑定规则

### 3.1 本地账号的定义

**本地账号** = 用户记录存在，但 `UserAuthentication` 记录不包含 SSO 类型的认证提供者。

典型场景：
1. **邀请用户**：`lastActiveAt IS NULL`，通过 `userInviter.ts` 创建
2. **邮箱注册用户**：通过 email 魔法链接登录，无关联 SSO 认证记录
3. **Passkey 用户**：通过 WebAuthn 登录，认证记录在 `UserPasskey` 模型

### 3.2 绑定决策流程

```
                    ┌─────────────────────────────┐
                    │  用户通过 SSO 发起登录请求   │
                    │  (Google/Azure/OIDC/SAML)  │
                    └──────────────┬──────────────┘
                                   │
                                   ▼
                    ┌─────────────────────────────┐
                    │  从 SSO 响应提取关键字段     │
                    │  - email (邮箱)              │
                    │  - providerId (外部用户ID)  │
                    └──────────────┬──────────────┘
                                   │
                                   ▼
              ┌────────────────────────────────────────┐
              │  步骤 1: providerId 精确匹配           │
              │  SELECT * FROM user_authentications    │
              │  WHERE providerId = ?                   │
              │  AND user.teamId = current_teamId      │
              └───────────────────┬────────────────────┘
                                  │
                  ┌───────────────┴───────────────┐
                  │                               │
              有匹配                          无匹配
                  │                               │
                  ▼                               ▼
    ┌───────────────────────┐       ┌───────────────────────────┐
    │  场景 A: 已有 SSO 绑定 │       │  进入步骤 2: email 模糊匹配 │
    │  - 更新令牌           │       │                           │
    │  - 更新用户信息       │       │  SELECT * FROM users      │
    │  - 自动迁移认证提供者 │       │  WHERE email ILIKE ?      │
    │  (如有变更)           │       │  AND teamId = ?           │
    └───────────────────────┘       └─────────────┬─────────────┘
                                                    │
                                        ┌───────────┴───────────┐
                                        │                       │
                                    有匹配                  无匹配
                                        │                       │
                                        ▼                       ▼
                            ┌───────────────────┐   ┌─────────────────────┐
                            │  场景 B: 邮箱匹配  │   │  场景 C: 全新用户    │
                            │  合并到现有账号    │   │  创建新用户和认证记录 │
                            └─────────┬─────────┘   └─────────────────────┘
                                      │
                                      ▼
                            ┌─────────────────────────────┐
                            │  为现有用户创建新的          │
                            │  UserAuthentication 记录    │
                            │  (允许一个用户绑定多个 SSO)  │
                            └─────────────────────────────┘
```

### 3.3 场景详解

#### 场景 A：已有 SSO 绑定（providerId 匹配）

**代码位置**：`userProvisioner.ts:59-119`

```typescript
// 1. 查找匹配的认证记录
const auth = authentication
  ? await UserAuthentication.findOne({
      where: {
        providerId: String(authentication.providerId),
      },
      include: [
        {
          model: User,
          as: "user",
          where: { teamId },
          required: true,
        },
      ],
    })
  : undefined;

// 2. 找到匹配，更新信息
if (auth && authentication) {
  const { providerId, authenticationProviderId, ...rest } = authentication;
  const { user } = auth;

  // 2a. 自动迁移：认证提供者变更（如公司从 Google 迁到 Azure）
  if (auth.authenticationProviderId !== authenticationProviderId) {
    Logger.info("authentication", "Migrating user to new authentication provider", {
      userId: user?.id,
      fromAuthenticationProviderId: auth.authenticationProviderId,
      toAuthenticationProviderId: authenticationProviderId,
    });
    await auth.update({ authenticationProviderId });
  }

  // 2b. 更新用户信息（以 SSO 返回的最新信息为准）
  if (user) {
    if (avatarUrl && !user.getFlag(UserFlag.AvatarUpdated)) {
      await new UploadUserAvatarTask().schedule({ userId: user.id, avatarUrl });
    }
    await user.update({ email });  // 邮箱可能变更
    await auth.update(rest);       // 更新令牌等

    return { user, authentication: auth, isNewUser: false };
  }

  // 2c. 用户已删除，清理无效认证记录
  await auth.destroy();
}
```

**边界情况：认证提供者自动迁移**

触发条件：`auth.authenticationProviderId !== authenticationProviderId`

常见场景：
- 公司从 Google Workspace 迁移到 Azure AD
- 更换 OIDC Identity Provider
- 域名变更导致 providerId 变化

处理方式：直接更新 `UserAuthentication.authenticationProviderId`，用户无感知。

---

#### 场景 B：邮箱匹配（合并到现有账号）

**代码位置**：`userProvisioner.ts:122-200`

```typescript
// 1. 通过邮箱查找现有用户（不区分大小写）
const existingUser = await User.scope(["withTeam"]).findOne({
  where: {
    email: {
      [Op.iLike]: email,  // ILIKE = 大小写不敏感
    },
    teamId,
  },
});

// 2. 找到匹配用户
if (existingUser) {
  const isInvite = existingUser.isInvited;  // lastActiveAt === null

  // 2a. 在事务中执行合并
  const userAuth = await sequelize.transaction(async (transaction) => {
    // 更新现有用户的信息（以 SSO 最新信息为准）
    await existingUser.update(
      {
        name,
        avatarUrl,
        lastActiveAt: new Date(),  // 标记为已激活
        lastActiveIp: ctx.ip,
      },
      { transaction }
    );

    // 为现有用户创建新的 SSO 认证记录
    // 关键点：一个用户可以有多个 UserAuthentication 记录
    if (!authentication) {
      return null;
    }
    return await existingUser.$create<UserAuthentication>(
      "authentication",
      authentication,
      { transaction }
    );
  });

  // 2b. 邀请用户的特殊处理
  if (isInvite) {
    // 发送欢迎邮件
    await new WelcomeEmail({ ... }).schedule();
    
    // 通知邀请人
    const inviter = await existingUser.$get("invitedBy");
    if (inviter) {
      await new InviteAcceptedEmail({ ... }).schedule();
    }
  }

  return {
    user: existingUser,
    authentication: userAuth,
    isNewUser: isInvite,  // 邀请用户接受邀请视为"新"用户
  };
}
```

**关键设计要点**：

| 设计决策 | 说明 |
|---------|------|
| `Op.iLike` 邮箱匹配 | 不区分大小写，如 `User@Company.Com` 匹配 `user@company.com` |
| 多认证记录支持 | 一个用户可绑定多个 SSO 提供者（Google + Azure + OIDC） |
| 信息覆盖策略 | SSO 返回的 `name`、`avatarUrl` 覆盖现有值 |
| 邀请用户激活 | `lastActiveAt` 从 `null` 变为当前时间，`isNewUser` 返回 `true` |

---

#### 场景 C：全新用户

**代码位置**：`userProvisioner.ts:201-259`

```typescript
// 1. 检查团队设置
if (team?.inviteRequired) {
  throw InviteRequiredError();  // 需要邀请才能加入
}

if (team && !(await team.isDomainAllowed(email))) {
  throw DomainNotAllowedError();  // 邮箱域名不在白名单
}

// 2. 创建新用户和认证记录
const user = await User.createWithCtx(
  ctx,
  {
    name,
    email,
    language,
    role: role ?? team?.defaultUserRole,
    teamId,
    avatarUrl,
    authentications: authentication ? [authentication] : [],
    lastActiveAt: new Date(),
    lastActiveIp: ctx.ip,
  },
  undefined,
  { include: "authentications" }
);
```

---

## 4. 冲突判定逻辑

### 4.1 冲突类型与判定

| 冲突类型 | 触发条件 | 判定位置 | 错误类型 |
|---------|---------|---------|---------|
| **邮箱域名冲突** | 邮箱域名不在团队白名单 | `userProvisioner.ts:228-230` | `DomainNotAllowedError` |
| **邀请制冲突** | 团队要求邀请，但用户未被邀请 | `userProvisioner.ts:217-224` | `InviteRequiredError` |
| **认证提供者冲突** | 使用未启用的认证提供者登录 | `accountProvisioner.ts:174-176` | `AuthenticationProviderDisabledError` |
| **云托管团队冲突** | 云托管模式下使用未知认证提供者 | `teamProvisioner.ts:95-104` | `InvalidAuthenticationError` |
| **用户挂起冲突** | 用户或团队被挂起 | `authentication.ts:38-43` | 重定向到 `notice=user-suspended` |
| **团队待删除冲突** | 团队已标记删除 | `teamProvisioner.ts:82-84` | `TeamPendingDeletionError` |
| **个人 Gmail 限制** | 个人 Gmail 尝试创建新团队 | `google.ts:78-96` | `GmailAccountCreationError` / `TeamDomainRequiredError` |

### 4.2 邮箱域名白名单冲突详解

**代码位置**：`userProvisioner.ts:228-230`

```typescript
// 仅在创建新用户时检查
if (team && !(await team.isDomainAllowed(email))) {
  throw DomainNotAllowedError();
}
```

**`Team.isDomainAllowed` 逻辑**（推断）：
1. 如果团队有 `allowedDomains` 配置，检查邮箱后缀是否匹配
2. 如果团队没有配置域名限制，允许任意域名

**注意**：邮箱匹配（场景 B）不检查域名白名单！
- 原因：现有用户已在团队中，通过邀请或之前的合法途径加入
- 影响：即使域名后来从白名单移除，该用户仍可通过 SSO 登录

### 4.3 认证提供者冲突详解

**代码位置**：`accountProvisioner.ts:174-176`

```typescript
// 在 teamProvisioner 之后，userProvisioner 之前检查
if (!authenticationProvider.enabled) {
  throw AuthenticationProviderDisabledError();
}
```

**关键点**：
- 检查的是 `AuthenticationProvider.enabled`，而不是全局插件启用状态
- 云托管模式更严格：必须 `enabled === true`
- 自托管模式较宽松：`enabled !== false` 即可（即 `null` 或 `undefined` 也视为启用）

**自托管 vs 云托管差异**（`AuthenticationHelper.ts:77-80`）：

```typescript
return (
  (!isCloudHosted && authProvider?.enabled !== false) ||  // 自托管：非禁用即可
  (isCloudHosted && authProvider?.enabled)                  // 云托管：必须显式启用
);
```

### 4.4 个人 Gmail 账号限制

**代码位置**：`google.ts:68-96`

```typescript
// 无 domain (hd = hosted domain) 且无团队上下文
if (!domain && !team) {
  const userExists = await User.count({
    where: { email: profile.email.toLowerCase() },
    include: [{ association: "team", required: true }],
  });

  // 个人 Gmail 不能创建新团队
  if (!userExists) {
    throw GmailAccountCreationError();
  }
  
  // 必须通过子域名指定团队
  throw TeamDomainRequiredError();
}
```

**设计意图**：
- 防止用户使用个人 Gmail 创建团队（需要 Google Workspace 账号）
- 但允许个人 Gmail 登录已有团队（通过子域名访问）

---

## 5. 回退策略

### 5.1 emailMatchOnly 模式

**代码位置**：`accountProvisioner.ts:127-169`

```typescript
try {
  result = await teamProvisioner(ctx, { ... });
} catch (err) {
  // 团队配置失败时，尝试仅通过邮箱匹配的回退模式
  if (err.id === "invalid_authentication") {
    const authProvider = await AuthenticationProvider.findOne({
      where: {
        name: authenticationProviderParams.name,
        teamId: teamParams.teamId,
      },
      include: [{ model: Team, as: "team", required: true }],
      order: [["enabled", "DESC"]],
    });

    if (authProvider) {
      emailMatchOnly = true;
      result = {
        authenticationProvider: authProvider,
        team: authProvider.team,
        isNewTeam: false,
      };
    }
  }
}

// 传递给 userProvisioner
result = await userProvisioner(ctx, {
  // ...
  authentication: emailMatchOnly
    ? undefined  // 关键：不创建 UserAuthentication 记录
    : {
        authenticationProviderId: authenticationProvider.id,
        ...authenticationParams,
      },
});
```

**emailMatchOnly 模式的影响**：

| 行为 | 正常模式 | emailMatchOnly 模式 |
|-----|---------|---------------------|
| providerId 匹配 | ✅ 执行 | ❌ 跳过（`authentication` 为 `undefined`） |
| 创建认证记录 | ✅ 创建 `UserAuthentication` | ❌ 不创建 |
| 邮箱匹配 | ✅ 执行 | ✅ 仍执行 |
| 创建新用户 | ✅ 允许（受设置限制） | ❌ 可能失败（见下文） |

**emailMatchOnly 模式下创建新用户的失败场景**：

```typescript
// userProvisioner.ts:201-207
else if (!authentication && !team?.allowedDomains.length) {
  // 无认证信息，且无域名白名单
  // 无法判断是否允许该用户加入
  throw InvalidAuthenticationError(
    "No matching user for email or allowed domain"
  );
}
```

### 5.2 邮箱登录的 SSO 重定向回退

**代码位置**：`email.ts:64-73`

```typescript
// 用户尝试使用邮箱登录时
const user = await User.scope("withAuthentications").findOne({
  where: {
    teamId: team.id,
    email: email.toLowerCase(),
  },
});

// 如果用户已有 SSO 认证记录，重定向到 SSO 登录页
if (user.authentications.length) {
  const authenticationProvider = user.authentications[0].authenticationProvider;
  ctx.body = {
    redirect: `${team.url}/auth/${authenticationProvider?.name}`,
  };
  return;
}

// 否则发送邮箱魔法链接
```

**设计意图**：
- 安全性：SSO 通常比邮箱链接更安全
- 用户体验：引导用户使用更安全的认证方式
- 策略：用户绑定 SSO 后，邮箱登录变为"重定向到 SSO"

### 5.3 令牌过期与刷新

**代码位置**：`UserAuthentication.ts:90-134`

```typescript
public async validateAccess(
  options: SaveOptions = {},
  force = false
): Promise<boolean> {
  // 5 分钟内已验证过则跳过
  if (this.lastValidatedAt > subMinutes(Date.now(), 5) && !force) {
    return true;
  }

  try {
    // 刷新令牌（如果即将过期）
    await this.refreshAccessTokenIfNeeded(authenticationProvider, options);

    // 调用第三方验证令牌有效性
    const client = authenticationProvider.oauthClient;
    if (client) {
      await client.userInfo(this.accessToken);
    }

    this.lastValidatedAt = new Date();
    await this.save({ transaction: options.transaction });
    return true;
  } catch (error) {
    if (error.id === "authentication_required") {
      return false;  // 令牌无效，需要重新登录
    }
    throw error;  // 其他错误重试
  }
}
```

**触发验证的场景**（`auth.ts:143-161`）：

```typescript
const isOAuthSession = !service || !NON_SSO_SERVICES.includes(service);

// SSO 会话且上次登录超过 1 小时
if (isOAuthSession && user.lastSignedInAt && 
    user.lastSignedInAt < subHours(new Date(), 1)) {
  await new ValidateSSOAccessTask().schedule(
    { userId: user.id },
    { jobId: `validate-sso:${user.id}` }
  );
}
```

### 5.4 认证记录与用户不同步的清理

**代码位置**：`userProvisioner.ts:116-119`

```typescript
// 找到了 UserAuthentication，但关联的 User 不存在
// 可能是用户被硬删除，或数据不一致
if (user) {
  // ... 正常处理
} else {
  // 清理孤立的认证记录
  await auth.destroy();
  // 继续执行，可能创建新用户或通过邮箱匹配
}
```

---

## 6. 边界场景矩阵

### 6.1 登录场景汇总表

| # | 场景描述 | providerId 匹配 | email 匹配 | 预期行为 | 关键代码 |
|---|---------|----------------|-----------|---------|---------|
| 1 | 同一用户重复 SSO 登录 | ✅ | 任意 | 更新令牌和用户信息 | `userProvisioner.ts:77-113` |
| 2 | 更换 SSO 提供者（同邮箱同用户） | ✅ (不同 authProviderId) | ✅ | 自动迁移认证提供者 | `userProvisioner.ts:85-97` |
| 3 | 已有本地账号，首次 SSO 登录（同邮箱） | ❌ | ✅ | 合并：为用户添加新认证记录 | `userProvisioner.ts:140-174` |
| 4 | 接受邀请（被邀请用户通过 SSO 登录） | ❌ | ✅ | 激活邀请用户，发送通知 | `userProvisioner.ts:182-199` |
| 5 | 全新用户，域名白名单 | ❌ | ❌ | 创建新用户和认证记录 | `userProvisioner.ts:232-255` |
| 6 | 全新用户，域名不在白名单 | ❌ | ❌ | 抛出 `DomainNotAllowedError` | `userProvisioner.ts:228-230` |
| 7 | 团队要求邀请制，用户未被邀请 | ❌ | ❌ | 抛出 `InviteRequiredError` | `userProvisioner.ts:217-224` |
| 8 | 认证提供者已禁用 | 任意 | 任意 | 抛出 `AuthenticationProviderDisabledError` | `accountProvisioner.ts:174-176` |
| 9 | 团队待删除 | 任意 | 任意 | 抛出 `TeamPendingDeletionError` | `teamProvisioner.ts:82-84` |
| 10 | 用户被挂起 | 任意 | 任意 | 重定向到 `notice=user-suspended` | `authentication.ts:41-43` |
| 11 | 个人 Gmail 创建新团队 | 不适用 | 不适用 | 抛出 `GmailAccountCreationError` | `google.ts:89-92` |
| 12 | 个人 Gmail 登录已有团队 | 不适用 | 需指定子域名 | 允许（通过子域名指定团队） | `google.ts:93-96` |
| 13 | emailMatchOnly 模式，邮箱匹配现有用户 | 跳过 | ✅ | 允许登录，不创建认证记录 | `accountProvisioner.ts:153-194` |
| 14 | emailMatchOnly 模式，无匹配用户 | 跳过 | ❌ | 可能抛出 `InvalidAuthenticationError` | `userProvisioner.ts:201-207` |
| 15 | 邮箱大小写不一致（`User@Domain.Com` vs `user@domain.com`） | 取决于 providerId | ✅ (Op.iLike) | 正确匹配到同一用户 | `userProvisioner.ts:127-129` |

### 6.2 多认证提供者绑定场景

| 场景 | 用户状态 | 操作 | 结果 |
|-----|---------|------|------|
| A | 已通过 Google 登录 | 用同一邮箱通过 Azure 登录 | 同一用户，两条 UserAuthentication 记录 |
| B | 已通过 Google 登录 | 用不同邮箱通过 Azure 登录 | 两个独立用户 |
| C | 已邀请（本地账号，无认证） | 用邀请邮箱通过 OIDC 登录 | 合并到邀请用户，激活账号 |
| D | 已通过邮箱链接登录 | 用同一邮箱通过 Google 登录 | 添加 Google 认证记录，以后邮箱登录重定向到 Google |

### 6.3 管理员操作场景

| 场景 | 权限要求 | 代码位置 | 行为 |
|-----|---------|---------|------|
| 管理员添加新认证提供者 | 已登录 + 是管理员 + 同一团队 | `accountProvisioner.ts:103-125` | 创建新的 AuthenticationProvider，不影响用户 |
| 管理员禁用认证提供者 | 管理员 | `AuthenticationProvider.disable()` | 阻止该提供者的新登录，已有用户会话仍有效 |
| 管理员删除认证提供者 | 管理员 | `AuthenticationProvider.beforeDestroy()` | 检查是否是最后一个启用的提供者，是则禁止删除 |

---

## 7. 决策流程详解图

### 7.1 完整登录决策流程

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              入口：SSO 回调处理                                  │
│                    (passportMiddleware → accountProvisioner)                  │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  Phase 1: 团队解析 (teamProvisioner)                                           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  输入: teamId (可选，来自子域名/上下文)                                        │
│        authenticationProvider: { name, providerId }                          │
│                                                                               │
│  查找逻辑:                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ 1. 优先查找: name + providerId + teamId (如有)                           ││
│  │    WHERE authentication_providers.name = ?                               ││
│  │      AND authentication_providers.providerId = ?                          ││
│  │      AND authentication_providers.teamId = ?                              ││
│  │    INNER JOIN teams ON teams.id = authentication_providers.teamId        ││
│  │    ORDER BY enabled DESC                                                   ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                               │                                               │
│              ┌────────────────┴────────────────┐                              │
│              │                                 │                              │
│          找到匹配                          未找到匹配                           │
│              │                                 │                              │
│              ▼                                 ▼                              │
│  ┌─────────────────────┐         ┌─────────────────────────────────────┐    │
│  │ 检查团队是否被软删除  │         │ 云托管?                              │    │
│  │ (paranoid: false)    │         │                                     │    │
│  └──────────┬──────────┘         │ 是 → 抛出 InvalidAuthenticationError │    │
│             │                    │ 否 → 自托管模式检查                   │    │
│             ▼                    │      ┌─────────────────────────────┐ │    │
│    ┌─────────────────┐           │      │ 1. 检查 domain 是否在        │ │    │
│    │ deletedAt 有值? │           │      │    allowedDomains            │ │    │
│    └────────┬────────┘           │      └──────────────┬──────────────┘ │    │
│             │                     │                     │                 │    │
│        是 ──┴── 否               │                是 ───┴── 否            │    │
│        │         │               │                │         │            │    │
│        ▼         ▼               │                ▼         ▼            │    │
│  ┌──────────┐ ┌─────────────┐    │    ┌──────────────┐ ┌─────────────┐  │    │
│  │ 抛出     │ │ 返回现有    │    │    │ 创建新的     │ │ 抛出        │  │    │
│  │ TeamPending│ │ 团队和提供者 │    │    │ AuthProvider │ │ DomainNot   │  │    │
│  │ Deletion   │ └─────────────┘    │    └──────────────┘ │ AllowedError│  │    │
│  │ Error     │                    │                     └─────────────┘  │    │
│  └──────────┘                    └─────────────────────────────────────────┘    │
│                                                                               │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  检查点: 认证提供者是否启用?                                                    │
│  IF !authenticationProvider.enabled                                            │
│    → 抛出 AuthenticationProviderDisabledError                                  │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  Phase 2: 用户解析 (userProvisioner)                                           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  输入: user: { name, email, avatarUrl }                                      │
│        authentication: { providerId, accessToken, ... } 或 undefined         │
│                      (emailMatchOnly 模式下为 undefined)                       │
│                                                                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  步骤 1: providerId 精确匹配 (仅当 authentication 有值时)                     │
│  ─────────────────────────────────────────────────────────────────────────   │
│                                                                               │
│  SELECT * FROM user_authentications                                          │
│  INNER JOIN users ON users.id = user_authentications.userId                  │
│  WHERE user_authentications.providerId = ?                                   │
│    AND users.teamId = ?                                                       │
│                                                                               │
│  找到?                                                                        │
│  ├── 是: 更新流程                                                             │
│  │   ├── 检查 authenticationProviderId 是否变更                               │
│  │   │   └── 是: 自动迁移 (更新 authenticationProviderId)                      │
│  │   ├── 更新用户: email, avatarUrl (如果用户未手动更新过头像)                 │
│  │   ├── 更新认证记录: accessToken, refreshToken, expiresAt, scopes          │
│  │   └── 返回: { user, authentication: auth, isNewUser: false }              │
│  │                                                                             │
│  └── 否或 emailMatchOnly: 进入步骤 2                                          │
│                                                                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  步骤 2: email 模糊匹配 (Op.iLike = 大小写不敏感)                             │
│  ─────────────────────────────────────────────────────────────────────────   │
│                                                                               │
│  SELECT * FROM users                                                          │
│  WHERE email ILIKE ?                                                          │
│    AND teamId = ?                                                             │
│                                                                               │
│  找到?                                                                        │
│  ├── 是: 合并流程                                                             │
│  │   ├── 判断是否为邀请用户: isInvite = (user.lastActiveAt IS NULL)          │
│  │   ├── 在事务中执行:                                                         │
│  │   │   ├── 更新用户: name, avatarUrl, lastActiveAt = NOW(), lastActiveIp    │
│  │   │   └── 如有 authentication: 创建新的 UserAuthentication 记录            │
│  │   ├── 邀请用户特殊处理:                                                      │
│  │   │   ├── 发送 WelcomeEmail                                                 │
│  │   │   └── 如有邀请人: 发送 InviteAcceptedEmail                              │
│  │   └── 返回: { user, authentication, isNewUser: isInvite }                  │
│  │                                                                             │
│  └── 否: 进入步骤 3 (创建新用户或错误)                                         │
│                                                                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  步骤 3: 创建新用户或错误                                                      │
│  ─────────────────────────────────────────────────────────────────────────   │
│                                                                               │
│  检查 1: emailMatchOnly 且无域名白名单?                                        │
│  IF !authentication AND !team.allowedDomains.length                          │
│    → 抛出 InvalidAuthenticationError                                          │
│                                                                               │
│  检查 2: 团队要求邀请?                                                         │
│  IF team.inviteRequired                                                       │
│    → 抛出 InviteRequiredError                                                 │
│                                                                               │
│  检查 3: 域名在白名单?                                                         │
│  IF team AND !team.isDomainAllowed(email)                                     │
│    → 抛出 DomainNotAllowedError                                               │
│                                                                               │
│  通过所有检查:                                                                 │
│  ├── 创建 User 记录                                                           │
│  ├── 如有 authentication: 创建 UserAuthentication 记录                        │
│  ├── 新团队或无欢迎集合: 预配 Welcome 集合和文档                               │
│  └── 返回: { user, authentication, isNewUser: true }                          │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. 关键配置与环境变量

### 8.1 认证相关环境变量

| 变量名 | 说明 | 适用场景 |
|-------|------|---------|
| `OIDC_CLIENT_ID` | OIDC 客户端 ID | OIDC |
| `OIDC_CLIENT_SECRET` | OIDC 客户端密钥 | OIDC |
| `OIDC_ISSUER_URL` | OIDC 发行者 URL（自动发现） | OIDC |
| `OIDC_AUTH_URI` | OIDC 授权端点 | OIDC（手动配置） |
| `OIDC_TOKEN_URI` | OIDC 令牌端点 | OIDC（手动配置） |
| `OIDC_USERINFO_URI` | OIDC 用户信息端点 | OIDC（手动配置） |
| `OIDC_SCOPES` | OIDC 授权范围（默认 `openid email profile`） | OIDC |
| `OIDC_USERNAME_CLAIM` | OIDC 用户名字段（默认 `preferred_username`） | OIDC |
| `GOOGLE_CLIENT_ID` | Google OAuth 客户端 ID | Google |
| `GOOGLE_CLIENT_SECRET` | Google OAuth 客户端密钥 | Google |
| `AZURE_CLIENT_ID` | Azure AD 客户端 ID | Azure |
| `AZURE_CLIENT_SECRET` | Azure AD 客户端密钥 | Azure |
| `AZURE_TENANT_ID` | Azure AD 租户 ID（可选，不指定用 common） | Azure |
| `SMTP_HOST` / `SMTP_SERVICE` | 邮件服务配置 | 邮箱登录 |
| `URL` | 应用基础 URL | 所有 SSO 回调 |

### 8.2 团队级别设置

| 设置 | 模型字段 | 说明 |
|-----|---------|------|
| 邮箱登录启用 | `Team.emailSigninEnabled` | 是否允许邮箱魔法链接登录 |
| Passkey 启用 | `Team.passkeysEnabled` | 是否允许 WebAuthn 登录 |
| 邀请制 | `Team.inviteRequired` | 是否需要邀请才能加入 |
| 默认用户角色 | `Team.defaultUserRole` | 新用户默认角色 |
| 域名白名单 | `TeamDomains` (关联表) | 允许的邮箱域名 |

---

## 9. 代码索引

### 9.1 核心命令

| 文件 | 职责 | 关键函数 |
|-----|------|---------|
| `server/commands/accountProvisioner.ts` | 账号供应主入口 | `accountProvisioner()` |
| `server/commands/teamProvisioner.ts` | 团队解析与创建 | `teamProvisioner()` |
| `server/commands/userProvisioner.ts` | 用户解析、合并、创建 | `userProvisioner()` |
| `server/commands/userInviter.ts` | 用户邀请 | `userInviter()` |

### 9.2 认证插件

| 插件 | 主路由文件 | 策略文件 |
|-----|-----------|---------|
| OIDC | `plugins/oidc/server/auth/oidcRouter.ts` | `OIDCStrategy.ts` |
| Google | `plugins/google/server/auth/google.ts` | (使用 passport-google-oauth2) |
| Azure AD | `plugins/azure/server/auth/azure.ts` | (使用 passport-azure-ad-oauth2) |
| 邮箱 | `plugins/email/server/auth/email.ts` | (自定义，无 passport) |
| Passkey | `plugins/passkeys/server/auth/passkeys.ts` | (WebAuthn) |

### 9.3 模型

| 模型 | 文件 | 关键关联 |
|-----|------|---------|
| Team | `server/models/Team.ts` | hasMany AuthenticationProvider |
| User | `server/models/User.ts` | hasMany UserAuthentication, belongsTo Team |
| AuthenticationProvider | `server/models/AuthenticationProvider.ts` | belongsTo Team, hasMany UserAuthentication |
| UserAuthentication | `server/models/UserAuthentication.ts` | belongsTo User, belongsTo AuthenticationProvider |
| UserPasskey | `server/models/UserPasskey.ts` | belongsTo User (WebAuthn) |

---

## 10. 测试用例参考

`accountProvisioner.test.ts` 中的关键测试覆盖了以下场景：

| 测试用例 | 覆盖场景 |
|---------|---------|
| `should create a new user and team` | 全新用户和团队 |
| `should update existing user and authentication` | 重复登录更新 |
| `should allow authentication by email matching` | 本地账号 + SSO 合并（场景 3） |
| `should throw an error when authentication provider is disabled` | 认证提供者禁用 |
| `should prioritize enabled authentication provider` | 多提供者启用优先级 |
| `should throw an error when the domain is not allowed` | 域名白名单冲突 |
| `should create a new user in an existing team when the domain is allowed` | 域名白名单允许 |
| `should handle emails with capital letters correctly` | 邮箱大小写不敏感 |
| `should allow connecting a new authentication provider while logged in` | 管理员添加认证提供者 |
| `should fail if existing team and domain not in allowed list` | 自托管域名限制 |
| `should always use existing team if self-hosted` | 自托管团队行为 |

---

## 11. 总结与建议

### 11.1 核心设计原则

1. **双标识匹配**：providerId 精确匹配优先，email 模糊匹配作为安全网
2. **多认证支持**：一个用户可绑定多个 SSO 提供者，灵活应对迁移场景
3. **自动迁移**：认证提供者变更时自动更新，用户无感知
4. **配置驱动**：团队级设置（邀请制、域名白名单）控制用户准入
5. **安全回退**：emailMatchOnly 模式处理认证配置不一致的边缘情况

### 11.2 运维建议

| 场景 | 建议操作 |
|-----|---------|
| 公司从 Google 迁移到 Azure | 配置 Azure AD，用户首次登录时自动合并（通过邮箱匹配） |
| 公司域名变更 | 新用户通过 providerId 匹配，自动迁移 authenticationProviderId |
| 启用 SSO 后禁用邮箱登录 | 设置 `emailSigninEnabled = false`，已绑定 SSO 的用户邮箱登录会重定向到 SSO |
| 排查账号合并问题 | 检查 `user_authentications` 表的 `providerId` 和 `email` 字段 |
| 邀请用户激活 | 邀请用户通过 SSO 登录（使用邀请邮箱）会自动激活账号 |

### 11.3 潜在风险点

| 风险 | 说明 | 缓解措施 |
|-----|------|---------|
| 邮箱接管 | 如果攻击者控制了用户邮箱，可通过邮箱匹配接管账号 | SSO 绑定后邮箱登录重定向到 SSO，降低风险 |
| 域名白名单绕过 | 邮箱匹配不检查域名白名单 | 仅限现有用户，新用户仍受白名单限制 |
| 认证提供者混淆 | 不同 IdP 的 `providerId` 可能冲突 | 使用 `(authenticationProviderId, providerId)` 组合唯一标识 |
| 个人 Gmail 滥用 | 个人 Gmail 可能被用来登录团队 | 需要通过子域名指定团队，不能创建新团队 |

---

*报告生成时间: 2026-05-05*
*基于代码提交: 最新工作区状态*
