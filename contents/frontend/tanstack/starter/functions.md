---
short_title: 服务器函数
---

# 服务器函数 (Server Functions)

## 什么是服务器函数？

服务器函数允许你定义仅限服务器运行的逻辑，并可以从应用程序的任何地方调用 —— 包括加载器 (loaders)、组件、钩子 (hooks) 或其他服务器函数。它们在服务器上运行，但可以从客户端代码中无缝调用。

```tsx
import { createServerFn } from "@tanstack/react-start";

export const getServerTime = createServerFn().handler(async () => {
  // 这段代码仅在服务器上运行
  return new Date().toISOString();
});

// 可以从任何地方调用 - 组件、加载器、钩子等
const time = await getServerTime();
```

服务器函数提供了服务器端的能力（访问数据库、环境变量、文件系统），同时在跨越网络边界时保持类型安全。

---

## 基础用法

服务器函数通过 `createServerFn()` 创建，并可以指定 HTTP 方法：

```tsx
import { createServerFn } from "@tanstack/react-start";

// GET 请求 (默认)
export const getData = createServerFn().handler(async () => {
  return { message: "来自服务器的问候！" };
});

// POST 请求
export const saveData = createServerFn({ method: "POST" }).handler(async () => {
  // 仅限服务器的逻辑
  return { success: true };
});
```

---

## 在何处调用服务器函数

你可以从以下位置调用服务器函数：

- **路由加载器 (Route loaders)** —— 非常适合数据获取。
- **组件 (Components)** —— 配合 `useServerFn()` 钩子使用。
- **其他服务器函数** —— 组合服务器逻辑。
- **事件处理器 (Event handlers)** —— 处理表单提交、点击事件等。

```tsx
// 在路由加载器中
export const Route = createFileRoute("/posts")({
  loader: () => getPosts(),
});

// 在组件中
function PostList() {
  const getPosts = useServerFn(getServerPosts);

  const { data } = useQuery({
    queryKey: ["posts"],
    queryFn: () => getPosts(),
  });
}
```

---

## 文件组织结构

对于大型应用程序，考虑将服务器端代码组织到独立的文件中。以下是一种推荐方案：

```text
src/utils/
├── users.functions.ts   # 服务器函数包装器 (createServerFn)
├── users.server.ts      # 仅限服务器的辅助函数 (数据库查询、内部逻辑)
└── schemas.ts           # 共享验证模式 (客户端安全)
```

- **`.functions.ts`** —— 导出 `createServerFn` 包装器，可以安全地在任何地方导入。
- **`.server.ts`** —— 仅限服务器的代码，只在服务器函数处理器内部导入。
- **`.ts`** (无后缀) —— 客户端安全的逻辑（类型、模式、常量）。

### 示例

```tsx
// users.server.ts - 仅限服务器的辅助函数
import { db } from "~/db";

export async function findUserById(id: string) {
  return db.query.users.findFirst({ where: eq(users.id, id) });
}
```

```tsx
// users.functions.ts - 服务器函数
import { createServerFn } from "@tanstack/react-start";
import { findUserById } from "./users.server";

export const getUser = createServerFn({ method: "GET" })
  .inputValidator((data: { id: string }) => data)
  .handler(async ({ data }) => {
    return findUserById(data.id);
  });
```

---

## 静态导入是安全的

服务器函数可以在任何文件中静态导入，包括客户端组件：

```tsx
// ✅ 安全 - 构建过程会处理环境摇树优化 (Environment Shaking)
import { getUser } from "~/utils/users.functions";

function UserProfile({ id }) {
  const { data } = useQuery({
    queryKey: ["user", id],
    queryFn: () => getUser({ data: { id } }),
  });
}
```

构建过程会将客户端 bundle 中的服务器函数实现替换为 RPC 存根 (Stubs)。实际的服务器代码永远不会到达浏览器。

> [!WARNING]
> 避免对服务器函数使用动态导入：
>
> ```tsx
> // ❌ 可能会导致构建工具出现问题
> const { getUser } = await import("~/utils/users.functions");
> ```

## 参数与验证 (Parameters & Validation)

服务器函数接收一个名为 `data` 的单一参数。由于它们需要跨越网络边界执行，因此必须进行验证以确保类型安全和运行时的正确性。

### 基础参数

```tsx
import { createServerFn } from "@tanstack/react-start";

export const greetUser = createServerFn({ method: "GET" })
  // 定义输入验证器
  .inputValidator((data: { name: string }) => data)
  .handler(async ({ data }) => {
    return `你好, ${data.name}!`;
  });

// 调用示例
await greetUser({ data: { name: "John" } });
```

### 使用 Zod 进行验证

为了实现更健壮的验证，建议使用 Zod 等模式库：

```tsx
import { createServerFn } from "@tanstack/react-start";
import { z } from "zod";

const UserSchema = z.object({
  name: z.string().min(1),
  age: z.number().min(0),
});

export const createUser = createServerFn({ method: "POST" })
  .inputValidator(UserSchema)
  .handler(async ({ data }) => {
    // 此时 data 已被完全类型化并验证
    return `已创建用户: ${data.name}, 年龄 ${data.age}`;
  });
```

### 处理表单数据 (Form Data)

处理 `FormData` 类型的表单提交：

