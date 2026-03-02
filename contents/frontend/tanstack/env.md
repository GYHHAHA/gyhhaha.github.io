---
short_title: 环境变量
---

# 环境变量 (Environment Variables)

了解如何在 TanStack Start 应用程序的不同上下文（服务器函数、客户端代码和构建过程）中安全地配置和使用环境变量。

## 快速上手

TanStack Start 会自动加载 `.env` 文件，并在确保安全边界的前提下，使变量在服务器和客户端上下文中均可使用。

```bash
# .env
# 仅限服务端访问
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb

# 允许客户端访问（必须带有 VITE_ 前缀）
VITE_APP_NAME=我的 TanStack Start 应用
```

```typescript
// 服务器函数 - 可以访问任何环境变量
const getUser = createServerFn().handler(async () => {
  const db = await connect(process.env.DATABASE_URL) // ✅ 仅限服务端
  return db.user.findFirst()
})

// 客户端组件 - 仅能访问带有 VITE_ 前缀的变量
export function AppHeader() {
  return <h1>{import.meta.env.VITE_APP_NAME}</h1> // ✅ 客户端安全
}
```

---

## 环境变量上下文

### 服务端上下文 (服务器函数与 API 路由)

服务器函数可以通过 `process.env` 访问 **所有** 环境变量：

```typescript
import { createServerFn } from "@tanstack/react-start";

// 数据库连接（仅限服务端）
const connectToDatabase = createServerFn().handler(async () => {
  const connectionString = process.env.DATABASE_URL; // 无需前缀
  const apiKey = process.env.EXTERNAL_API_SECRET; // 仅保留在服务端

  // 这些变量绝不会暴露给客户端
  return await database.connect(connectionString);
});

// 身份验证（仅限服务端）
const authenticateUser = createServerFn()
  .inputValidator(z.object({ token: z.string() }))
  .handler(async ({ data }) => {
    const jwtSecret = process.env.JWT_SECRET; // 仅限服务端
    return jwt.verify(data.token, jwtSecret);
  });
```

### 客户端上下文 (组件与客户端代码)

客户端代码仅能访问带有 `VITE_` 前缀的变量。这是为了防止敏感信息意外泄露到浏览器 Bundle 中：

```typescript
// 客户端配置
export function ApiProvider({ children }: { children: React.ReactNode }) {
  const apiUrl = import.meta.env.VITE_API_URL     // ✅ 公开
  const apiKey = import.meta.env.VITE_PUBLIC_KEY  // ✅ 公开

  // 以下代码将返回 undefined（安全特性）：
  // const secret = import.meta.env.DATABASE_URL   // ❌ Undefined

  return (
    <ApiContext.Provider value={{ apiUrl, apiKey }}>
      {children}
    </ApiContext.Provider>
  )
}

// 特性开关 (Feature Flags)
export function FeatureGatedComponent() {
  const enableNewFeature = import.meta.env.VITE_ENABLE_NEW_FEATURE === 'true'

  if (!enableNewFeature) return null

  return <NewFeature />
}
```

---

## 环境文件设置

### 文件层级 (按顺序加载)

TanStack Start 会自动按以下顺序加载环境文件：

```text
.env.local          # 本地覆盖（应添加至 .gitignore）
.env.production     # 生产环境专用变量
.env.development    # 开发环境专用变量
.env                # 默认变量（通常提交至 Git）
```

### 示例设置

**.env** (提交至代码仓库)：

```bash
# 公共配置
VITE_APP_NAME=我的 TanStack Start 应用
VITE_API_URL=[https://api.example.com](https://api.example.com)
VITE_SENTRY_DSN=https://...

# 服务端配置模板
DATABASE_URL=postgresql://localhost:5432/myapp_dev
REDIS_URL=redis://localhost:6379
```

**.env.local** (务必添加至 .gitignore)：

```bash
# 本地开发环境的覆盖配置
DATABASE_URL=postgresql://user:password@localhost:5432/myapp_local
STRIPE_SECRET_KEY=sk_test_...
JWT_SECRET=your-local-secret
```

**.env.production**：

```bash
# 生产环境覆盖
VITE_API_URL=[https://api.myapp.com](https://api.myapp.com)
DATABASE_POOL_SIZE=20
```

## 常用模式 (Common Patterns)

### 数据库配置

```typescript
// src/lib/database.ts
import { createServerFn } from "@tanstack/react-start";

const getDatabaseConnection = createServerFn().handler(async () => {
  const config = {
    url: process.env.DATABASE_URL,
    // 解析环境变量并提供默认值
    maxConnections: parseInt(process.env.DB_MAX_CONNECTIONS || "10"),
    ssl: process.env.NODE_ENV === "production",
  };

  return createConnection(config);
});
```

### 身份验证提供者设置 (Authentication)

