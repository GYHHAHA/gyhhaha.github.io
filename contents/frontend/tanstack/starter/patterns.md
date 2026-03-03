---
short_title: 代码执行模式
---

# 代码执行模式 (Code Execution Patterns)

本指南涵盖了在 TanStack Start 应用程序中控制代码运行位置的模式 —— 仅限服务器、仅限客户端或同构（双端环境）。有关基础概念，请参阅 [执行模型 (Execution Model)](./execution.md) 指南。

## 快速上手

在你的 TanStack Start 应用程序中设置执行边界：

```tsx
import {
  createServerFn,
  createServerOnlyFn,
  createClientOnlyFn,
  createIsomorphicFn,
} from "@tanstack/react-start";

// 服务器函数 (RPC 调用)
const getUsers = createServerFn().handler(async () => {
  return await db.users.findMany();
});

// 仅限服务器的工具函数（在客户端调用会崩溃）
const getSecret = createServerOnlyFn(() => process.env.API_SECRET);

// 仅限客户端的工具函数（在服务器端调用会崩溃）
const saveToStorage = createClientOnlyFn((data: any) => {
  localStorage.setItem("data", JSON.stringify(data));
});

// 每个环境有不同的实现
const logger = createIsomorphicFn()
  .server((msg) => console.log(`[SERVER]: ${msg}`))
  .client((msg) => console.log(`[CLIENT]: ${msg}`));
```

---

## 实现模式

### 渐进式增强 (Progressive Enhancement)

使用渐进式增强模式，可以确保组件在没有 JavaScript 的情况下也能工作，并在有 JS 时获得更好的体验。

```tsx
// 组件在没有 JS 的情况下可以工作，有 JS 时则得到增强
function SearchForm() {
  const [query, setQuery] = useState("");

  return (
    <form action="/search" method="get">
      <input
        name="q"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
      />
      {/* 使用 ClientOnly 为客户端提供增强的搜索按钮 */}
      <ClientOnly fallback={<button type="submit">搜索</button>}>
        <SearchButton onSearch={() => search(query)} />
      </ClientOnly>
    </form>
  );
}
```

### 环境感知型存储 (Environment-Aware Storage)

利用 `createIsomorphicFn` 根据运行环境自动切换存储机制。

```tsx
const storage = createIsomorphicFn()
  .server((key: string) => {
    // 服务器端：基于文件的缓存
    const fs = require("node:fs");
    return JSON.parse(fs.readFileSync(".cache", "utf-8"))[key];
  })
  .client((key: string) => {
    // 客户端：localStorage
    return JSON.parse(localStorage.getItem(key) || "null");
  });
```

## 常见问题

### 环境变量泄露 (Environment Variable Exposure)

```tsx
// ❌ 错误做法：泄露到客户端 bundle 中
const apiKey = process.env.SECRET_KEY;

// ✅ 正确做法：仅限服务器访问
const apiKey = createServerOnlyFn(() => process.env.SECRET_KEY);
```

### 错误的加载器 (Loader) 假设

```tsx
// ❌ 错误做法：假设 loader 仅在服务器运行
export const Route = createFileRoute("/users")({
  loader: () => {
    // 注意：这段逻辑在服务器和客户端都会运行！
    const secret = process.env.SECRET; // 会泄露到客户端
    return fetch(`/api/users?key=${secret}`);
  },
});

// ✅ 正确做法：使用服务器函数处理仅限服务器的操作
const getUsersSecurely = createServerFn().handler(() => {
  const secret = process.env.SECRET; // 仅在服务器端运行
  return fetch(`/api/users?key=${secret}`);
});

export const Route = createFileRoute("/users")({
  // 调用同构的服务器函数
  loader: () => getUsersSecurely(),
});
```

### 水合不匹配 (Hydration Mismatches)

```tsx
// ❌ 错误做法：服务器与客户端内容不一致
function CurrentTime() {
  return <div>{new Date().toLocaleString()}</div>;
}

// ✅ 正确做法：保持一致的渲染逻辑
function CurrentTime() {
  const [time, setTime] = useState<string>();

  useEffect(() => {
    // 仅在客户端水合后更新时间
    setTime(new Date().toLocaleString());
  }, []);

  return <div>{time || "正在加载..."}</div>;
}
```

---

## 生产环境检查清单 (Production Checklist)

- [ ] **Bundle 分析**：验证仅限服务器的代码没有包含在客户端 bundle 中。
- [ ] **环境变量**：确保秘钥类变量使用了 `createServerOnlyFn()` 或 `createServerFn()`。
- [ ] **Loader 逻辑**：时刻记住 loader 是同构的，而不是仅在服务器端运行。
- [ ] **ClientOnly 回退 (Fallbacks)**：提供适当的回退组件以防止布局抖动（Layout Shift）。
- [ ] **错误边界 (Error Boundaries)**：优雅地处理服务器/客户端执行错误。

---

## 相关资源

- [执行模型 (Execution Model)](./execution.md) —— 核心概念与架构模式
- [服务器函数 (Server Functions)](./functions.md) —— 深入了解服务器函数模式
- [环境变量 (Environment Variables)](./env.md) —— 安全的环境变量处理方式
- [中间件 (Middleware)](./middleware.md) —— 服务器函数中间件模式
