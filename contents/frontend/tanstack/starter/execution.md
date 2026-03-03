---
short_title: 执行模型
---

# 执行模型 (Execution Model)

理解代码在哪里运行是构建 TanStack Start 应用程序的基础。本指南将解释 TanStack Start 的执行模型以及如何控制代码的执行位置。

## 核心原则：默认同构 (Isomorphic by Default)

**TanStack Start 中的所有代码默认都是同构的** —— 除非明确限制，否则它会在服务器和客户端 bundle 中同时运行并包含。

```tsx
// ✅ 这段代码在服务器和客户端都会运行
function formatPrice(price: number) {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: "USD",
  }).format(price);
}

// ✅ 路由加载器 (Route loaders) 是同构的
export const Route = createFileRoute("/products")({
  loader: async () => {
    // 这在 SSR 期间在服务器运行，在页面导航期间在客户端运行
    const response = await fetch("/api/products");
    return response.json();
  },
});
```

> **关键理解**：路由 `loader` 是同构的 —— 它们在服务器和客户端都会运行，而不仅仅是在服务器上。

---

## 执行边界 (The Execution Boundary)

TanStack Start 应用程序运行在两个环境中：

### 服务器环境 (Server Environment)

- **Node.js 运行时**：可以访问文件系统、数据库和环境变量。
- **SSR 期间**：初始页面在服务器上进行渲染。
- **API 请求**：服务器函数 (Server functions) 在服务端执行。
- **构建时**：进行静态生成和预渲染。

### 客户端环境 (Client Environment)

- **浏览器运行时**：可以访问 DOM、localStorage 和用户交互。
- **水合 (Hydration) 之后**：初始服务器渲染后，客户端接管应用。
- **页面导航**：在应用内导航时，路由加载器在客户端运行。
- **用户交互**：事件处理器、表单提交等逻辑。

---

## 执行控制 API (Execution Control APIs)

### 仅限服务器执行 (Server-Only Execution)

| API                      | 用例               | 客户端行为           |
| :----------------------- | :----------------- | :------------------- |
| `createServerFn()`       | RPC 调用、数据变更 | 向服务器发起网络请求 |
| `createServerOnlyFn(fn)` | 工具函数           | 抛出错误             |

```tsx
import { createServerFn, createServerOnlyFn } from "@tanstack/react-start";

// RPC：在服务器执行，可从客户端调用
const updateUser = createServerFn({ method: "POST" })
  .inputValidator((data: UserData) => data)
  .handler(async ({ data }) => {
    // 仅在服务器运行，但客户端可以触发调用
    return await db.users.update(data);
  });

// 工具函数：仅限服务器，若在客户端调用会导致崩溃
const getEnvVar = createServerOnlyFn(() => process.env.DATABASE_URL);
```

### 仅限客户端执行 (Client-Only Execution)

| API                      | 用例                  | 服务器端行为            |
| :----------------------- | :-------------------- | :---------------------- |
| `createClientOnlyFn(fn)` | 浏览器工具函数        | 抛出错误                |
| `<ClientOnly>`           | 需要浏览器 API 的组件 | 渲染回退内容 (Fallback) |

```tsx
import { createClientOnlyFn } from "@tanstack/react-start";
import { ClientOnly } from "@tanstack/react-router";

// 工具函数：仅限客户端，若在服务器调用会导致崩溃
const saveToStorage = createClientOnlyFn((key: string, value: any) => {
  localStorage.setItem(key, JSON.stringify(value));
});

// 组件：仅在水合完成后渲染子组件
function Analytics() {
  return (
    <ClientOnly fallback={null}>
      <GoogleAnalyticsScript />
    </ClientOnly>
  );
}
```

#### useHydrated 钩子

如果你需要对依赖水合的状态进行更精细的控制，可以使用 `useHydrated` 钩子。它会返回一个布尔值，指示客户端是否已完成水合：

```tsx
import { useHydrated } from "@tanstack/react-router";

function TimeZoneDisplay() {
  const hydrated = useHydrated();
  const timeZone = hydrated
    ? Intl.DateTimeFormat().resolvedOptions().timeZone
    : "UTC";

  return <div>你的时区: {timeZone}</div>;
}
```

**行为特性：**

- **SSR 期间**：始终返回 `false`
- **客户端首次渲染**：返回 `false`
- **水合完成后**：返回 `true`（并在后续所有渲染中保持为 `true`）

当你需要根据客户端数据（如浏览器时区、语言区域或 localStorage）进行条件渲染，同时又需要为服务器渲染提供合理的退路（Fallback）时，这个钩子非常有用。

---

### 环境特定实现 (Environment-Specific Implementations)

```tsx
import { createIsomorphicFn } from "@tanstack/react-start";

// 为不同环境提供不同的实现逻辑
const getDeviceInfo = createIsomorphicFn()
  .server(() => ({ type: "server", platform: process.platform }))
  .client(() => ({ type: "client", userAgent: navigator.userAgent }));
```

---

## 架构模式 (Architectural Patterns)

### 渐进式增强 (Progressive Enhancement)

构建在没有 JavaScript 的情况下也能工作，并在客户端通过 JS 增强功能的组件：

```tsx
function SearchForm() {
  const [query, setQuery] = useState("");

  return (
    <form action="/search" method="get">
      <input
        name="q"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
      />
      {/* 默认渲染标准提交按钮，水合后替换为增强版的搜索按钮 */}
      <ClientOnly fallback={<button type="submit">搜索</button>}>
        <SearchButton onSearch={() => search(query)} />
      </ClientOnly>
    </form>
  );
}
```

