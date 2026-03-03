---
short_title: 选择性 SSR
---

# 选择性服务器端渲染 (Selective SSR)

## 什么是选择性 SSR？

在 TanStack Start 中，匹配初始请求的路由默认在服务器上进行渲染。这意味着 `beforeLoad` 和 `loader` 会在服务器上执行，随后渲染路由组件。生成的 HTML 会发送到客户端，客户端随后将标记“水合 (hydrate)”为完全可交互的应用程序。

然而，在某些情况下，你可能希望禁用某些路由或所有路由的 SSR，例如：

- 当 `beforeLoad` 或 `loader` 需要仅限浏览器的 API（如 `localStorage`）时。
- 当路由组件依赖于仅限浏览器的 API（如 `canvas`）时。

TanStack Start 的选择性 SSR 功能允许你配置：

- 哪些路由应该在服务器上执行 `beforeLoad` 或 `loader`。
- 哪些路由组件应该在服务器上进行渲染。

---

## 这与 SPA 模式有何不同？

TanStack Start 的 [SPA 模式](./spa-mode) 会完全禁用 `beforeLoad` 和 `loader` 的服务端执行，以及路由组件的服务端渲染。而 **选择性 SSR** 允许你以静态或动态的方式，按路由维度配置服务端的处理逻辑。

---

## 配置方法

你可以通过 `ssr` 属性来控制路由在初始服务器请求期间的处理方式。如果未设置此属性，则默认为 `true`。你可以通过 `createStart` 中的 `defaultSsr` 选项更改此默认值：

```tsx
// src/start.ts
import { createStart } from "@tanstack/react-start";

export const startInstance = createStart(() => ({
  // 默认禁用 SSR
  defaultSsr: false,
}));
```

### `ssr: true`

这是默认行为（除非另有配置）。在初始请求时，它将：

- 在服务器上运行 `beforeLoad`，并将结果上下文发送到客户端。
- 在服务器上运行 `loader`，并将加载器数据发送到客户端。
- 在服务器上渲染组件，并将 HTML 标记发送到客户端。

```tsx
// src/routes/posts/$postId.tsx
export const Route = createFileRoute("/posts/$postId")({
  ssr: true,
  beforeLoad: () => {
    console.log("在初始请求期间于服务器执行");
    console.log("在后续导航期间于客户端执行");
  },
  loader: () => {
    console.log("在初始请求期间于服务器执行");
    console.log("在后续导航期间于客户端执行");
  },
  component: () => <div>此组件在服务器上渲染</div>,
});
```

### `ssr: false`

这将禁用服务器端的：

- 路由 `beforeLoad` 和 `loader` 的执行。
- 路由组件的渲染。

```tsx
// src/routes/posts/$postId.tsx
export const Route = createFileRoute("/posts/$postId")({
  ssr: false,
  beforeLoad: () => {
    console.log("在水合期间于客户端执行");
  },
  loader: () => {
    console.log("在水合期间于客户端执行");
  },
  component: () => <div>此组件在客户端渲染</div>,
});
```

### `ssr: 'data-only'`

这是一种混合模式，它将：

- 在服务器上运行 `beforeLoad`，并将结果上下文发送到客户端。
- 在服务器上运行 `loader`，并将加载器数据发送到客户端。
- **禁用**路由组件的服务器端渲染。

```tsx
// src/routes/posts/$postId.tsx
export const Route = createFileRoute("/posts/$postId")({
  ssr: "data-only",
  beforeLoad: () => {
    console.log("在初始请求期间于服务器执行");
    console.log("在后续导航期间于客户端执行");
  },
  loader: () => {
    console.log("在初始请求期间于服务器执行");
    console.log("在后续导航期间于客户端执行");
  },
  component: () => <div>此组件在客户端渲染</div>,
});
```

### 函数形式 (Functional Form)

为了获得更大的灵活性，你可以使用 `ssr` 属性的函数形式，在运行时决定是否对某个路由进行 SSR：

