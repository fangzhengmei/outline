# Outline SSO 账号合并分析报告

## 1. 概述

Outline 作为一个协作知识库系统，支持多种单点登录（SSO）协议，包括 OIDC、Google OAuth 2.0 和 Azure AD。本文档深入分析这些 SSO 协议的接入方式以及账号合并关联的逻辑。

### 1.1 支持的 SSO 协议

| 协议类型 | 实现位置 | 状态 |
|---------|---------|------|
| OIDC (OpenID Connect) | `plugins/oidc/` | ✅ 已实现 |
| Google OAuth 2.0 | `plugins/google/` | ✅ 已实现 |
| Azure AD (Microsoft 365) | `plugins/azure/` | ✅ 已实现 |
| SAML 2.0 | - | ❌ 社区版未实现 |

> 注：SAML 仅在翻译文件中有相关字符串，但代码库中未找到实际实现，可能为企业版独有功能。

---

## 2. 核心数据模型

### 2.1 模型关系图

```
┌─────────────────┐         ┌─────────────────────────┐         ┌─────────────────────┐
│      User       │         │   UserAuthentication    │         │ AuthenticationProvider│
├─────────────────┤         ├─────────────────────────┤         ├─────────────────────┤
│ id (UUID)       │◄────────│ userId (UUID)           │────────►│ id (UUID)           │
│ email           │         │ providerId (string)     │         │ name (string)       │
│ name            │         │ authenticationProviderId │         │ providerId (string) │
│ teamId          │         │ accessToken (encrypted)  │         │ teamId (UUID)       │
│ ...             │         │ refreshToken (encrypted) │         │ enabled (boolean)   │
└─────────────────┘         │ expiresAt (Date)        │         │ settings (JSONB)     │
                            │ scopes (string[])        │         └─────────────────────┘
                            └─────────────────────────┘
```

### 2.2 关键模型说明

#### AuthenticationProvider (`server/models/AuthenticationProvider.ts`)

认证提供者模型，用于标识团队配置的 SSO 认证方式：

```typescript
interface AuthenticationProvider {
  id: string;           // UUID
  name: string;         // 认证类型: "google" | "azure" | "oidc"
  providerId: string;   // 外部标识（如域名、租户ID）
  teamId: string;       // 所属团队
  enabled: boolean;     // 是否启用
  settings: JSONB;      // 额外配置（如组同步设置）
}
```

#### UserAuthentication (`server/models/UserAuthentication.ts`)

用户认证记录，关联用户与认证提供者：

```typescript
interface UserAuthentication {
  id: string;                     // UUID
  userId: string;                 // 关联的用户
  authenticationProviderId: string; // 关联的认证提供者
  providerId: string;             // 外部用户ID（sub、oid 等）
  accessToken: string;            // 加密存储的访问令牌
  refreshToken: string;           // 加密存储的刷新令牌
  expiresAt: Date;                // 令牌过期时间
  scopes: string[];               // 授权范围
}
```

---

## 3. SSO 协议接入实现

### 3.1 OIDC 协议接入 (`plugins/oidc/`)

#### 3.1.1 配置方式

OIDC 支持两种配置模式：

1. **手动配置**：
   - `OIDC_CLIENT_ID` - 客户端ID
   - `OIDC_CLIENT_SECRET` - 客户端密钥
   - `OIDC_AUTH_URI` - 授权端点
   - `OIDC_TOKEN_URI` - 令牌端点
   - `OIDC_USERINFO_URI` - 用户信息端点
   - `OIDC_LOGOUT_URI` - 登出端点（可选）

2. **自动发现**：
   - `OIDC_CLIENT_ID`
   - `OIDC_CLIENT_SECRET`
   - `OIDC_ISSUER_URL` - 通过 `.well-known/openid-configuration` 自动发现端点

#### 3.1.2 用户信息提取

OIDC 用户信息提取逻辑 (`oidcRouter.ts:88-177`)：

```typescript
// 1. 从 userinfo 端点获取用户信息
const profile = await request("GET", endpoints.userInfoURL, accessToken);

// 2. 从 id_token 解码获取补充信息（兼容 ADFS 等不完整实现）
const token = JWT.decode(params.id_token) as { email?: string; sub?: string };

// 3. 提取关键字段
const email = profile.email ?? token.email ?? null;
const name = profile.name || username || profile.username;
const profileId = profile.sub ?? token.sub ?? profile.id;  // 核心关联字段
```

#### 3.1.3 认证提供者匹配策略

