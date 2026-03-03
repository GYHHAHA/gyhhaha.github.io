---
short_title: 代码分割
---

# 代码分割 (Code Splitting)

代码分割和懒加载是提升应用包体积（Bundle Size）和加载性能的强大技术。

- **减少初始加载量**：减少页面首次加载时需要下载的代码量。
- **按需加载**：代码仅在需要时才会进行加载。
- **更小的分块 (Chunks)**：产生更多体积更小的代码块，更有利于浏览器缓存。

---

## TanStack Router 如何拆分代码？

TanStack Router 将代码分为两类：

### 1. 关键路由配置 (Critical Route Configuration)

这是渲染当前路由并**尽可能早地**启动数据加载过程所必需的代码。它通常包含在主包中。

- 路径解析/序列化
- 查询参数 (Search Param) 校验
- Loaders、Before Load 钩子
- 路由上下文 (Route Context)
- 静态数据
- 链接 (Links)、脚本 (Scripts) 和样式 (Styles)
- 除非在下方列出，否则所有其他路由配置均属于此类

### 2. 非关键/懒加载路由配置 (Non-Critical/Lazy Route Configuration)

这些代码对于匹配路由不是必需的，可以按需加载。

- 路由组件 (Route Component)
- 错误组件 (Error Component)
- 等待中组件 (Pending Component)
- 未找到组件 (Not-found Component)

> 🧠 **为什么 Loader 不被自动拆分？**
>
> - **避免双重延迟**：Loader 本身就是一个异步边界。如果拆分它，你不仅要等待代码块（Chunk）下载，还要等待 Loader 执行，这会造成“双重等待”。
> - **体积占比较小**：相比于组件，Loader 的逻辑代码通常不会占用巨大的包体积。
> - **预加载优先级高**：Loader 是路由中最重要的可预加载资产。如果你启用了预加载（例如鼠标悬停在链接上时），让 Loader 保持立即可用且没有额外异步开销是非常关键的。
>
> 如果你了解这些弊端后仍想拆分 Loader，请参考 [数据加载器拆分](data-loader-splitting) 章节。

---

## 将路由文件封装到目录中

TanStack Router 的基于文件的路由系统支持扁平化和嵌套结构。你可以将路由文件封装到单个目录中，而无需额外配置。

只需将路由文件移动到一个与原文件名同名的目录中，并重命名为 `route.tsx`。

**之前：**

- `posts.tsx`

**之后：**

- `posts/`
  - `route.tsx`

---

## 代码分割的几种方式

TanStack Router 支持多种代码分割方式。如果你使用的是基于代码的路由，请跳至 [基于代码的分割](code-based-splitting) 部分。

使用**基于文件**的路由时，你可以选择以下方式：

- [使用自动代码分割 ✨](using-automatic-code-splitting)
- [使用 `.lazy.tsx` 后缀](using-the-lazytsx-suffix)
- [使用虚拟路由](using-virtual-routes)

---

## 使用自动代码分割 ✨

这是实现路由代码分割最简单且最强大的方式。

当开启 `autoCodeSplitting` 功能时，TanStack Router 会根据上述的“非关键配置”规则自动帮你完成拆分。

