---
short_title: 路由
---

# 路由 (Routing)

TanStack Start 构建在 **TanStack Router** 之上，因此 TanStack Router 的所有功能对你来说都是可用的。

> [!NOTE]
> 我们强烈建议阅读 [TanStack Router 官方文档](https://tanstack.com/router/latest/docs/framework/react/overview) 以深入了解其功能。本节内容更多是对 TanStack Router 及其在 Start 中工作方式的高层级概述。

## 路由器配置 (The Router)

`src/router.tsx` 文件决定了 TanStack Router 在 Start 应用程序中的行为。该文件位于项目的 `src` 目录下。

```text
src/
├── router.tsx
```

在这里，你可以配置从默认的 [预加载功能](https://tanstack.com/router/latest/docs/framework/react/guide/preloading) 到 [缓存过期逻辑](https://tanstack.com/router/latest/docs/framework/react/guide/data-loading) 的所有内容。

```tsx
// src/router.tsx
import { createRouter } from "@tanstack/react-router";
import { routeTree } from "./routeTree.gen";

// 你必须导出一个 getRouter 函数，
// 每次调用该函数都会返回一个新的路由器实例
export function getRouter() {
  const router = createRouter({
    routeTree,
    scrollRestoration: true, // 启用滚动恢复
  });

  return router;
}
```

## 基于文件的路由 (File-Based Routing)

Start 采用了 TanStack Router 的文件路由方案，以确保获得正确的代码拆分（Code-splitting）和高级的类型安全。

你可以在 `src/routes` 目录中管理你的路由。

```text
src/
├── routes <-- 这是存放路由文件的地方
│   ├── __root.tsx
│   ├── index.tsx
│   ├── about.tsx
│   ├── posts.tsx
│   ├── posts/$postId.tsx
```

## 根路由 (The Root Route)

根路由（Root Route）是整个路由树的最顶层路由，它像壳一样包裹着所有其他子路由。它必须命名为 `__root.tsx` 并存放在 `src/routes/__root.tsx`。

```text
src/
├── routes
│   ├── __root.tsx <-- 根路由
```

**根路由的特性：**

- 它没有路径前缀，且**始终**会被匹配。
- 它的 `component` 属性定义的组件**始终**会被渲染。
- 这是你渲染文档外壳的地方，例如 `<html>`、`<body>` 等标签。
- 正因为它**始终渲染**，所以它是构建应用程序外壳和处理全局逻辑的绝佳场所。

```tsx
// src/routes/__root.tsx
import {
  Outlet,
  createRootRoute,
  HeadContent,
  Scripts,
} from "@tanstack/react-router";
import type { ReactNode } from "react";

export const Route = createRootRoute({
  head: () => ({
    meta: [
      {
        charSet: "utf-8",
      },
      {
        name: "viewport",
        content: "width=device-width, initial-scale=1",
      },
      {
        title: "TanStack Start 入门项目",
      },
    ],
  }),
  component: RootComponent,
});

function RootComponent() {
  return (
    <RootDocument>
      {/* Outlet 用于渲染匹配的子路由内容 */}
      <Outlet />
    </RootDocument>
  );
}

function RootDocument({ children }: Readonly<{ children: ReactNode }>) {
  return (
    <html>
      <head>
        <HeadContent />
      </head>
      <body>
        {children}
        {/* Scripts 组件负责加载客户端 JS，必须包含以确保功能正常 */}
        <Scripts />
      </body>
    </html>
  );
}
```

请注意 `<body>` 标签底部的 `Scripts` 组件。它用于加载应用程序所有的客户端 JavaScript，为了保证功能正常，应当始终包含该组件。

## HeadContent 组件

`HeadContent` 组件用于渲染文档的 `head`、`title`、`meta`、`link` 以及与头部相关的 `script` 标签。

它应当被**渲染在根路由布局的 `<head>` 标签中**。

## Outlet 组件

`Outlet` 组件用于渲染下一个潜在匹配的子路由内容。`<Outlet />` 不接收任何 Props，并且可以渲染在路由组件树中的任何位置。如果没有匹配的子路由，`<Outlet />` 将渲染为 `null`。

## Scripts 组件

`Scripts` 组件用于渲染文档的正文脚本（Body Scripts）。

它应当被**渲染在根路由布局的 `<body>` 标签中**。

---

## 路由树生成 (Route Tree Generation)

你可能会注意到项目中有一个 `routeTree.gen.ts` 文件。

```text
src/
├── routeTree.gen.ts <-- 自动生成的路由树文件
```

当你运行 TanStack Start（通过 `npm run dev` 或 `npm run start`）时，该文件会自动生成。此文件包含了生成的路由树以及一系列 TypeScript 工具函数，它们使 TanStack Start 的类型安全变得极其高效且支持完全推导。

---

## 嵌套路由 (Nested Routing)

TanStack Router 使用嵌套路由将 URL 与要渲染的正确定组件树进行匹配。

例如，给定以下路由文件：

```text
routes/
├── __root.tsx         // 渲染 <Root> 组件
├── posts.tsx          // 渲染 <Posts> 组件
├── posts.$postId.tsx  // 渲染 <Post> 组件
```

当访问 URL `/posts/123` 时，生成的组件树如下所示：

```tsx
<Root>
  <Posts>
    <Post />
  </Posts>
</Root>
```

---

## 路由类型 (Types of Routes)

你可以在项目中创建几种不同类型的路由：

- **索引路由 (Index Routes)**：当 URL 与路由路径完全一致时匹配。
- **动态/通配符/Splat 路由**：动态捕获 URL 路径的部分或全部内容到变量中，供应用程序使用。

此外，还有一些工具类路由用于组织和管理你的路由逻辑：

- **无路径布局路由 (Pathless Layout Routes)**：为一组路由应用布局或逻辑，而不增加额外的路径嵌套。
- **非嵌套路由 (Non-Nested Routes)**：使路由脱离父级嵌套，渲染其独立的组件树。
- **分组路由 (Grouped Routes)**：仅为了组织目录而将路由分在一个文件夹中，而不影响路径层级。

---

## 路由定义示例

要创建路由，只需创建一个与你想要创建的路径相对应的文件即可：

| 路径             | 文件名              | 类型       |
| :--------------- | :------------------ | :--------- |
| `/`              | `index.tsx`         | 索引路由   |
| `/about`         | `about.tsx`         | 静态路由   |
|                  | `posts.tsx`         | “布局”路由 |
| `/posts/`        | `posts/index.tsx`   | 索引路由   |
| `/posts/:postId` | `posts/$postId.tsx` | 动态路由   |
| `/rest/*`        | `rest/$.tsx`        | 通配符路由 |

### 如何定义路由

使用 `createFileRoute` 函数并将路由作为 `Route` 变量导出。

例如，要处理 `/posts/:postId` 路由，你需要在此位置创建一个名为 `posts/$postId.tsx` 的文件：

```tsx
// src/routes/posts/$postId.tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/posts/$postId")({
  component: PostComponent,
});
```

> [!NOTE]
> 传递给 `createFileRoute` 的路径字符串是**由 TanStack Router Bundler 插件或 Router CLI 自动编写和管理的**。因此，当你创建新路由、移动或重命名路由时，路径会自动更新。

---

## 这仅仅是“开始”

以上只是对如何使用 TanStack Router 配置路由的高层概述。欲了解更详细的信息，请参考 [TanStack Router 官方文档](https://tanstack.com/router/latest/docs/framework/react/routing/file-based-routing)。