```typescript
// 优先匹配：name + teamId + providerId（域名）
let authenticationProvider = await AuthenticationProvider.findOne({
  where: {
    name: "oidc",
    teamId: team.id,
    providerId: domain,  // 从 email 解析的域名
  },
});

// 降级匹配：仅 name + teamId
if (!authenticationProvider) {
  authenticationProvider = await AuthenticationProvider.findOne({
    where: {
      name: "oidc",
      teamId: team.id,
    },
  });
}

// 新提供者：从授权URL主机名派生
const providerId = authenticationProvider?.providerId ?? oidcURL.hostname;
```

### 3.2 Google OAuth 2.0 接入 (`plugins/google/`)

#### 3.2.1 配置项

- `GOOGLE_CLIENT_ID`
- `GOOGLE_CLIENT_SECRET`

#### 3.2.2 特殊处理

1. **个人 Gmail 账号限制**：
```typescript
// 无 domain（hd 字段）且无团队上下文时
if (!domain && !team) {
  const userExists = await User.count({
    where: { email: profile.email.toLowerCase() },
    include: [{ association: "team", required: true }],
  });

  // 个人 Gmail 不能创建新团队
  if (!userExists) {
    throw GmailAccountCreationError();
  }
  // 必须指定团队子域名
  throw TeamDomainRequiredError();
}
```

2. **Google Workspace 域名识别**：
```typescript
const domain = profile._json.hd;  // hd = hosted domain
const providerId = domain ?? "";   // 使用域名作为 providerId
```

### 3.3 Azure AD 接入 (`plugins/azure/`)

#### 3.3.1 配置项

- `AZURE_CLIENT_ID`
- `AZURE_CLIENT_SECRET`
- `AZURE_TENANT_ID`（可选，不指定则使用 common 端点）
- `AZURE_RESOURCE_APP_ID`（可选）

#### 3.3.2 双源用户信息获取

Azure AD 使用 id_token + Microsoft Graph API 双重获取：

```typescript
// 1. 从 id_token 解码基础信息
const profile = jwt.decode(params.id_token) as jwt.JwtPayload;

// 2. 调用 Graph API 获取完整用户信息
const [profileResponse, organizationResponse] = await Promise.all([
  request("GET", `https://graph.microsoft.com/v1.0/me`, accessToken),
  request("GET", `https://graph.microsoft.com/v1.0/organization`, accessToken),
]);

// 3. 多源 email 回退
const email = 
  profile.email ||               // id_token 中的 email
  profileResponse.mail ||         // Graph API 的 mail
  profileResponse.userPrincipalName; // Graph API 的 userPrincipalName
```

#### 3.3.3 租户与用户标识

```typescript
const providerId = profile.tid;  // 租户ID (tenant ID)
const profileId = profile.oid;    // 用户对象ID (object ID)
```

---

## 4. 账号合并关联逻辑

### 4.1 核心流程

账号合并的核心逻辑在 `server/commands/` 目录下的三个命令中：

```
accountProvisioner.ts (主入口)
    ├── teamProvisioner.ts (团队配置)
    └── userProvisioner.ts (用户账号合并)
```

### 4.2 账号合并策略详解 (`userProvisioner.ts`)

#### 4.2.1 合并优先级

```
优先级 1: providerId 精确匹配 → 直接关联
优先级 2: email 模糊匹配    → 合并到现有用户
优先级 3: 无匹配           → 创建新用户
```

#### 4.2.2 详细合并逻辑

**第一步：providerId 精确匹配**

```typescript
// 通过外部用户ID查找现有认证记录
const auth = authentication
  ? await UserAuthentication.findOne({
      where: {
        providerId: String(authentication.providerId),
      },
      include: [
        {
          model: User,
          as: "user",
          where: { teamId },  // 限制在当前团队
          required: true,
        },
      ],
    })
  : undefined;