```typescript
// src/lib/auth.ts (服务端)
export const authConfig = {
  secret: process.env.AUTH_SECRET,
  providers: {
    auth0: {
      domain: process.env.AUTH0_DOMAIN,
      clientId: process.env.AUTH0_CLIENT_ID,
      clientSecret: process.env.AUTH0_CLIENT_SECRET, // 仅限服务端
    }
  }
}

// src/components/AuthProvider.tsx (客户端)
export function AuthProvider({ children }: { children: React.ReactNode }) {
  return (
    <Auth0Provider
      domain={import.meta.env.VITE_AUTH0_DOMAIN}
      clientId={import.meta.env.VITE_AUTH0_CLIENT_ID}
      // 这里不包含客户端密钥 - 它仅安全地保存在服务器端
    >
      {children}
    </Auth0Provider>
  )
}
```

### 外部 API 集成

```typescript
// src/lib/external-api.ts
import { createServerFn } from "@tanstack/react-start";

// 服务端 API 调用（可以使用私有密钥）
const fetchUserData = createServerFn()
  .inputValidator(z.object({ userId: z.string() }))
  .handler(async ({ data }) => {
    const response = await fetch(
      `${process.env.EXTERNAL_API_URL}/users/${data.userId}`,
      {
        headers: {
          // 使用只有服务端可见的 Bearer Token
          Authorization: `Bearer ${process.env.EXTERNAL_API_SECRET}`,
          "Content-Type": "application/json",
        },
      },
    );

    return response.json();
  });

// 客户端 API 调用（仅限公开端点）
export function usePublicData() {
  const apiUrl = import.meta.env.VITE_PUBLIC_API_URL;

  return useQuery({
    queryKey: ["public-data"],
    queryFn: () => fetch(`${apiUrl}/public/stats`).then((r) => r.json()),
  });
}
```

### 特性开关与配置

```typescript
// src/config/features.ts
export const featureFlags = {
  enableNewDashboard: import.meta.env.VITE_ENABLE_NEW_DASHBOARD === 'true',
  enableAnalytics: import.meta.env.VITE_ENABLE_ANALYTICS === 'true',
  debugMode: import.meta.env.VITE_DEBUG_MODE === 'true',
}

// 在组件中使用
export function Dashboard() {
  if (featureFlags.enableNewDashboard) {
    return <NewDashboard />
  }

  return <LegacyDashboard />
}
```

---

## 类型安全 (Type Safety)

### TypeScript 类型声明

创建 `src/env.d.ts` 以增加类型提示：

```typescript
/// <reference types="vite/client" />

interface ImportMetaEnv {
  // 客户端环境变量类型定义
  readonly VITE_APP_NAME: string;
  readonly VITE_API_URL: string;
  readonly VITE_AUTH0_DOMAIN: string;
  readonly VITE_AUTH0_CLIENT_ID: string;
  readonly VITE_SENTRY_DSN?: string;
  readonly VITE_ENABLE_NEW_DASHBOARD?: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}

// 服务端环境变量类型定义
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      readonly DATABASE_URL: string;
      readonly REDIS_URL: string;
      readonly JWT_SECRET: string;
      readonly AUTH0_CLIENT_SECRET: string;
      readonly STRIPE_SECRET_KEY: string;
      readonly NODE_ENV: "development" | "production" | "test";
    }
  }
}

export {};
```

### 运行时验证 (Runtime Validation)

使用 Zod 对环境变量进行运行时验证，确保应用启动时配置是正确的：

```typescript
// src/config/env.ts
import { z } from "zod";

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  NODE_ENV: z.enum(["development", "production", "test"]),
});

const clientEnvSchema = z.object({
  VITE_APP_NAME: z.string(),
  VITE_API_URL: z.string().url(),
  VITE_AUTH0_DOMAIN: z.string(),
  VITE_AUTH0_CLIENT_ID: z.string(),
});

// 验证服务端环境（如果失败将抛出错误）
export const serverEnv = envSchema.parse(process.env);

// 验证客户端环境
export const clientEnv = clientEnvSchema.parse(import.meta.env);
```

## 安全最佳实践 (Security Best Practices)

### 1. 绝不向客户端泄露密钥

```typescript
// ❌ 错误做法 - 密钥泄露到客户端 Bundle 中
const config = {
  apiKey: import.meta.env.VITE_SECRET_API_KEY, // 这将出现在你的 JS 源码中！
};

// ✅ 正确做法 - 将密钥留在服务端
const getApiData = createServerFn().handler(async () => {
  const response = await fetch(apiUrl, {
    // 使用 process.env 访问不带 VITE_ 前缀的私有变量
    headers: { Authorization: `Bearer ${process.env.SECRET_API_KEY}` },
  });
  return response.json();
});
```

### 2. 使用正确的前缀

