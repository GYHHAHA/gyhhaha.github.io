---
short_title: 路由概念
---

# 路由概念 (Routing Concepts)

TanStack Router 支持许多强大的路由概念，让您能够轻松构建复杂且动态的路由系统。

这些概念中的每一个都非常实用且强大，我们将在接下来的章节中深入探讨它们。

## 路由剖析 (Anatomy of a Route)

除了根路由 (Root Route)之外，所有其他路由都是使用 `createFileRoute` 函数配置的，它在处理基于文件的路由时提供了类型安全：

```tsx title="src/routes/index.tsx"
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/")({
  component: PostsComponent,
});
```

`createFileRoute` 函数接受一个参数，即字符串形式的文件路由路径。

**❓❓❓ “等等，你要让我手动把路由文件的路径传给 `createFileRoute`？”**

是的！但别担心，这个路径会**由 TanStack Router 捆绑器插件或 Router CLI 为您自动写入和管理**。因此，当您创建新路由、移动路由或重命名路由时，该路径会自动为您更新。

之所以需要这个路径名，完全是为了实现 TanStack Router 那神奇的类型安全。如果没有这个路径名，TypeScript 就无法知道我们当前处于哪个文件中！（我们希望 TypeScript 能内置这种功能，但目前还没有 🤷‍♂️）

## 根路由 (The Root Route)

根路由是整个路由树中最顶层的路由，它像容器一样封装了所有其他作为子路由的路由。

- 它没有路径（path）
- 它**总是**被匹配
- 它的 `component` **总是**被渲染

尽管它没有路径，但根路由拥有与其他路由完全相同的功能，包括：

- 组件 (components)
- 加载器 (loaders)
- 查询参数校验 (search param validation)
- 等等

要创建根路由，请调用 `createRootRoute()` 函数，并将其作为路由文件中的 `Route` 变量导出：

```tsx
// 标准根路由
import { createRootRoute } from "@tanstack/react-router";

export const Route = createRootRoute();

// 带有上下文 (Context) 的根路由
import { createRootRouteWithContext } from "@tanstack/react-router";
import type { QueryClient } from "@tanstack/react-query";

export interface MyRouterContext {
  queryClient: QueryClient;
}
export const Route = createRootRouteWithContext<MyRouterContext>();
```

要了解有关 TanStack Router 中上下文的更多信息，请参阅 [路由上下文 (Router Context)](../guide/router-context.md) 指南。

## 基础路由 (Basic Routes)

基础路由匹配特定的路径，例如 `/about`、`/settings`、`/settings/notifications` 都是基础路由，因为它们与路径完全匹配。

让我们看一个 `/about` 路由：

```tsx title="src/routes/about.tsx"
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/about")({
  component: AboutComponent,
});

function AboutComponent() {
  return <div>About</div>;
}
```

基础路由简单直接。它们完全匹配路径并渲染提供的组件。

## 索引路由 (Index Routes)

索引路由专门用于在其父路由**完全匹配且没有子路由匹配**时进行目标渲染。

让我们看一个针对 `/posts` URL 的索引路由：

```tsx title="src/routes/posts.index.tsx"
import { createFileRoute } from "@tanstack/react-router";

// 注意末尾的斜杠，它用于定位索引路由
export const Route = createFileRoute("/posts/")({
  component: PostsIndexComponent,
});

function PostsIndexComponent() {
  return <div>Please select a post!</div>;
}
```

当 URL 正好是 `/posts` 时，此路由将被匹配。

## 动态路由段 (Dynamic Route Segments)

以 `$` 开头后跟标签的路由路径段是动态的，它们会将 URL 的该部分捕获到 `params` 对象中，供您的应用程序使用。例如，路径名 `/posts/123` 将匹配 `/posts/$postId` 路由，而 `params` 对象将为 `{ postId: '123' }`。

这些参数可以在您的路由配置和组件中使用！让我们看一个 `posts.$postId.tsx` 路由：

```tsx title="src/routes/posts.tsx"
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/posts/$postId")({
  // 在加载器中
  loader: ({ params }) => fetchPost(params.postId),
  // 或者在组件中
  component: PostComponent,
});

function PostComponent() {
  // 在组件中！
  const { postId } = Route.useParams();
  return <div>Post ID: {postId}</div>;
}
```

> 🧠 动态段在路径的**每个**分段中都有效。例如，您可以拥有一个路径为 `/posts/$postId/$revisionId` 的路由，每个 `$` 段都会被捕获到 `params` 对象中。

## 通配符 / 全匹配路由 (Splat / Catch-All Routes)

路径仅为 `$` 的路由被称为“通配符 (splat)”路由，因为它*总是*捕获 URL 路径名中从 `$` 到末尾的*任何*剩余部分。捕获的路径名随后可在 `params` 对象的特殊 `_splat` 属性中找到。