```

**场景 1a：找到匹配的认证记录**

```typescript
if (auth && authentication) {
  const { providerId, authenticationProviderId, ...rest } = authentication;
  const { user } = auth;

  // 自动迁移：认证提供者变更（如更换 Google Workspace 域名）
  if (auth.authenticationProviderId !== authenticationProviderId) {
    Logger.info("authentication", "Migrating user to new authentication provider", {
      userId: user?.id,
      fromAuthenticationProviderId: auth.authenticationProviderId,
      toAuthenticationProviderId: authenticationProviderId,
    });
    await auth.update({ authenticationProviderId });
  }

  // 更新用户信息和令牌
  if (user) {
    if (avatarUrl && !user.getFlag(UserFlag.AvatarUpdated)) {
      await new UploadUserAvatarTask().schedule({ userId: user.id, avatarUrl });
    }
    await user.update({ email });
    await auth.update(rest);

    return { user, authentication: auth, isNewUser: false };
  }

  // 关联用户已被删除，清理无效认证记录
  await auth.destroy();
}
```

**第二步：email 模糊匹配**

```typescript
// 通过邮箱查找现有用户（不区分大小写）
const existingUser = await User.scope(["withTeam"]).findOne({
  where: {
    email: {
      [Op.iLike]: email,  // ILIKE = 不区分大小写
    },
    teamId,
  },
});
```

**场景 2：找到匹配邮箱的用户**

```typescript
if (existingUser) {
  const isInvite = existingUser.isInvited;  // 判断是否为邀请用户

  const userAuth = await sequelize.transaction(async (transaction) => {
    // 更新现有用户的信息（以最新 SSO 信息为准）
    await existingUser.update(
      {
        name,
        avatarUrl,
        lastActiveAt: new Date(),
        lastActiveIp: ctx.ip,
      },
      { transaction }
    );

    // 创建新的认证记录（用户可关联多个 SSO 提供商）
    if (!authentication) {
      return null;
    }
    return await existingUser.$create<UserAuthentication>(
      "authentication",
      authentication,
      { transaction }
    );
  });

  // 邀请用户接受邀请的特殊处理
  if (isInvite) {
    const inviter = await existingUser.$get("invitedBy");
    if (inviter) {
      await new InviteAcceptedEmail({
        to: inviter.email,
        invitedName: existingUser.name,
        teamUrl: existingUser.team.url,
      }).schedule();
    }
  }

  return {
    user: existingUser,
    authentication: userAuth,
    isNewUser: isInvite,  // 邀请用户视为"新"用户
  };
}
```

**第三步：创建全新用户**

```typescript
// 检查团队设置：是否需要邀请
if (team?.inviteRequired) {
  throw InviteRequiredError();
}

// 检查域名白名单
if (team && !(await team.isDomainAllowed(email))) {
  throw DomainNotAllowedError();
}

// 创建新用户及认证记录
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

### 4.3 账号合并场景矩阵

| 场景 | providerId 匹配 | email 匹配 | 行为 |
|-----|----------------|-----------|------|
| 同一用户重复登录 | ✅ | 任意 | 更新令牌和用户信息 |
| 更换认证提供者（同 domain） | ✅ (不同 authProviderId) | 任意 | 自动迁移到新认证提供者 |
| 不同 SSO 但相同邮箱 | ❌ | ✅ | 合并：为现有用户创建新认证记录 |
| 接受邀请（邮箱匹配但无 SSO） | ❌ | ✅ | 合并：邀请用户变为正式用户 |
| 完全新用户 | ❌ | ❌ | 创建新用户和认证记录 |
| 纯邮箱登录（emailMatchOnly 模式） | N/A | ✅ | 仅通过邮箱匹配，不创建认证记录 |

### 4.4 emailMatchOnly 模式

在 `accountProvisioner.ts` 中存在一种特殊的回退模式：

```typescript
try {
  result = await teamProvisioner(ctx, { ... });
} catch (err) {
  // 团队配置失败时，尝试仅通过邮箱匹配
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
    ? undefined  // 不创建认证记录
    : {
        authenticationProviderId: authenticationProvider.id,
        ...authenticationParams,
      },
});
```

**使用场景**：当团队已配置了认证提供者，但用户的 providerId 无法匹配时，允许仅通过邮箱进行登录。

---

## 5. 多 SSO 账号关联机制

### 5.1 用户与认证记录的关系

一个用户可以关联多个认证提供者：

```typescript
// User 模型中的关联
@HasMany(() => UserAuthentication)
authentications: UserAuthentication[];

// 查询时获取所有可用认证
User.scope({
  method: ["withAuthentications"],
});
```

### 5.2 实际示例

假设用户 `user@company.com` 先后通过以下方式登录：

1. **首次通过 Google Workspace 登录**
   - 创建 User: { email: "user@company.com", ... }
   - 创建 UserAuthentication: { providerId: "12345", authenticationProviderId: "google-uuid" }

2. **后续通过 Azure AD 登录（同一邮箱）**
   - 找到现有 User（通过 email 匹配）
   - 为该用户创建新的 UserAuthentication: { providerId: "oid-abc", authenticationProviderId: "azure-uuid" }

3. **最终状态**
   ```
   User (user@company.com)
       ├── UserAuthentication (Google, providerId=12345)
       └── UserAuthentication (Azure AD, providerId=oid-abc)
   ```

### 5.3 登录时的认证选择

用户登录时，系统通过以下方式确定使用哪个认证记录：

1. **通过 SSO 提供商入口登录**：使用对应的 authenticationProviderId
2. **通过任意 SSO 登录后**：通过 providerId 匹配对应记录
3. **通过邮箱匹配**：可为已有用户添加新的认证方式

---

## 6. 安全与边界情况处理

### 6.1 令牌安全

```typescript
// UserAuthentication 模型中的加密字段
@Column(DataType.BLOB)
@Encrypted  // 自动加密
accessToken: string;

@Column(DataType.BLOB)
@Encrypted  // 自动加密
refreshToken: string;
```