### 环境感知型存储 (Environment-Aware Storage)

```tsx
const storage = createIsomorphicFn()
  .server((key: string) => {
    // 服务器端：基于文件的缓存
    const fs = require("node:fs");
    return JSON.parse(fs.readFileSync(".cache", "utf-8"))[key];
  })
  .client((key: string) => {
    // 客户端：使用 localStorage
    return JSON.parse(localStorage.getItem(key) || "null");
  });
```

### RPC 与直接函数调用 (RPC vs Direct Function Calls)

理解何时使用“服务器函数 (Server Functions)”与“仅限服务器函数 (Server-only functions)”：

```tsx
// createServerFn: RPC 模式 - 在服务器执行，可从客户端调用
const fetchUser = createServerFn().handler(async () => await db.users.find());

// 在客户端组件中使用：
const user = await fetchUser(); // ✅ 发起网络请求

// createServerOnlyFn: 如果从客户端调用会崩溃
const getSecret = createServerOnlyFn(() => process.env.SECRET);

// 在客户端中使用：
const secret = getSecret(); // ❌ 抛出错误
```

---

## 常见反模式 (Common Anti-Patterns)

### 环境变量泄露 (Environment Variable Exposure)

```tsx
// ❌ 错误做法：会将变量泄露到客户端 bundle 中
const apiKey = process.env.SECRET_KEY;

// ✅ 正确做法：仅限服务器端访问
const apiKey = createServerOnlyFn(() => process.env.SECRET_KEY);
```

### 错误的加载器 (Loader) 假设

```tsx
// ❌ 错误做法：假设 loader 仅在服务器运行
export const Route = createFileRoute("/users")({
  loader: () => {
    // 这段逻辑在服务器和客户端都会运行！
    const secret = process.env.SECRET; // 会泄露给客户端
    return fetch(`/api/users?key=${secret}`);
  },
});

// ✅ 正确做法：使用服务器函数处理仅限服务器的操作
const getUsersSecurely = createServerFn().handler(() => {
  const secret = process.env.SECRET; // 仅限服务器
  return fetch(`/api/users?key=${secret}`);
});

export const Route = createFileRoute("/users")({
  // 调用同构的服务器函数
  loader: () => getUsersSecurely(),
});
```

### 水合不匹配 (Hydration Mismatches)

```tsx
// ❌ 错误做法：服务器与客户端内容不同
function CurrentTime() {
  return <div>{new Date().toLocaleString()}</div>;
}

// ✅ 正确做法：保持一致的渲染
function CurrentTime() {
  const [time, setTime] = useState<string>();

  useEffect(() => {
    // 仅在水合后更新时间
    setTime(new Date().toLocaleString());
  }, []);

  return <div>{time || "正在加载..."}</div>;
}
```

---

## 手动 vs API 驱动的环境检测

```tsx
// 手动模式：你自己处理逻辑
function logMessage(msg: string) {
  if (typeof window === "undefined") {
    console.log(`[SERVER]: ${msg}`);
  } else {
    console.log(`[CLIENT]: ${msg}`);
  }
}

// API 模式：框架为你处理逻辑
const logMessage = createIsomorphicFn()
  .server((msg) => console.log(`[SERVER]: ${msg}`))
  .client((msg) => console.log(`[CLIENT]: ${msg}`));
```

---

## 架构决策框架 (Architecture Decision Framework)

**在以下情况选择“仅限服务器 (Server-Only)”：**

- 访问敏感数据（环境变量、密钥）
- 文件系统操作
- 数据库连接
- 外部 API 密钥

**在以下情况选择“仅限客户端 (Client-Only)”：**

- DOM 操作
- 浏览器 API（localStorage、地理定位）
- 用户交互处理
- 数据分析/追踪脚本

**在以下情况选择“同构 (Isomorphic)”：**

- 数据格式化/转换
- 业务逻辑
- 共享工具函数
- 路由加载器 (Loaders)（它们本质上就是同构的）

---

## 安全考虑 (Security Considerations)

### Bundle 分析

务必验证仅限服务器的代码没有包含在客户端 bundle 中：

```bash
# 分析客户端 bundle
npm run build
# 检查 dist/client 目录，确保没有任何仅限服务器的导入
```

### 环境变量策略

- **客户端公开**：对需要客户端访问的变量使用 `VITE_` 前缀。
- **仅限服务器**：通过 `createServerOnlyFn()` 或 `createServerFn()` 访问。
- **绝不泄露**：数据库 URL、内部 API 密钥、密钥。

### 错误边界 (Error Boundaries)

优雅地处理服务器/客户端执行错误：

```tsx
function ErrorBoundary({ children }: { children: React.ReactNode }) {
  return (
    <ErrorBoundaryComponent
      fallback={<div>出错了</div>}
      onError={(error) => {
        if (typeof window === "undefined") {
          console.error("[服务器错误]:", error);
        } else {
          console.error("[客户端错误]:", error);
        }
      }}
    >
      {children}
    </ErrorBoundaryComponent>
  );
}
```

理解 TanStack Start 的执行模型对于构建安全、高性能且易于维护的应用程序至关重要。默认同构的方法提供了灵活性，而执行控制 API 则让你在需要时拥有精确的掌控力。