> [!IMPORTANT]
> 自动代码分割功能**仅在**你使用基于文件的路由，并配合 [受支持的捆绑器 (Bundler)](../routing/file-based-routing.md#getting-started-with-file-based-routing) 时可用。
> 如果你**仅**使用 CLI (`@tanstack/router-cli`)，该功能将无法工作。

要在 Vite 插件中启用它：

```ts
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { tanstackRouter } from "@tanstack/router-plugin/vite";

export default defineConfig({
  plugins: [
    tanstackRouter({
      // ...
      autoCodeSplitting: true,
    }),
    react(), // 确保在 TanStack Router 插件之后添加 React 插件
  ],
});
```

搞定！TanStack Router 现在会自动根据关键/非关键配置拆分你的所有路由文件。

---

## 使用 `.lazy.tsx` 后缀

如果你无法使用自动拆分功能，依然可以通过 `.lazy.tsx` 后缀手动拆分。**这非常简单：只需将组件代码移动到带有 `.lazy.tsx` 后缀的文件中**，并使用 `createLazyFileRoute` 替代 `createFileRoute`。

> [!IMPORTANT]
> `__root.tsx` 路由文件（使用 `createRootRoute` 或 `createRootRouteWithContext`）**不支持**代码分割，因为它在任何路由下都会被渲染。

`createLazyFileRoute` 仅支持以下选项：

| 导出名称            | 描述                                 |
| :------------------ | :----------------------------------- |
| `component`         | 路由渲染的主组件。                   |
| `errorComponent`    | 路由加载出错时渲染的组件。           |
| `pendingComponent`  | 路由加载过程中（等待时）渲染的组件。 |
| `notFoundComponent` | 抛出“未找到”错误时渲染的组件。       |

### `.lazy.tsx` 拆分示例

**拆分前（单文件）：**

```tsx title="src/routes/posts.tsx"
import { createFileRoute } from "@tanstack/react-router";
import { fetchPosts } from "./api";

export const Route = createFileRoute("/posts")({
  loader: fetchPosts,
  component: Posts,
});

function Posts() {
  // ... 组件逻辑
}
```

**拆分后（两个文件）：**

主文件（包含关键路由配置）：

```tsx title="src/routes/posts.tsx"
import { createFileRoute } from "@tanstack/react-router";
import { fetchPosts } from "./api";

export const Route = createFileRoute("/posts")({
  loader: fetchPosts,
});
```

懒加载文件（包含非关键配置，使用 `.lazy.tsx` 后缀）：

```tsx title="src/routes/posts.lazy.tsx"
import { createLazyFileRoute } from "@tanstack/react-router";

export const Route = createLazyFileRoute("/posts")({
  component: Posts,
});

function Posts() {
  // ... 组件逻辑
}
```

## 使用虚拟路由 (Using Virtual Routes)

你可能会遇到这样一种情况：当你把路由文件中的所有内容都拆分出去后，原文件变空了！在这种情况下，只需**彻底删除该路由文件**即可！系统会自动为你生成一个“虚拟路由”，作为你代码分割文件的锚点。这个虚拟路由将直接存在于生成的路由树文件中。

**拆分前（包含虚拟路由锚点）**

```tsx title="src/routes/posts.tsx"
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/posts")({
  // 还有人在吗？
});
```

```tsx title="src/routes/posts.lazy.tsx"
import { createLazyFileRoute } from "@tanstack/react-router";

export const Route = createLazyFileRoute("/posts")({
  component: Posts,
});

function Posts() {
  // ...
}
```

**拆分后（纯虚拟路由模式）**

[Image of TanStack Router virtual route generation flow]

```tsx title="src/routes/posts.lazy.tsx"
import { createLazyFileRoute } from "@tanstack/react-router";

export const Route = createLazyFileRoute("/posts")({
  component: Posts,
});

function Posts() {
  // ...
}
```

大功告成！🎉

---

## 基于代码的拆分 (Code-Based Splitting)

如果你使用的是基于代码的路由，你依然可以使用 `Route.lazy()` 方法和 `createLazyRoute` 函数来进行代码分割。你需要将路由配置分为两部分：

1. 使用 `createLazyRoute` 函数创建一个懒加载路由。

```tsx title="src/posts.lazy.tsx"
export const Route = createLazyRoute("/posts")({
  component: MyComponent,
});

function MyComponent() {
  return <div>我的组件</div>;
}
```

2. 然后，在 `app.tsx` 的路由定义中调用 `.lazy()` 方法，以此导入包含非关键配置的拆分代码。

```tsx title="src/app.tsx"
const postsRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: "/posts",
}).lazy(() => import("./posts.lazy").then((d) => d.Route));
```

---

## 数据加载器 (Loader) 的拆分

**警告！！！** 拆分路由 Loader 是一场危险的游戏。

虽然它是减小包体积的有力工具，但也伴随着代价，正如在 [TanStack Router 如何拆分代码？](how-does-tanstack-router-split-code) 章节中所提到的。

[Image of JavaScript code splitting loader context diagram]

你可以使用路由的 `loader` 选项来分割数据加载逻辑。虽然这个过程会使得保持传递给 Loader 参数的类型安全变得困难，但你总是可以使用通用的 `LoaderContext` 类型来完成大部分工作：

```tsx
import { lazyFn } from "@tanstack/react-router";

const route = createRoute({
  path: "/my-route",
  component: MyComponent,
  loader: lazyFn(() => import("./loader"), "loader"),
});

// 在另一个文件中...
export const loader = async (context: LoaderContext) => {
  /// ...
};
```

如果你使用的是基于文件的路由，只有当你启用了 [自动代码分割](using-automatic-code-splitting) 并自定义了捆绑选项时，才能拆分 `loader`。

---

## 使用 `getRouteApi` 辅助函数在其他文件中手动访问路由 API

你可能已经猜到了，将组件代码放在与路由定义不同的文件中，会导致组件很难直接消费路由对象。为了解决这个问题，TanStack Router 导出了一个非常方便的 `getRouteApi` 函数。你可以使用它在不直接导入路由对象的情况下，访问该路由的类型安全 API。

[Image of React type-safe API access via helper function]

```tsx title="src/my-route.tsx"
import { createRoute } from "@tanstack/react-router";
import { MyComponent } from "./MyComponent";

const route = createRoute({
  path: "/my-route",
  loader: () => ({
    foo: "bar",
  }),
  component: MyComponent,
});
```

```tsx title="src/MyComponent.tsx"
import { getRouteApi } from "@tanstack/react-router";

// 获取对应路径的 API 对象
const route = getRouteApi("/my-route");

export function MyComponent() {
  const loaderData = route.useLoaderData();
  //    ^? { foo: string } 类型安全依然有效！

  return <div>...</div>;
}
```

`getRouteApi` 函数对于访问其他类型安全 API 非常有用，例如：

- `useLoaderData`
- `useLoaderDeps`
- `useMatch`
- `useParams`
- `useRouteContext`
- `useSearch`