```bash
# ✅ 仅限服务端（无前缀）
DATABASE_URL=postgresql://...
JWT_SECRET=super-secret-key
STRIPE_SECRET_KEY=sk_live_...

# ✅ 客户端安全（带 VITE_ 前缀）
VITE_APP_NAME=我的应用
VITE_API_URL=[https://api.example.com](https://api.example.com)
VITE_SENTRY_DSN=https://...
```

### 3. 验证必需的变量

```typescript
// src/config/validation.ts
const requiredServerEnv = ["DATABASE_URL", "JWT_SECRET"] as const;
const requiredClientEnv = ["VITE_APP_NAME", "VITE_API_URL"] as const;

// 在服务器启动时验证
for (const key of requiredServerEnv) {
  if (!process.env[key]) {
    throw new Error(`缺少必需的服务端环境变量: ${key}`);
  }
}

// 在构建时验证客户端环境
for (const key of requiredClientEnv) {
  if (!import.meta.env[key]) {
    throw new Error(`缺少必需的客户端环境变量: ${key}`);
  }
}
```

---

## 生产环境检查清单 (Production Checklist)

- [ ] 所有敏感变量均为服务端专用（不带 `VITE_` 前缀）。
- [ ] 客户端变量均使用 `VITE_` 前缀。
- [ ] `.env.local` 已添加至 `.gitignore`。
- [ ] 生产环境变量已在托管平台（如 Vercel, Cloudflare）上配置。
- [ ] 必需的环境变量在启动时进行验证。
- [ ] 源代码中没有硬编码的密钥。
- [ ] 生产环境的数据库 URL 使用了连接池（Connection Pooling）。
- [ ] API 密钥定期轮换。

---

## 常见问题排查

### 环境变量为 Undefined

**问题**：`import.meta.env.MY_VARIABLE` 返回 `undefined`。

**解决方案**：

1. **检查前缀**：确保使用了 `VITE_` 前缀。
2. **重启开发服务器**：修改 `.env` 文件后必须重启。
3. **检查文件位置**：`.env` 文件必须位于项目根目录。
4. **验证构建配置**：确保变量在构建时被正确注入。

### 生产环境中的运行时客户端变量

**问题**：由于 `VITE_` 变量在构建阶段就被替换成了静态值，如何在运行时动态传递变量给客户端？

**解决方案**：通过服务器函数将变量从服务端传递下去：

```tsx
const getRuntimeVar = createServerFn({ method: "GET" }).handler(() => {
  // 注意这里使用的是 process.env，且变量不需要 VITE_ 前缀
  return process.env.MY_RUNTIME_VAR;
});

export const Route = createFileRoute("/")({
  loader: async () => {
    const foo = await getRuntimeVar();
    return { foo };
  },
  component: RouteComponent,
});

function RouteComponent() {
  const { foo } = Route.useLoaderData();
  return <div>运行时变量值: {foo}</div>;
}
```

---

## 服务器构建配置 (Server Build Configuration)

### 静态 `NODE_ENV` 替换

默认情况下，TanStack Start 会在 **服务器构建** 阶段静态替换 `process.env.NODE_ENV`。这可以在生产环境的服务器 Bundle 中实现“死代码消除”（Tree-shaking）。

**为什么这很重要：**
如果没有静态替换，像下面这样的代码会残留在你的生产环境 Bundle 中：

```typescript
if (process.env.NODE_ENV === "development") {
  // 如果不进行静态替换，这段代码不会被剔除
  enableDevTools();
}
```

开启静态替换后，构建器会将表达式识别为 `"production" === 'development'`，从而在构建结果中完全删除该代码块。

### 配置静态替换

你可以通过 `server.build.staticNodeEnv` 选项进行控制：

```ts
// vite.config.ts
export default defineConfig({
  plugins: [
    tanstackStart({
      server: {
        build: {
          // 在构建时替换 process.env.NODE_ENV (默认值: true)
          staticNodeEnv: true,
        },
      },
    }),
  ],
});
```

### 何时禁用静态替换？

如果你需要同一个构建产物部署到多个环境（如 Staging 和 Production），且需要在运行时动态读取环境状态，请将其设置为 `false`。

> **重要提示**：如果你禁用了 `staticNodeEnv`，在生产环境运行服务器时 **必须** 手动设置 `NODE_ENV=production`。否则 React 等库将运行在开发模式，这会严重降低性能并产生额外的警告。

---

## 相关资源

- [执行模式 (Execution Patterns)](./execution.md) —— 了解服务端与客户端代码执行差异。
- [服务器函数 (Server Functions)](./server-functions.md) —— 深入了解服务端代码。
- [托管部署 (Hosting)](./hosting.md) —— 平台特定的环境变量配置。
- [Vite 环境变量文档](https://vitejs.dev/guide/env-and-mode.html) —— Vite 官方说明。
