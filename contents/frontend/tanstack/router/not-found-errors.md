---
short_title: 未找到错误
---

# 未找到错误 (Not Found Errors)

> ⚠️ 本页涵盖了用于处理“未找到”错误的较新 `notFound` 函数和 `notFoundComponent` API。`NotFoundRoute` 路由已弃用，并将在未来的版本中删除。更多信息请参见 [从 `NotFoundRoute` 迁移](从-notfoundroute-迁移)。

## 概述

TanStack Router 中有两种情况会触发“未找到”错误：

- **不匹配的路由路径**：当路径不匹配任何已知的路由匹配模式，**或者**当它部分匹配某个路由，但带有额外的路径段时。
  - 当路径不匹配任何已知路由模式时，**路由器**会自动抛出“未找到”错误。
  - 如果路由器的 `notFoundMode` 设置为 `fuzzy`，则由具有 `notFoundComponent` 的最近父路由处理错误。如果设置为 `root`，则由根路由处理。
  - 示例：
    - 尝试访问 `/users` 但不存在 `/users` 路由。
    - 尝试访问 `/posts/1/edit` 但路由树仅处理 `/posts/$postId`。
- **缺失的资源**：当无法找到资源时，例如具有给定 ID 的文章或任何不可用/不存在的异步数据。
  - **作为开发者的你**必须在找不到资源时抛出“未找到”错误。这可以在 `beforeLoad` 或 `loader` 函数中使用 `notFound` 工具完成。
  - 错误将由具有 `notFoundComponent` 的最近父路由处理（当在 `loader` 中调用 `notFound` 时），或者由根路由处理。
  - 示例：
    - 尝试访问 `/posts/1` 但 ID 为 1 的文章不存在。
    - 尝试访问 `/docs/path/to/document` 但该文档不存在。

在底层，这两种情况都是使用相同的 `notFound` 函数和 `notFoundComponent` API 实现的。

## `notFoundMode` 选项

当 TanStack Router 遇到一个完全不匹配任何已知路由模式的 **pathname**，**或者**部分匹配但带有额外尾随路径段时，它会自动抛出“未找到”错误。

根据 `notFoundMode` 选项的不同，路由器处理这些自动错误的方式也不同：

- fuzzy 模式（默认）：路由器会智能地寻找最接近的合适匹配路由，并显示其 `notFoundComponent`。
- root 模式：所有“未找到”错误都由根路由的 `notFoundComponent` 处理，而不考虑最近的匹配路由。

### `notFoundMode: 'fuzzy'`

默认情况下，路由器的 `notFoundMode` 设置为 `fuzzy`。这表示如果路径名不匹配任何已知路由，路由器将尝试使用具有子节点（即具有 Outlet）并配置了 `notFoundComponent` 的最接近匹配路由。

> **❓ 为什么这是默认设置？** 模糊匹配可以尽可能多地为用户保留父级布局，从而为他们提供更多上下文，方便他们根据原本预期的位置导航到其他有用的地方。

寻找最近合适路由的标准如下：

- 路由必须有子节点，因此必须有一个 `Outlet` 来渲染 `notFoundComponent`。
- 路由必须配置了 `notFoundComponent`，或者路由器必须配置了 `defaultNotFoundComponent`。

例如，考虑以下路由树：

- `__root__`（配置了 `notFoundComponent`）
  - `posts`（配置了 `notFoundComponent`）
    - `$postId`（配置了 `notFoundComponent`）

如果访问路径 `/posts/1/edit`，将渲染以下组件结构：

- `<Root>`
  - `<Posts>`
    - `<Posts.notFoundComponent>`

渲染 `posts` 路由的 `notFoundComponent` 是因为它具有子节点（Outlet）且是配置了该组件的**最近合适父路由**。

### `notFoundMode: 'root'`

当 `notFoundMode` 设置为 `root` 时，所有“未找到”错误都将由根路由的 `notFoundComponent` 处理，而不是从最近的模糊匹配路由冒泡。