```tsx
// src/routes/docs/$docType/$docId.tsx
export const Route = createFileRoute("/docs/$docType/$docId")({
  validateSearch: z.object({ details: z.boolean().optional() }),
  ssr: ({ params, search }) => {
    // 根据动态参数决定：如果是 sheet 类型，则完全禁用 SSR
    if (params.status === "success" && params.value.docType === "sheet") {
      return false;
    }
    // 根据搜索参数决定：如果开启了详情，则仅加载数据不渲染组件
    if (search.status === "success" && search.value.details) {
      return "data-only";
    }
  },
  beforeLoad: () => {
    console.log("是否在服务器执行取决于 ssr() 的结果");
  },
  loader: () => {
    console.log("是否在服务器执行取决于 ssr() 的结果");
  },
  component: () => <div>此组件将在客户端渲染</div>,
});
```

`ssr` 函数仅在初始请求期间于服务器上运行，并会从客户端 bundle 中剔除。

`search` 和 `params` 在经过验证后，作为**辨别联合类型 (Discriminated Union)** 传入：

```tsx
params:
    | { status: 'success'; value: Expand<ResolveAllParamsFromParent<TParentRoute, TParams>> }
    | { status: 'error'; error: unknown }
search:
    | { status: 'success'; value: Expand<ResolveFullSearchSchema<TParentRoute, TSearchValidator>> }
    | { status: 'error'; error: unknown }
```

如果验证失败，`status` 将为 `'error'`；否则为 `'success'`，此时 `value` 将包含验证后的数据。

---

### 继承机制 (Inheritance)

在运行时，子路由会继承其父路由的选择性 SSR 配置。然而，继承的值**只能向更严格的方向修改**（即：`true` -> `data-only` 或 `false`；`data-only` -> `false`）。

**示例 A：**

```text
root { ssr: undefined } (默认为 true)
  posts { ssr: false }
      $postId { ssr: true }
```

- `root` 默认为 `ssr: true`。
- `posts` 设置为 `ssr: false`，因此其 `beforeLoad/loader` 和组件都不会在服务端运行。
- `$postId` 虽然设置了 `ssr: true`，但它继承了父级的 `ssr: false`。由于不能放宽限制，`ssr: true` 不生效，维持 `ssr: false`。

**示例 B：**

```text
root { ssr: undefined }
  posts { ssr: 'data-only' }
      $postId { ssr: true }
        details { ssr: false }
```

- `posts` 设置为 `data-only`：数据在服务端加载，组件不在服务端渲染。
- `$postId` 继承了 `data-only`。
- `details` 设置为 `ssr: false`：由于这比 `data-only` 更严格，因此配置生效。

---

## 回退渲染 (Fallback Rendering)

对于第一个匹配到 `ssr: false` 或 `ssr: 'data-only'` 的路由，服务器将渲染该路由的 `pendingComponent` 作为回退 UI。如果没有配置，则渲染 `defaultPendingComponent`。如果两者都没配置，则不渲染回退内容。

在客户端水合期间，即使路由没有定义 `beforeLoad` 或 `loader`，此回退 UI 也会至少显示 `minPendingMs`（或全局默认的 `defaultPendingMinMs`）的时间。

---

## 如何禁用根路由的 SSR？

你可以禁用根路由组件的服务端渲染，但 `<html>` 外壳（Shell）仍需在服务端渲染。这个外壳通过 `shellComponent` 属性配置，并接收一个 `children` 属性。`shellComponent` **始终**会进行 SSR，它包裹着根组件、错误组件或“未找到”组件。

一个禁用根组件 SSR 的最小化配置如下：

```tsx
import * as React from "react";
import {
  HeadContent,
  Outlet,
  Scripts,
  createRootRoute,
} from "@tanstack/react-router";

export const Route = createRootRoute({
  shellComponent: RootShell, // 始终在服务端渲染
  component: RootComponent,
  errorComponent: () => <div>出错了</div>,
  notFoundComponent: () => <div>页面未找到</div>,
  ssr: false, // 禁用根路由组件的 SSR
});

function RootShell({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head>
        <HeadContent />
      </head>
      <body>
        {children} {/* 这里是 RootComponent 挂载的地方 */}
        <Scripts />
      </body>
    </html>
  );
}

function RootComponent() {
  return (
    <div>
      <h1>此组件将在客户端渲染</h1>
      <Outlet />
    </div>
  );
}
```