### 6.2 过期令牌刷新

```typescript
// validateAccess 方法自动刷新即将过期的令牌
public async validateAccess(options: SaveOptions = {}, force = false): Promise<boolean> {
  // 5分钟内已验证过则跳过
  if (this.lastValidatedAt > subMinutes(Date.now(), 5) && !force) {
    return true;
  }

  // 刷新令牌（如果需要）
  await this.refreshAccessTokenIfNeeded(authenticationProvider, options);

  // 调用第三方验证令牌有效性
  const client = authenticationProvider.oauthClient;
  if (client) {
    await client.userInfo(this.accessToken);
  }

  this.lastValidatedAt = new Date();
  await this.save({ transaction: options.transaction });
  return true;
}
```

### 6.3 用户被删除后的清理

```typescript
// 在 providerId 匹配但用户不存在时
// "We found an authentication record, but the associated user was deleted or
// otherwise didn't exist. Cleanup the auth record and proceed with creating
// a new user."
await auth.destroy();
```

### 6.4 域名变更处理

当公司更换域名或认证提供者时：

```typescript
// 检测到认证提供者ID变更时自动迁移
if (auth.authenticationProviderId !== authenticationProviderId) {
  Logger.info("authentication", "Migrating user to new authentication provider", {
    userId: user?.id,
    providerId,
    fromAuthenticationProviderId: auth.authenticationProviderId,
    toAuthenticationProviderId: authenticationProviderId,
  });
  await auth.update({ authenticationProviderId });
}
```

---

## 7. 配置与运维建议

### 7.1 SSO 配置最佳实践

1. **OIDC 配置**：
   - 优先使用 `OIDC_ISSUER_URL` 自动发现，减少配置错误
   - 确保 IDP 返回 `email` 声明，这是账号合并的关键
   - 推荐使用 `email` + `sub` 双标识

2. **Google Workspace**：
   - 确保用户有 `hd` (hosted domain) 声明
   - 个人 Gmail 账号默认被限制创建新团队

3. **Azure AD**：
   - 配置应用注册时确保 `User.Read` 权限
   - 考虑使用租户特定端点（配置 `AZURE_TENANT_ID`）

### 7.2 账号合并相关的团队设置

```typescript
// 影响账号创建的关键设置
team.inviteRequired     // 是否需要邀请才能加入
team.allowedDomains     // 允许的邮箱域名白名单
team.defaultUserRole    // 新用户默认角色
team.emailSigninEnabled // 是否允许邮箱密码登录
```

### 7.3 迁移场景处理

**场景：公司从 Google Workspace 迁移到 Azure AD**

1. 配置 Azure AD 认证提供者
2. 用户首次使用 Azure AD 登录时：
   - providerId 不匹配（Google 的 sub ≠ Azure 的 oid）
   - email 匹配
   - 系统自动为该用户添加 Azure AD 认证记录
3. 后续用户可通过任意一种方式登录
4. 确认所有用户迁移完成后，可禁用 Google 认证提供者

---

## 8. 代码索引

| 功能 | 文件路径 | 关键函数/类 |
|-----|---------|------------|
| OIDC 认证路由 | `plugins/oidc/server/auth/oidcRouter.ts` | `createOIDCRouter()` |
| OIDC 策略 | `plugins/oidc/server/auth/OIDCStrategy.ts` | `OIDCStrategy` |
| Google 认证 | `plugins/google/server/auth/google.ts` | passport 回调 |
| Azure 认证 | `plugins/azure/server/auth/azure.ts` | passport 回调 |
| 账号供应主入口 | `server/commands/accountProvisioner.ts` | `accountProvisioner()` |
| 团队供应 | `server/commands/teamProvisioner.ts` | `teamProvisioner()` |
| 用户账号合并 | `server/commands/userProvisioner.ts` | `userProvisioner()` |
| 认证提供者模型 | `server/models/AuthenticationProvider.ts` | `AuthenticationProvider` |
| 用户认证模型 | `server/models/UserAuthentication.ts` | `UserAuthentication` |
| 用户模型 | `server/models/User.ts` | `User` |

---

## 9. 总结

Outline 的 SSO 账号合并机制设计合理且灵活：

1. **双标识匹配**：优先通过 providerId（外部用户ID）精确匹配，其次通过 email 模糊匹配
2. **多认证支持**：一个用户可关联多个 SSO 认证提供者
3. **平滑迁移**：支持认证提供者变更时的自动迁移
4. **安全保障**：令牌加密存储、自动刷新、定期验证
5. **配置灵活**：支持邀请制、域名白名单等团队级控制

这种设计确保了用户在使用不同 SSO 协议登录时能够正确关联到同一账号，同时保持了系统的安全性和可维护性。