例如，针对 `files/$` 路径的路由是一个通配符路由。如果 URL 路径名是 `/files/documents/hello-world`，则 `params` 对象将在特殊的 `_splat` 属性下包含 `documents/hello-world`：

```js
{
  '_splat': 'documents/hello-world'
}
```

> ⚠️ 在路由器的 v1 版本中，为了向后兼容，通配符路由也可以用 `*` 代替 `_splat` 键。这将在 v2 版本中移除。

> 🧠 为什么要用 `$`？多亏了像 Remix 这样的工具，我们知道尽管 `*` 是表示通配符最常见的字符，但它们在文件名或 CLI 工具中并不友好，所以像他们一样，我们决定改用 `$`。

## 可选路径参数 (Optional Path Parameters)

可选路径参数允许您定义在 URL 中可能存在也可能不存在的路由段。它们使用 `{-$paramName}` 语法，并提供灵活的路由模式，其中某些参数是可选的。

```tsx title="src/routes/posts.{-$category}.tsx"
// `-$category` 段是可选的，因此该路由同时匹配 `/posts` 和 `/posts/tech`
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/posts/{-$category}")({
  component: PostsComponent,
});

function PostsComponent() {
  const { category } = Route.useParams();

  return <div>{category ? `Posts in ${category}` : "All Posts"}</div>;
}
```

此路由将同时匹配 `/posts`（category 为 `undefined`）和 `/posts/tech`（category 为 `"tech"`）。

您还可以在单个路由中定义多个可选参数：

```tsx title="src/routes/posts.{-$category}.${-$slug}.tsx"
// `-$category` 段是可选的，因此该路由同时匹配 `/posts` 和 `/posts/tech`
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/posts/{-$category}/{-$slug}")({
  component: PostsComponent,
});
```

此路由匹配 `/posts`、`/posts/tech` 以及 `/posts/tech/hello-world`。

> 🧠 带有可选参数的路由在优先级排名上低于精确匹配路由，这确保了像 `/posts/featured` 这样更具体的路由会在 `/posts/{-$category}` 之前被匹配。

## 布局路由 (Layout Routes)

布局路由用于通过额外的组件和逻辑来包裹子路由。它们在以下场景中非常有用：

- 使用布局组件包裹子路由
- 在显示任何子路由之前强制执行 `loader` 要求
- 验证并为子路由提供查询参数 (search params)
- 为子路由提供错误组件 (error components) 或挂起元素 (pending elements) 的回退方案
- 向所有子路由提供共享上下文 (shared context)
- 以及更多功能！

让我们看一个名为 `app.tsx` 的布局路由示例：

```
routes/
├── app.tsx
├── app.dashboard.tsx
├── app.settings.tsx
```

在上面的树形结构中，`app.tsx` 是一个布局路由，它包裹了两个子路由：`app.dashboard.tsx` 和 `app.settings.tsx`。

这种树形结构用于通过布局组件包裹子路由：

```tsx title="src/routes/app.tsx"
import { Outlet, createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/app")({
  component: AppLayoutComponent,
});

function AppLayoutComponent() {
  return (
    <div>
      <h1>App Layout</h1>
      <Outlet />
    </div>
  );
}
```

下表显示了根据 URL 将渲染哪些组件：

| URL 路径         | 组件                     |
| ---------------- | ------------------------ |
| `/app`           | `<AppLayout>`            |
| `/app/dashboard` | `<AppLayout><Dashboard>` |
| `/app/settings`  | `<AppLayout><Settings>`  |

由于 TanStack Router 支持混合扁平路由和目录路由，您也可以在目录中使用布局路由来表达应用的路由：

```
routes/
├── app/
│   ├── route.tsx
│   ├── dashboard.tsx
│   ├── settings.tsx
```

在这个嵌套树中，`app/route.tsx` 文件是布局路由的配置，它包裹了两个子路由 `app/dashboard.tsx` 和 `app/settings.tsx`。

布局路由还允许您为动态路由段 (Dynamic Route Segments) 强制执行组件和加载器逻辑：

```
routes/
├── app/users/
│   ├── $userId/
|   |   ├── route.tsx
|   |   ├── index.tsx
|   |   ├── edit.tsx
```

## 无路径布局路由 (Pathless Layout Routes)

与布局路由(layout-routes)类似，无路径布局路由用于通过额外的组件和逻辑包裹子路由。然而，无路径布局路由不需要在 URL 中有匹配的 `path`。

无路径布局路由以前缀下划线 (`_`) 开头，表示它们是“无路径”的。

> 🧠 下划线 `_` 前缀之后的路径部分被用作路由的 ID。这是必需的，因为每个路由都必须是唯一可标识的，特别是在使用 TypeScript 时，这有助于避免类型错误并有效地实现自动补全。

让我们看一个名为 `_pathlessLayout.tsx` 的路由示例：

```
routes/
├── _pathlessLayout.tsx
├── _pathlessLayout.a.tsx
├── _pathlessLayout.b.tsx
```