```tsx
export const submitForm = createServerFn({ method: "POST" })
  .inputValidator((data) => {
    if (!(data instanceof FormData)) {
      throw new Error("预期为 FormData");
    }

    return {
      name: data.get("name")?.toString() || "",
      email: data.get("email")?.toString() || "",
    };
  })
  .handler(async ({ data }) => {
    // 处理表单数据
    return { success: true };
  });
```

---

## 错误处理与重定向 (Error Handling & Redirects)

服务器函数可以抛出错误、重定向以及“未找到 (not-found)”响应。当从路由生命周期或使用 `useServerFn()` 钩子的组件中调用时，这些响应会被自动处理。

### 基础错误处理

```tsx
import { createServerFn } from "@tanstack/react-start";

export const riskyFunction = createServerFn().handler(async () => {
  if (Math.random() > 0.5) {
    throw new Error("出错了！");
  }
  return { success: true };
});

// 错误会被序列化并传回客户端
try {
  await riskyFunction();
} catch (error) {
  console.log(error.message); // "出错了！"
}
```

### 重定向 (Redirects)

在服务器函数中使用重定向来处理身份验证、导航等逻辑：

```tsx
import { createServerFn } from "@tanstack/react-start";
import { redirect } from "@tanstack/react-router";

export const requireAuth = createServerFn().handler(async () => {
  const user = await getCurrentUser();

  if (!user) {
    // 抛出重定向异常，路由器会自动处理
    throw redirect({ to: "/login" });
  }

  return user;
});
```

### 未找到 (Not Found)

为缺失的资源抛出“未找到”错误：

```tsx
import { createServerFn } from "@tanstack/react-start";
import { notFound } from "@tanstack/react-router";

export const getPost = createServerFn()
  .inputValidator((data: { id: string }) => data)
  .handler(async ({ data }) => {
    const post = await db.findPost(data.id);

    if (!post) {
      // 抛出 404 异常
      throw notFound();
    }

    return post;
  });
```

---

## 进阶主题 (Advanced Topics)

关于服务器函数更高级的模式和特性，请参阅以下专门指南：

### 服务器上下文与请求处理 (Server Context & Request Handling)

访问请求头、Cookie 并自定义响应内容：

```tsx
import { createServerFn } from "@tanstack/react-start";
import {
  getRequest,
  getRequestHeader,
  setResponseHeaders,
  setResponseStatus,
} from "@tanstack/react-start/server";

export const getCachedData = createServerFn({ method: "GET" }).handler(
  async () => {
    // 访问原始请求对象
    const request = getRequest();
    const authHeader = getRequestHeader("Authorization");

    // 设置响应头（例如：用于缓存控制）
    setResponseHeaders(
      new Headers({
        "Cache-Control": "public, max-age=300",
        "CDN-Cache-Control": "max-age=3600, stale-while-revalidate=600",
      }),
    );

    // 可选：设置 HTTP 状态码
    setResponseStatus(200);

    return fetchData();
  },
);
```

**常用工具函数：**

- `getRequest()` —— 获取完整的 `Request` 对象。
- `getRequestHeader(name)` —— 读取特定的请求头。
- `setResponseHeader(name, value)` —— 设置单个响应头。
- `setResponseHeaders(headers)` —— 通过 `Headers` 对象设置多个响应头。
- `setResponseStatus(code)` —— 设置 HTTP 状态码。

### 数据流 (Streaming)

从服务器函数向客户端流式传输类型化数据。

### 原始响应 (Raw Responses)

返回 `Response` 对象、二进制数据或自定义内容类型。

### 渐进式增强 (Progressive Enhancement)

通过利用 `.url` 属性配合 HTML 表单，在不依赖 JavaScript 的情况下使用服务器函数。

### 中间件 (Middleware)

通过中间件组合服务器函数，实现身份验证、日志记录和逻辑共享。参见 [中间件指南](./middleware.md)。

### 静态服务器函数 (Static Server Functions)

在构建时缓存服务器函数结果，用于静态生成。参见 [静态服务器函数](./staticfunctions.md)。

---

## 生产构建中的函数 ID 生成 (Function ID generation)

在底层，服务器函数通过生成的稳定 **函数 ID** 进行寻址。这些 ID 会被嵌入到客户端和 SSR 构建中，服务器在运行时利用它们来定位并导入正确的模块。

- **默认机制**：ID 是基于特定种子的 SHA256 哈希值，以保持 bundle 紧凑并避免泄露文件路径。
- **去重**：如果两个函数生成了相同的 ID，系统会通过添加递增后缀（如 `_1`, `_2`）来自动去重。

### 自定义配置 (实验性)

你可以在 Vite 配置中自定义函数 ID 的生成方式。建议使用确定性的输入（如文件名 + 函数名），以确保构建之间的 ID 保持稳定。

```ts
// vite.config.ts
export default defineConfig({
  plugins: [
    tanstackStart({
      serverFns: {
        generateFunctionId: ({ filename, functionName }) => {
          // 返回自定义 ID 字符串
          return crypto
            .createHash("sha1")
            .update(`${filename}--${functionName}`)
            .digest("hex");
        },
      },
    }),
  ],
});
```

---

> **注意**：服务器函数采用编译过程，从客户端 bundle 中提取服务器代码，同时保持无缝的调用模式。在客户端，这些调用会被转换为向服务器发起的 `fetch` 请求。