例如，对于上述相同的路由树和路径 `/posts/1/edit`，渲染结构将变为：

- `<Root>`
  - `<Root.notFoundComponent>`

## 配置路由的 `notFoundComponent`

为了处理这两类“未找到”错误，你可以为路由附加一个 `notFoundComponent`。当抛出错误时，将渲染此组件。

例如，为 `/settings` 路由配置 `notFoundComponent` 以处理不存在的设置页面：

```tsx
export const Route = createFileRoute("/settings")({
  component: () => {
    return (
      <div>
        <p>设置页面</p>
        <Outlet />
      </div>
    );
  },
  notFoundComponent: () => {
    return <p>该设置页面不存在！</p>;
  },
});
```

或者为 `/posts/$postId` 路由配置组件以处理不存在的文章：

```tsx
export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ params: { postId } }) => {
    const post = await getPost(postId);
    if (!post) throw notFound();
    return { post };
  },
  component: ({ post }) => {
    return (
      <div>
        <h1>{post.title}</h1>
        <p>{post.body}</p>
      </div>
    );
  },
  notFoundComponent: () => {
    return <p>文章未找到！</p>;
  },
});
```

## 全局默认未找到处理

你可能希望为应用中每个带有子路由的路由提供一个默认的“未找到”组件。

> 为什么只针对有子路由的路由？**叶子节点路由（没有子路由的路由）永远不会渲染 `Outlet`，因此无法处理“未找到”错误。**

要实现这一点，请向 `createRouter` 函数传递 `defaultNotFoundComponent`：

```tsx
const router = createRouter({
  defaultNotFoundComponent: () => {
    return (
      <div>
        <p>页面未找到！</p>
        <Link to="/">返回首页</Link>
      </div>
    );
  },
});
```

## 手动抛出 `notFound` 错误

你可以在 loader 方法和组件中使用 `notFound` 函数手动抛出错误。当你需要发出资源不可用的信号时，这非常有用。

`notFound` 函数的工作方式与 `redirect` 类似。要触发错误，你可以执行 **`throw notFound()`**。

```tsx
export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ params: { postId } }) => {
    // 如果文章不存在则返回 `null`
    const post = await getPost(postId);
    if (!post) {
      throw notFound();
      // 或者，你也可以让 notFound 函数内部抛出：
      // notFound({ throw: true })
    }
    // 这里文章被保证是定义过的，因为如果不存在我们就已经抛出错误了
    return { post };
  },
});
```

上述错误将由同一个路由或配置了 `notFoundComponent`（或全局 `defaultNotFoundComponent`）的最近父路由处理。

如果没有找到任何合适的路由来处理错误，根路由将使用 TanStack Router **极其基础（且故意设计得不太美观）**的默认组件来处理，它仅渲染 `<p>Not Found</p>`。强烈建议至少为根路由附加一个 `notFoundComponent` 或配置全局默认组件。

> ⚠️ 在 `beforeLoad` 方法中抛出 `notFound` 错误将始终触发 `__root` 的 `notFoundComponent`。因为 `beforeLoad` 在路由 loader 之前运行，在抛出错误之前无法保证布局所需的任何数据已成功加载。

## 指定处理“未找到”错误的路由

有时你可能希望在特定的父路由上触发“未找到”，并跳过正常的组件传播。为此，请在 `notFound` 函数的 `route` 选项中传入路由 ID。

```tsx
// _pathlessLayout.tsx
export const Route = createFileRoute("/_pathlessLayout")({
  // 这将被渲染
  notFoundComponent: () => {
    return <p>未找到 (在 _pathlessLayout 中)</p>;
  },
  component: () => {
    return (
      <div>
        <p>这是一个无路径布局路由！</p>
        <Outlet />
      </div>
    );
  },
});

// _pathlessLayout/route-a.tsx
export const Route = createFileRoute("/_pathless/route-a")({
  loader: async () => {
    // 这将强制由布局路由处理“未找到”错误
    throw notFound({ routeId: "/_pathlessLayout" });
    //                     ^^^^^^^^^^^^^^^^ 注册过的路由器会自动补全此处
  },
  // 这将不会被渲染
  notFoundComponent: () => {
    return <p>未找到 (在 route-a 中)</p>;
  },
});
```

