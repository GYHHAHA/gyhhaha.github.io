---
short_title: 身份验证
---

# 身份验证 (Authentication)

本指南涵盖了身份验证模式，并展示了如何使用 TanStack Start 实现你自己的身份验证系统。

> **📋 开始之前：** 请查看我们的 [身份验证概览](./authentication-overview.md)，了解所有可用选项，包括合作伙伴解决方案和托管服务。

## 身份验证方案

在 TanStack Start 应用程序中，你有多种身份验证选项：

**托管解决方案：**

1. **[Clerk](https://clerk.dev)** - 带有 UI 组件的完整身份验证平台
2. **[WorkOS](https://workos.com)** - 专注于企业级，具备 SSO 和合规特性
3. **[Better Auth](https://www.better-auth.com/)** - 开源 TypeScript 库
4. **[Auth.js](https://authjs.dev/)** - 支持 80 多个 OAuth 提供商的开源库

**自定义实现 (DIY) 的优势：**

- **完全控制**：对身份验证流程进行完整的自定义
- **无供应商锁定**：拥有你自己的身份验证逻辑和用户数据
- **定制需求**：实现特定的业务逻辑或合规需求
- **成本控制**：没有按用户计费的定价或使用限制

身份验证涉及许多考量因素，包括密码安全、会话管理、速率限制、CSRF 保护以及各种攻击向量。

## 核心概念

### 身份验证 vs 授权

- **身份验证 (Authentication)**：这个用户是谁？（登录/注销）
- **授权 (Authorization)**：这个用户能做什么？（权限/角色）

TanStack Start 通过服务器函数 (Server Functions)、会话 (Sessions) 和路由保护为这两者提供了工具。

## 核心构建块

### 1. 用于身份验证的服务器函数

服务器函数在服务器上安全地处理敏感的身份验证逻辑：

```tsx
import { createServerFn } from "@tanstack/react-start";
import { redirect } from "@tanstack/react-router";

// 登录服务器函数
export const loginFn = createServerFn({ method: "POST" })
  .inputValidator((data: { email: string; password: string }) => data)
  .handler(async ({ data }) => {
    // 验证凭据（在此替换为你的身份验证逻辑）
    const user = await authenticateUser(data.email, data.password);

    if (!user) {
      return { error: "凭据无效" };
    }

    // 创建会话
    const session = await useAppSession();
    await session.update({
      userId: user.id,
      email: user.email,
    });

    // 重定向到受保护区域
    throw redirect({ to: "/dashboard" });
  });

// 注销服务器函数
export const logoutFn = createServerFn({ method: "POST" }).handler(async () => {
  const session = await useAppSession();
  await session.clear();
  throw redirect({ to: "/" });
});

// 获取当前用户
export const getCurrentUserFn = createServerFn({ method: "GET" }).handler(
  async () => {
    const session = await useAppSession();
    const userId = session.data.userId;

    if (!userId) {
      return null;
    }

    return await getUserById(userId);
  },
);
```

### 2. 会话管理

TanStack Start 提供了安全的 HTTP-only cookie 会话：

```tsx
// utils/session.ts
import { useSession } from "@tanstack/react-start/server";

type SessionData = {
  userId?: string;
  email?: string;
  role?: string;
};

export function useAppSession() {
  return useSession<SessionData>({
    // 会话配置
    name: "app-session",
    password: process.env.SESSION_SECRET!, // 至少 32 个字符
    // 可选：自定义 cookie 设置
    cookie: {
      secure: process.env.NODE_ENV === "production",
      sameSite: "lax",
      httpOnly: true,
    },
  });
}
```

### 3. 身份验证上下文

在整个应用程序中共享身份验证状态：

```tsx
// contexts/auth.tsx
import { createContext, useContext, ReactNode } from "react";
import { useServerFn } from "@tanstack/react-start";
import { getCurrentUserFn } from "../server/auth";

type User = {
  id: string;
  email: string;
  role: string;
};

type AuthContextType = {
  user: User | null;
  isLoading: boolean;
  refetch: () => void;
};

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const { data: user, isLoading, refetch } = useServerFn(getCurrentUserFn);

  return (
    <AuthContext.Provider value={{ user, isLoading, refetch }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth 必须在 AuthProvider 中使用");
  }
  return context;
}
```

### 4. 路由保护

使用 `beforeLoad` 保护路由：

```tsx
// routes/_authed.tsx - 受保护页面的布局路由
import { createFileRoute, redirect } from "@tanstack/react-router";
import { getCurrentUserFn } from "../server/auth";

export const Route = createFileRoute("/_authed")({
  beforeLoad: async ({ location }) => {
    const user = await getCurrentUserFn();

    if (!user) {
      throw redirect({
        to: "/login",
        search: { redirect: location.href },
      });
    }

    // 将用户数据传递给子路由
    return { user };
  },
});
```

```tsx
// routes/_authed/dashboard.tsx - 受保护路由
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/_authed/dashboard")({
  component: DashboardComponent,
});

function DashboardComponent() {
  const { user } = Route.useRouteContext();

  return (
    <div>
      <h1>欢迎回来，{user.email}！</h1>
      {/* 仪表板内容 */}
    </div>
  );
}
```

## 实现模式

### 基础 邮箱/密码 身份验证

```tsx
// server/auth.ts
import bcrypt from "bcryptjs";
import { createServerFn } from "@tanstack/react-start";

// 用户注册
export const registerFn = createServerFn({ method: "POST" })
  .inputValidator(
    (data: { email: string; password: string; name: string }) => data,
  )
  .handler(async ({ data }) => {
    // 检查用户是否已存在
    const existingUser = await getUserByEmail(data.email);
    if (existingUser) {
      return { error: "用户已存在" };
    }

    // 哈希处理密码
    const hashedPassword = await bcrypt.hash(data.password, 12);

    // 创建用户
    const user = await createUser({
      email: data.email,
      password: hashedPassword,
      name: data.name,
    });

    // 创建会话
    const session = await useAppSession();
    await session.update({ userId: user.id });

    return { success: true, user: { id: user.id, email: user.email } };
  });

async function authenticateUser(email: string, password: string) {
  const user = await getUserByEmail(email);
  if (!user) return null;

  const isValid = await bcrypt.compare(password, user.password);
  return isValid ? user : null;
}
```

### 基于角色的访问控制 (RBAC)

```tsx
// utils/auth.ts
export const roles = {
  USER: "user",
  ADMIN: "admin",
  MODERATOR: "moderator",
} as const;

type Role = (typeof roles)[keyof typeof roles];

export function hasPermission(userRole: Role, requiredRole: Role): boolean {
  const hierarchy = {
    [roles.USER]: 0,
    [roles.MODERATOR]: 1,
    [roles.ADMIN]: 2,
  };

  return hierarchy[userRole] >= hierarchy[requiredRole];
}

// 带有角色检查的受保护路由
export const Route = createFileRoute("/_authed/admin/")({
  beforeLoad: async ({ context }) => {
    if (!hasPermission(context.user.role, roles.ADMIN)) {
      throw redirect({ to: "/unauthorized" });
    }
  },
});
```

### 社交平台身份验证集成

```tsx
// OAuth 提供商示例
export const authProviders = {
  google: {
    clientId: process.env.GOOGLE_CLIENT_ID!,
    redirectUri: `${process.env.APP_URL}/auth/google/callback`,
  },
  github: {
    clientId: process.env.GITHUB_CLIENT_ID!,
    redirectUri: `${process.env.APP_URL}/auth/github/callback`,
  },
};

export const initiateOAuthFn = createServerFn({ method: "POST" })
  .inputValidator((data: { provider: "google" | "github" }) => data)
  .handler(async ({ data }) => {
    const provider = authProviders[data.provider];
    const state = generateRandomState();

    // 将 state 存入会话以进行 CSRF 保护
    const session = await useAppSession();
    await session.update({ oauthState: state });

    // 生成 OAuth URL
    const authUrl = generateOAuthUrl(provider, state);

    throw redirect({ href: authUrl });
  });
```

### 密码重置流程

```tsx
// 密码重置请求
export const requestPasswordResetFn = createServerFn({ method: "POST" })
  .inputValidator((data: { email: string }) => data)
  .handler(async ({ data }) => {
    const user = await getUserByEmail(data.email);
    if (!user) {
      // 不要泄露邮箱是否存在
      return { success: true };
    }

    const token = generateSecureToken();
    const expires = new Date(Date.now() + 60 * 60 * 1000); // 1 小时后过期

    await savePasswordResetToken(user.id, token, expires);
    await sendPasswordResetEmail(user.email, token);

    return { success: true };
  });

// 密码重置确认
export const resetPasswordFn = createServerFn({ method: "POST" })
  .inputValidator((data: { token: string; newPassword: string }) => data)
  .handler(async ({ data }) => {
    const resetToken = await getPasswordResetToken(data.token);

    if (!resetToken || resetToken.expires < new Date()) {
      return { error: "令牌无效或已过期" };
    }

    const hashedPassword = await bcrypt.hash(data.newPassword, 12);
    await updateUserPassword(resetToken.userId, hashedPassword);
    await deletePasswordResetToken(data.token);

    return { success: true };
  });
```

## 安全最佳实践

### 1. 密码安全

```tsx
// 使用强哈希算法 (bcrypt, scrypt, 或 argon2)
import bcrypt from "bcryptjs";

const saltRounds = 12; // 根据你的安全需求进行调整
const hashedPassword = await bcrypt.hash(password, saltRounds);
```

### 2. 会话安全

```tsx
// 使用安全的会话配置
export function useAppSession() {
  return useSession({
    name: "app-session",
    password: process.env.SESSION_SECRET!, // 32 位以上字符
    cookie: {
      secure: process.env.NODE_ENV === "production", // 仅在生产环境中使用 HTTPS
      sameSite: "lax", // CSRF 保护
      httpOnly: true, // XSS 保护
      maxAge: 7 * 24 * 60 * 60, // 7 天
    },
  });
}
```

### 3. 速率限制 (Rate Limiting)

```tsx
// 简单的内存速率限制（生产环境请使用 Redis）
const loginAttempts = new Map<string, { count: number; resetTime: number }>();

export const rateLimitLogin = (ip: string): boolean => {
  const now = Date.now();
  const attempts = loginAttempts.get(ip);

  if (!attempts || now > attempts.resetTime) {
    loginAttempts.set(ip, { count: 1, resetTime: now + 15 * 60 * 1000 }); // 15 分钟
    return true;
  }

  if (attempts.count >= 5) {
    return false; // 尝试次数过多
  }

  attempts.count++;
  return true;
};
```

### 4. 输入验证

```tsx
import { z } from "zod";

const loginSchema = z.object({
  email: z.string().email().max(255),
  password: z.string().min(8).max(100),
});

export const loginFn = createServerFn({ method: "POST" })
  .inputValidator((data) => loginSchema.parse(data))
  .handler(async ({ data }) => {
    // 此时 data 已经过验证
  });
```

## 测试身份验证 (Testing Authentication)

### 服务器函数单元测试

```tsx
// __tests__/auth.test.ts
import { describe, it, expect, beforeEach } from "vitest";
import { loginFn } from "../server/auth";

describe("身份验证", () => {
  beforeEach(async () => {
    // 设置测试数据库
    await setupTestDatabase();
  });

  it("使用有效的凭据应当登录成功", async () => {
    const result = await loginFn({
      data: { email: "test@example.com", password: "password123" },
    });

    expect(result.error).toBeUndefined();
    expect(result.user).toBeDefined();
  });

  it("使用无效的凭据应当拒绝登录", async () => {
    const result = await loginFn({
      data: { email: "test@example.com", password: "wrongpassword" },
    });

    expect(result.error).toBe("凭据无效");
  });
});
```

### 集成测试

```tsx
// __tests__/auth-flow.test.tsx
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import { RouterProvider, createMemoryHistory } from "@tanstack/react-router";
import { router } from "../router";

describe("身份验证流程", () => {
  it("访问受保护路由时应当重定向到登录页面", async () => {
    const history = createMemoryHistory();
    history.push("/dashboard"); // 受保护路由

    render(<RouterProvider router={router} history={history} />);

    await waitFor(() => {
      expect(screen.getByText("登录")).toBeInTheDocument();
    });
  });
});
```

## 常见模式

### 加载状态 (Loading States)

```tsx
function LoginForm() {
  const [isLoading, setIsLoading] = useState(false);
  const loginMutation = useServerFn(loginFn);

  const handleSubmit = async (data: LoginData) => {
    setIsLoading(true);
    try {
      await loginMutation.mutate(data);
    } catch (error) {
      // 处理错误
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* 表单字段 */}
      <button disabled={isLoading}>{isLoading ? "正在登录..." : "登录"}</button>
    </form>
  );
}
```

### “记住我”功能 (Remember Me)

```tsx
export const loginFn = createServerFn({ method: "POST" })
  .inputValidator(
    (data: { email: string; password: string; rememberMe?: boolean }) => data,
  )
  .handler(async ({ data }) => {
    const user = await authenticateUser(data.email, data.password);
    if (!user) return { error: "凭据无效" };

    const session = await useAppSession();
    await session.update(
      { userId: user.id },
      {
        // 如果勾选了“记住我”，则延长会话有效期
        maxAge: data.rememberMe ? 30 * 24 * 60 * 60 : undefined, // 30 天 vs 仅限会话期间
      },
    );

    return { success: true };
  });
```

## 运行示例

通过学习这些实现来理解不同的身份验证模式：

- **[使用 Prisma 的基础认证](https://github.com/TanStack/router/tree/main/examples/react/start-basic-auth)** - 包含数据库和会话的完整自定义实现
- **[Supabase 集成](https://github.com/TanStack/router/tree/main/examples/react/start-supabase-basic)** - 第三方服务集成示例
- **[客户端 Context 认证](https://github.com/TanStack/router/tree/main/examples/react/authenticated-routes)** - 纯客户端身份验证模式

## 从其他方案迁移

### 从客户端验证迁移

如果你正在从客户端身份验证（仅使用 localStorage 或 Context）迁移：

1. 将身份验证逻辑移至服务器函数 (Server Functions)
2. 使用服务器会话 (Sessions) 替换 localStorage
3. 更新路由保护逻辑，使用 `beforeLoad`
4. 添加适当的安全响应头和 CSRF 保护

### 从其他框架迁移

- **Next.js**：将 API 路由替换为服务器函数，迁移 NextAuth 会话
- **Remix**：将 Loaders/Actions 转换为服务器函数，适配会话模式
- **SvelteKit**：将 Form Actions 移至服务器函数，更新路由保护

## 生产环境考量

在选择身份验证方法时，请考虑以下因素：

### 托管方案 vs 自定义实现对比

**托管解决方案 (Clerk, WorkOS, Better Auth)：**

- 预构建的安全措施和定期更新
- 提供 UI 组件和用户管理功能
- 具备合规性认证和审计日志
- 完善的支持和文档
- 按用户数量或订阅收费

**自定义实现 (DIY)：**

- 对实现方式和数据拥有完全控制权
- 无持续的订阅费用
- 可灵活定制业务逻辑和工作流
- 需自行负责安全更新和监控
- 需要处理各种极端情况和攻击向量

### 安全注意事项

身份验证系统需要处理各种安全层面：

- 密码哈希和防范计时攻击
- 会话管理和防范会话固定攻击
- CSRF 和 XSS 防护
- 速率限制和防范暴力破解
- OAuth 流程安全性
- 合规性要求（GDPR、CCPA 等）

## 下一步

在实现身份验证时，请考虑：

- **安全审计**：根据安全最佳实践审查你的实现
- **性能**：为用户查询和会话验证添加缓存
- **监控**：为身份验证事件添加日志和监控
- **合规性**：如果存储个人数据，确保符合相关法规要求

有关其他身份验证方法，请查看 [身份验证概览](./authentication-overview.md)。如需具体的集成帮助，请探索我们的 [运行示例](https://github.com/TanStack/router/tree/main/examples/react)。