在上面的树结构中，`_pathlessLayout.tsx` 是一个无路径布局路由，它包裹了两个子路由 `_pathlessLayout.a.tsx` 和 `_pathlessLayout.b.tsx`。

`_pathlessLayout.tsx` 路由用于通过无路径布局组件包裹子路由：

```tsx title="src/routes/_pathlessLayout.tsx"
import { Outlet, createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/_pathlessLayout")({
  component: PathlessLayoutComponent,
});

function PathlessLayoutComponent() {
  return (
    <div>
      <h1>Pathless layout</h1>
      <Outlet />
    </div>
  );
}
```

下表显示了根据 URL 将渲染哪个组件：

| URL 路径 | 组件                  |
| -------- | --------------------- |
| `/`      | `<Index>`             |
| `/a`     | `<PathlessLayout><A>` |
| `/b`     | `<PathlessLayout><B>` |

由于 TanStack Router 支持混合扁平路由和目录路由，您也可以在目录中使用无路径布局路由：

```
routes/
├── _pathlessLayout/
│   ├── route.tsx
│   ├── a.tsx
│   ├── b.tsx
```

然而，与普通布局路由不同，由于无路径布局路由不根据 URL 路径段进行匹配，这意味着这些路由不支持将动态路由段dynamic-route-segments作为其路径的一部分，因此无法在 URL 中匹配。

这意味着您**不能**这样做：

```
routes/
├── _$postId/ ❌
│   ├── ...
```

相反，您必须这样做：

```
routes/
├── $postId/
├── _postPathlessLayout/ ✅
│   ├── ...
```

## 非嵌套路由 (Non-Nested Routes)

非嵌套路由可以通过在父级文件路由段后面添加下划线 `_` 来创建，用于**解除**路由与其父级的嵌套关系，并渲染其自身的组件树。

考虑以下扁平路由树：

```
routes/
├── posts.tsx
├── posts.$postId.tsx
├── posts_.$postId.edit.tsx
```

下表显示了根据 URL 将渲染哪个组件：

| URL 路径          | 组件                         |
| ----------------- | ---------------------------- |
| `/posts`          | `<Posts>`                    |
| `/posts/123`      | `<Posts><Post postId="123">` |
| `/posts/123/edit` | `<PostEditor postId="123">`  |

- `posts.$postId.tsx` 路由按照常规嵌套在 `posts.tsx` 路由下，并将渲染 `<Posts><Post>`。
- `posts_.$postId.edit.tsx` 路由**不共享**与其他路由相同的 `posts` 前缀，因此它将被视为顶级路由，并将渲染 `<PostEditor>`。

## 从路由中排除文件和文件夹 (Excluding Files and Folders from Routes)

通过在文件名上添加横杠 `-` 前缀，可以将文件和文件夹排除在路由生成之外。这让您能够在路由目录中同地协作 (colocate) 逻辑。

考虑以下路由树：

```
routes/
├── posts.tsx
├── -posts-table.tsx // 👈🏼 被忽略
├── -components/ // 👈🏼 被忽略
│   ├── header.tsx // 👈🏼 被忽略
│   ├── footer.tsx // 👈🏼 被忽略
│   ├── ...
```

我们可以将排除的文件导入到我们的 posts 路由中：

```tsx title="src/routes/posts.tsx"
import { createFileRoute } from "@tanstack/react-router";
import { PostsTable } from "./-posts-table";
import { PostsHeader } from "./-components/header";
import { PostsFooter } from "./-components/footer";

export const Route = createFileRoute("/posts")({
  loader: () => fetchPosts(),
  component: PostComponent,
});

function PostComponent() {
  const posts = Route.useLoaderData();

  return (
    <div>
      <PostsHeader />
      <PostsTable posts={posts} />
      <PostsFooter />
    </div>
  );
}
```

被排除的文件不会被添加到 `routeTree.gen.ts` 中。

## 无路径路由组目录 (Pathless Route Group Directories)

无路径路由组目录使用括号 `()` 作为将路由文件组合在一起的一种方式，而不管它们的路径如何。它们纯粹是组织性的，不会以任何方式影响路由树或组件树。

```
routes/
├── index.tsx
├── (app)/
│   ├── dashboard.tsx
│   ├── settings.tsx
│   ├── users.tsx
├── (auth)/
│   ├── login.tsx
│   ├── register.tsx
```

在上面的示例中，`app` 和 `auth` 目录纯粹是组织性的。它们用于将相关的路由组合在一起，以便于导航和组织。

下表显示了根据 URL 将渲染哪个组件：

| URL 路径     | 组件          |
| ------------ | ------------- |
| `/`          | `<Index>`     |
| `/dashboard` | `<Dashboard>` |
| `/settings`  | `<Settings>`  |
| `/users`     | `<Users>`     |
| `/login`     | `<Login>`     |
| `/register`  | `<Register>`  |

如您所见，`app` 和 `auth` 目录不会以任何方式影响路由树或组件树。