### 手动指定根路由

你也可以通过将导出的 `rootRouteId` 变量传递给 `notFound` 函数来直接指向根路由：

```tsx
export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ params: { postId } }) => {
    const post = await getPost(postId);
    if (!post) throw notFound({ routeId: rootRouteId });
    return { post };
  },
});
```

### 在组件中抛出错误

你也可以在组件中抛出“未找到”错误。但是，**建议在 loader 方法中抛出，以便正确定义 loader 数据的类型并防止页面闪烁。**

TanStack Router 暴露了一个 `CatchNotFound` 组件（类似于 `CatchBoundary`），可用于捕获组件中的错误并显示相应的 UI。

### `notFoundComponent` 中的数据加载

`notFoundComponent` 在数据加载方面是一个特例。**取决于你尝试访问的路由以及错误抛出的位置，`SomeRoute.useLoaderData` 可能是未定义的**。不过，`Route.useParams`、`Route.useSearch`、`Route.useRouteContext` 等仍会返回已定义的值。

**如果你需要将不完整的 loader 数据传递给 `notFoundComponent`**，请通过 `notFound` 函数中的 `data` 选项传递，并在 `notFoundComponent` 中进行验证。

```tsx
export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ params: { postId } }) => {
    const post = await getPost(postId);
    if (!post)
      throw notFound({
        // 将一些数据转发给 notFoundComponent
        // data: someIncompleteLoaderData
      });
    return { post };
  },
  // `data: unknown` 会通过调用 `notFound` 时的 data 选项传递给组件
  notFoundComponent: ({ data }) => {
    // ❌ 在这里调用 useLoaderData 是无效的：const { post } = Route.useLoaderData()

    // ✅ 这些是有效的：
    const { postId } = Route.useParams();
    const search = Route.useSearch();
    const context = Route.useRouteContext();

    return <p>ID 为 {postId} 的文章未找到！</p>;
  },
});
```

## 配合 SSR 使用

更多信息请参见 [SSR 指南](./ssr.md)。

## 从 `NotFoundRoute` 迁移

`NotFoundRoute` API 已弃用，推荐使用 `notFoundComponent`。

**当你使用 `NotFoundRoute` 时，`notFound` 函数和 `notFoundComponent` 将失效。**

主要区别在于：

- `NotFoundRoute` 是一个路由，需要其父路由有 `<Outlet>` 才能渲染。`notFoundComponent` 是可以附加到任何路由的组件。
- 使用 `NotFoundRoute` 时无法配合布局。`notFoundComponent` 可以。
- 使用 `notFoundComponent` 时，路径匹配是严格的。这意味着如果你有 `/post/$postId`，访问 `/post/1/2/3` 会抛出错误。而使用 `NotFoundRoute` 时，`/post/1/2/3` 会匹配该路由并在有 Outlet 的情况下渲染它。

迁移步骤如下：

```tsx title='src/router.tsx'
import { createRouter } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen.'
- import { notFoundRoute } from './notFoundRoute'  // [!code --]

export const router = createRouter({
  routeTree,
- notFoundRoute // [!code --]
})

// routes/__root.tsx
import { createRootRoute } from '@tanstack/react-router'

export const Route = createRootRoute({
  // ...
+ notFoundComponent: () => {  // [!code ++]
+   return <p>页面未找到！</p>  // [!code ++]
+ } // [!code ++]
})
```

重要变更总结：

- 在根路由添加 `notFoundComponent` 以进行全局处理。
  - 你也可以在路由树的其他任何路由添加它。
- `notFoundComponent` 不支持在内部渲染 `<Outlet>`。
