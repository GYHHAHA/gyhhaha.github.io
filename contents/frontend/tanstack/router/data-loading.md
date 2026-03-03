---
short_title: 数据加载
---

# 数据加载 (Data Loading)

数据加载是 Web 应用程序的核心关注点，且与路由密不可分。在加载页面时，最理想的情况是并行、尽早地获取并完成所有异步需求。路由器是协调这些异步依赖的最佳场所，因为它通常是应用中唯一在内容渲染前就知道用户要去向何处的组件。

你可能熟悉 Next.js 的 `getServerSideProps` 或 Remix/React-Router 的 `loader`。TanStack Router 具有类似的功能，可以按路由并行预加载/加载资源，使其在通过 Suspense 进行获取时能以最快速度渲染。

除了这些路由器的常规预期外，TanStack Router 还更进一步，提供了 **内置的 SWR 缓存**——一个用于路由加载器的长期内存缓存层。这意味着你可以使用 TanStack Router 预加载路由数据，使其实现瞬时加载，或者临时缓存已访问路由的数据以便稍后再次使用。

## 路由加载生命周期 (The route loading lifecycle)

每当检测到 URL/历史记录更新时，路由器会按顺序执行以下操作：

- **路由匹配 (Route Matching)**（自顶向下）
  - `route.params.parse`
  - `route.validateSearch`
- **路由预加载 (Route Pre-Loading)**（串行）
  - `route.beforeLoad`
  - `route.onError`
    - `route.errorComponent` / `parentRoute.errorComponent` / `router.defaultErrorComponent`
- **路由加载 (Route Loading)**（并行）
  - `route.component.preload?`
  - `route.loader`
    - `route.pendingComponent`（可选）
    - `route.component`
  - `route.onError`
    - `route.errorComponent` / `parentRoute.errorComponent` / `router.defaultErrorComponent`

## 要不要使用路由器缓存？

对于大多数中小型应用，TanStack 的路由器缓存（Router Cache）很可能非常适用，但了解使用它与使用更强大的缓存方案（如 TanStack Query）之间的权衡非常重要：

**使用路由器缓存的优点 (Pros)：**

- **内置**：易于使用，无需额外依赖。
- **功能全**：支持按路由的去重（deduping）、预加载（preloading）、加载（loading）、过期即重新验证（stale-while-revalidate）和后台重新获取。
- **粗粒度失效**：支持一次性使所有路由和缓存失效。
- **自动垃圾回收**：自动清理不再使用的缓存。
- **共享少**：非常适合路由间共享数据较少的应用。
- **SSR 支持**：在服务器端渲染中“开箱即用”。

**使用路由器缓存的缺点 (Cons)：**

- 没有持久化适配器/模型（无法轻松持久化到本地存储）。
- 路由之间没有共享缓存/去重。
- 没有内置的 Mutation（变更）API（虽然很多示例中提供了一个基础的 `useMutation` 钩子，足以满足许多用例）。
- 没有内置的缓存级乐观更新 API（你仍然可以通过像 `useMutation` 这样的钩子在组件层级实现这一点）。

> [!TIP]
> 如果你明确知道需要使用更强大的方案（如 TanStack Query），请跳至 [外部数据加载](./external-data-loading.md) 指南。

## 使用路由器缓存

路由器缓存是内置的，只要从任何路由的 `loader` 函数中返回数据即可使用。让我们来看看如何操作！

## 路由加载器 (`loader`)

当路由匹配并加载时会调用 `loader` 函数。调用时会传入一个包含许多实用属性的对象参数。我们稍后会详细介绍这些参数，先看一个基础示例：

```tsx
// src/routes/posts.tsx
export const Route = createFileRoute("/posts")({
  loader: () => fetchPosts(),
});
```

## `loader` 参数

`loader` 函数接收一个包含以下属性的对象：

- **`abortController`**：路由的 AbortController。当路由卸载、不再相关或当前的 `loader` 调用已过时时，其信号（signal）会被取消。
- **`cause`**：当前路由匹配的原因。可以是以下之一：
  - `enter`：路由匹配且是在上一个位置未匹配的情况下加载的。
  - `preload`：路由正在被预加载。
  - `stay`：路由在上一个位置匹配后，当前位置依然匹配。
- **`context`**：路由的上下文对象，它是以下内容的合并：
  - 父路由上下文。
  - 由 `beforeLoad` 选项提供的当前路由上下文。
- **`deps`**：由 `Route.loaderDeps` 函数返回的对象值。如果未定义 `Route.loaderDeps`，则为空对象。
- **`location`**：当前位置信息。
- **`params`**：路由的路径参数（Path Params）。
- **`parentMatchPromise`**：`Promise<RouteMatch>`（根路由下为 `undefined`）。
- **`preload`**：布尔值，当路由是预加载而非正常加载时为 `true`。
- **`route`**：路由对象本身。

利用这些参数我们可以实现很多强大的功能，但首先，让我们看看如何消费这些数据。

## 消费来自加载器的数据

要消费来自 `loader` 的数据，请使用路由对象上定义的 `useLoaderData` 钩子。

```tsx
const posts = Route.useLoaderData();
```

如果你无法直接访问路由对象（例如在组件树的深层），可以使用 `getRouteApi` 来访问相同的钩子。这比直接导入路由对象更好，因为直接导入容易导致循环依赖。

```tsx
import { getRouteApi } from "@tanstack/react-router";

// 在你的组件中
const routeApi = getRouteApi("/posts");
const data = routeApi.useLoaderData();
```

## 基于依赖的过期即重新验证 (SWR) 缓存

TanStack Router 为路由加载器提供了内置的 SWR 缓存层，其键（Key）基于路由的依赖项：

- 路由完整解析后的 **Pathname**（例如 `/posts/1` 与 `/posts/2` 会被视为不同的键）。
- 由 **`loaderDeps`** 选项提供的任何额外依赖项（例如：`loaderDeps: ({ search: { pageIndex, pageSize } }) => ({ pageIndex, pageSize })`）。

使用这些依赖项作为键，TanStack Router 会缓存 `loader` 函数返回的数据，并用它来满足后续对同一路由匹配的请求。如果数据已在缓存中，它会立即返回，然后根据数据的“新鲜度（freshness）”**可能**在后台重新获取。

### 关键配置项

为了控制路由器依赖项和“新鲜度”，TanStack Router 提供了大量选项：

- **`routeOptions.loaderDeps`**：一个提供搜索参数（search params）并返回依赖对象的函数。当这些依赖项在导航间发生变化时，无论 `staleTime` 如何，路由都会重新加载。依赖项使用深比较（deep equality check）。
- **`routeOptions.staleTime` / `routerOptions.defaultStaleTime`**：数据被视为“新鲜”的毫秒数。
- **`routeOptions.preloadStaleTime` / `routerOptions.defaultPreloadStaleTime`**：尝试预加载时，数据被视为“新鲜”的毫秒数。
- **`routeOptions.gcTime` / `routerOptions.defaultGcTime`**：数据在被垃圾回收前在缓存中保留的毫秒数。
- **`routeOptions.shouldReload`**：接收与 `beforeLoad` 相同的参数，返回布尔值。它提供了比 `staleTime` 和 `loaderDeps` 更高级的控制，类似于 Remix 的 `shouldLoad`。

### ⚠️ 一些重要的默认设置

- 默认情况下，**`staleTime` 设置为 `0`**，这意味着路由数据始终被视为过期的，重访路由时总会在后台重新加载。
- 默认情况下，**预加载的路由新鲜度为 30 秒**。这意味着如果一个路由被预加载后，30 秒内再次触发预加载，第二次会被忽略。**当路由正常加载时，使用的是标准的 `staleTime`。**
- 默认情况下，**`gcTime` 设置为 30 分钟**。
- **`router.invalidate()`** 会强制所有激活的路由立即重新执行加载器，并标记所有缓存数据为过期。

### 使用 `loaderDeps` 访问搜索参数

假设 `/posts` 路由通过搜索参数 `offset` 和 `limit` 支持分页。为了让缓存唯一地存储这些数据，我们需要通过 `loaderDeps` 函数访问这些参数。通过显式标识它们，具有不同 `offset` 和 `limit` 的 `/posts` 路由匹配就不会混淆！

```tsx
// /routes/posts.tsx
export const Route = createFileRoute("/posts")({
  loaderDeps: ({ search: { offset, limit } }) => ({ offset, limit }),
  loader: ({ deps: { offset, limit } }) =>
    fetchPosts({
      offset,
      limit,
    }),
});
```

> [!WARNING]
> **只包含你在 loader 中实际使用的依赖项。**
>
> 一个常见的错误是返回整个 `search` 对象：
>
> ```tsx
> // ❌ 不要这样做——这会导致不必要的缓存失效
> loaderDeps: ({ search }) => search,
> loader: ({ deps }) => fetchPosts({ page: deps.page }), // 实际上只用到了 page！
> ```
>
> 这会导致只要有任何搜索参数变化（哪怕是 loader 没用到的 `viewMode`），路由都会重新加载。**正确做法：** 只提取需要的字段。
>
> ```tsx
> // ✅ 这样做——只有当用到的参数变化时才重新加载
> loaderDeps: ({ search }) => ({
>   page: search.page,
>   limit: search.limit,
> }),
> loader: ({ deps }) => fetchPosts(deps),
> ```

### 使用 `staleTime` 控制数据新鲜度

默认情况下，导航的 `staleTime` 为 `0`ms。**这对于大多数用例来说是很好的默认值，但你可能会发现某些路由数据非常静态，或者加载成本很高。** 这时你可以使用 `staleTime` 控制：

```tsx
// /routes/posts.tsx
export const Route = createFileRoute("/posts")({
  loader: () => fetchPosts(),
  // 将路由数据视为新鲜的时间设为 10 秒
  staleTime: 10_000,
});
```

通过传入 `10_000`，如果用户在上次获取结果后的 10 秒内从 `/about` 导航回 `/posts`，数据将不会重新加载。超过 10 秒后，数据才会在后台重新获取。

## 关闭过期即重新验证 (SWR) 缓存

如果您希望为一个路由完全禁用 SWR 缓存，可以将 `staleTime` 设置为 `Infinity`。这意味着数据一旦加载，就会被视为“永远新鲜”，除非手动触发失效，否则路由器不会在后台重新获取它。

```tsx
// /routes/posts.tsx
export const Route = createFileRoute("/posts")({
  loader: () => fetchPosts(),
  // 将数据视为永久新鲜
  staleTime: Infinity,
});
```

您甚至可以在路由器级别为所有路由统一下达此指令：

```tsx
const router = createRouter({
  routeTree,
  defaultStaleTime: Infinity,
});
```

---

## 使用 `shouldReload` 和 `gcTime` 选择性退出缓存

类似于 Remix 的默认行为，您可能希望配置一个路由仅在初次进入或关键的加载器依赖（loader deps）发生变化时才加载。您可以结合使用 `gcTime` 和 `shouldReload` 选项来实现。`shouldReload` 接收一个布尔值，或者一个接收 `beforeLoad` 和 `loaderContext` 参数并返回布尔值的函数。

```tsx
// /routes/posts.tsx
export const Route = createFileRoute("/posts")({
  loaderDeps: ({ search: { offset, limit } }) => ({ offset, limit }),
  loader: ({ deps }) => fetchPosts(deps),
  // 路由卸载后立即清理此路由的数据缓存
  gcTime: 0,
  // 仅在用户初次导航至该路由或依赖项变化时重新加载
  shouldReload: false,
});
```

### 在退出缓存的同时保留预加载优势

即使您选择了不为路由数据进行短期缓存，您仍然可以享受预加载（Preloading）带来的好处！在上述配置下，预加载依然会配合默认的 `preloadGcTime` 正常工作。这意味着如果一个路由被预加载了，随后用户立即导航过去，路由数据会被视为新鲜且不会重新加载。

如果您想完全禁用预加载，请不要在 `routerOptions.defaultPreload` 或 `routeOptions.preload` 中开启它。

---

## 将加载器事件传递给外部缓存

我们在 [外部数据加载](./external-data-loading.md) 页面详细拆解了这一用例。但简而言之，如果您想使用像 TanStack Query 这样的外部缓存，只需将所有加载器事件转发给它即可。只要您使用的是默认设置，唯一的改动就是在路由器上将 `defaultPreloadStaleTime` 设置为 `0`：

```tsx
const router = createRouter({
  routeTree,
  defaultPreloadStaleTime: 0,
});
```

这将确保每一次预加载（preload）、加载（load）和重新加载（reload）事件都会触发您的 `loader` 函数，进而由您的外部缓存进行处理和去重。

---

## 使用路由器上下文 (Router Context)

传递给 `loader` 函数的 `context` 参数是一个合并后的对象，包含：

- 父路由上下文
- 由当前路由的 `beforeLoad` 选项提供的上下文

从路由器的最顶层开始，您可以通过 `context` 选项向路由器传递初始上下文。此上下文对所有路由可见，并在路由匹配时被每个路由复制和扩展。通过 `beforeLoad` 传递给某个路由的上下文，会对该路由的所有子路由可见，并最终提供给路由的 `loader` 函数。

> 🧠 **上下文是实现依赖注入（Dependency Injection）的利器。** 您可以用它将服务、Hook 或其他对象注入到路由器和路由中。

### 示例：依赖注入

- `/utils/fetchPosts.tsx`

```tsx
export const fetchPosts = async () => {
  const res = await fetch(`/api/posts`);
  if (!res.ok) throw new Error("获取帖子失败");
  return res.json();
};
```

- `/routes/__root.tsx`

```tsx
import { createRootRouteWithContext } from "@tanstack/react-router";

// 使用 createRootRouteWithContext<{...}>() 定义您希望在上下文可用的类型
export const Route = createRootRouteWithContext<{
  fetchPosts: typeof fetchPosts;
}>()(); // 注意：双括号调用是必须的，因为这是一个工厂函数 ;)
```

- `/routes/posts.tsx`

```tsx
// 子路由通过 context 引用 fetchPosts 函数
export const Route = createFileRoute("/posts")({
  loader: ({ context: { fetchPosts } }) => fetchPosts(),
});
```

- `/router.tsx`

```tsx
import { routeTree } from "./routeTree.gen";

// 创建路由器时，必须满足 rootRoute 定义的上下文类型要求
const router = createRouter({
  routeTree,
  context: {
    fetchPosts, // 提供具体的函数实现
  },
});
```

---

## 在加载器中使用路径参数 (Path Params)

要在 `loader` 函数中使用路径参数，可以通过函数参数中的 `params` 属性访问：

```tsx
// src/routes/posts.$postId.tsx
export const Route = createFileRoute("/posts/$postId")({
  loader: ({ params: { postId } }) => fetchPostById(postId),
});
```

---

## 深入理解 `beforeLoad` 提供路由上下文

除了全局上下文，如果您想提供特定于某个路由的上下文怎么办？这就是 `beforeLoad` 派上用场的地方。`beforeLoad` 在尝试加载路由之前运行，它不仅可以重定向或阻塞请求，还可以返回一个对象，该对象将合并到该路由及其子路由的上下文中。

```tsx
// src/routes/posts.tsx
export const Route = createFileRoute("/posts")({
  beforeLoad: () => ({
    // 注入特定于此路由的逻辑
    loggingService: () => console.info("正在加载帖子..."),
  }),
  loader: ({ context: { loggingService } }) => {
    loggingService();
    // ...
  },
});
```

---

## 在加载器中使用查询参数 (Search Params)

> ❓ 等等... 我的 `search` 参数去哪了？为什么在 `loader` 的参数里找不到它？

我们刻意没有在 `loader` 参数中直接提供 `search`。这是为了引导您走向成功的最佳实践，原因如下：

1. **缓存唯一性**：加载器中使用查询参数，意味着这些参数应被用于唯一标识所加载的数据。例如，`?pageIndex=1` 标识的数据与 `?pageIndex=2` 是不同的。
2. **避免竞态与缓存污染**：如果直接访问 `search` 而不声明依赖，可能会导致预加载出错。例如，您在当前页预加载第 2 页的数据，如果没有明确的依赖区分，预加载的数据可能会覆盖当前页面的显示。
3. **响应性阈值**：通过声明依赖，路由器能理解哪些参数变化应该触发重新加载。

### 正确做法：通过 `loaderDeps` 访问

```tsx
// /routes/users.user.tsx
export const Route = createFileRoute("/users/user")({
  validateSearch: (search) => search as { userId: string },
  // 1. 声明 loader 依赖于 userId
  loaderDeps: ({ search: { userId } }) => ({ userId }),
  // 2. 在 loader 中通过 deps 访问，确保缓存键的唯一性
  loader: async ({ deps: { userId } }) => getUser(userId),
});
```

### 结合 Zod 进行校验并传给 Loader

```tsx
// /routes/posts.tsx
export const Route = createFileRoute("/posts")({
  validateSearch: z.object({
    offset: z.number().int().nonnegative().catch(0),
  }),
  // 将 offset 传递给 loaderDeps
  loaderDeps: ({ search: { offset } }) => ({ offset }),
  // 从 deps 中解构 offset
  loader: async ({ deps: { offset } }) => fetchPosts({ offset }),
});
```

## 使用 Abort Signal 取消请求

`loader` 函数的 `abortController` 属性是一个标准的 [AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)。当路由卸载（Unload）或当前的 `loader` 调用因参数变化而过时时，其信号（signal）会被取消。这对于在用户离开页面时及时取消网络请求非常有用。

以下是将它与 `fetch` 结合使用的示例：

```tsx
// src/routes/posts.tsx
export const Route = createFileRoute("/posts")({
  loader: ({ abortController }) =>
    fetchPosts({
      // 将信号传递给底层的 fetch 调用或任何支持 Signal 的异步操作
      signal: abortController.signal,
    }),
});
```

---

## 使用 `preload` 标志

`loader` 函数的 `preload` 属性是一个布尔值。当路由处于“预加载”状态而非正式“加载”状态时，该值为 `true`。某些数据加载库处理预加载的方式可能与标准 Fetch 不同，因此你可能需要将此标志传递给你的数据库，或据此执行特定的逻辑：

```tsx
// src/routes/posts.tsx
export const Route = createFileRoute("/posts")({
  loader: async ({ preload }) =>
    fetchPosts({
      // 预加载的数据可以设置稍长一点的有效期 (maxAge)
      maxAge: preload ? 10_000 : 0,
    }),
});
```

---

## 处理加载缓慢的 Loader

理想情况下，大多数路由加载器都能在瞬间解析数据，从而直接利用 Suspense 在准备就绪后渲染下一路由。但当关键数据加载确实很慢时，你有两种选择：

1. **拆分数据**：将数据拆分为“快”和“慢”两部分，并对慢数据使用 `defer`（详见 [延迟数据加载](./deferred-data-loading.md) 指南）。
2. **显示等待组件**：在达到乐观的时间阈值后显示加载界面（见下文）。

---

## 显示等待组件 (Pending Component)

**默认情况下，TanStack Router 会为加载时间超过 1 秒的 Loader 显示一个“等待组件”。** 这是一个乐观的阈值，可以通过以下选项配置：

- `routeOptions.pendingMs` 或
- `routerOptions.defaultPendingMs`

当超过该时间阈值时，如果配置了 `pendingComponent` 选项，路由器将渲染该组件。

### 避免加载动画闪现 (Avoiding Pending Component Flash)

使用等待组件时，最糟糕的体验是：刚好达到 1 秒阈值显示了加载动画，结果数据在 1.1 秒就回来了，导致动画闪现一下就消失了。为了避免这种违和感，**TanStack Router 默认会让等待组件至少显示 500 毫秒**。该阈值可以通过以下选项配置：

- `routeOptions.pendingMinMs` 或
- `routerOptions.defaultPendingMinMs`

---

## 错误处理 (Handling Errors)

TanStack Router 提供了多种方式来处理路由加载生命周期中发生的错误。

### 使用 `onError` 处理错误

`routeOptions.onError` 选项是一个函数，在路由加载期间发生错误时被调用，通常用于日志记录。

```tsx
// src/routes/posts.tsx
export const Route = createFileRoute("/posts")({
  loader: () => fetchPosts(),
  onError: ({ error }) => {
    // 记录错误日志
    console.error(error);
  },
});
```

### 使用 `onCatch` 处理错误

`routeOptions.onCatch` 选项是一个函数，每当错误被路由器的 `CatchBoundary` 捕获时就会被调用。

```tsx
export const Route = createFileRoute("/posts")({
  onCatch: ({ error, errorInfo }) => {
    console.error("被 CatchBoundary 捕获:", error);
  },
});
```

### 使用 `errorComponent` 显示错误界面

`routeOptions.errorComponent` 选项是一个组件，当路由加载或渲染生命周期发生错误时渲染。它接收以下 Props：

- `error`：发生的错误对象。
- `reset`：一个用于重置内部 `CatchBoundary` 的函数。

```tsx
// src/routes/posts.tsx
export const Route = createFileRoute("/posts")({
  loader: () => fetchPosts(),
  errorComponent: ({ error, reset }) => {
    return (
      <div>
        <p>出错了：{error.message}</p>
        <button onClick={() => reset()}>重试</button>
      </div>
    );
  },
});
```

如果错误是由 `loader` 数据获取失败导致的，建议调用 `router.invalidate()`，它会同时触发路由重新加载和错误边界重置：

```tsx
// src/routes/posts.tsx
export const Route = createFileRoute("/posts")({
  loader: () => fetchPosts(),
  errorComponent: ({ error }) => {
    const router = useRouter();

    return (
      <div>
        <p>{error.message}</p>
        <button onClick={() => router.invalidate()}>重新获取数据并重试</button>
      </div>
    );
  },
});
```

### 使用默认的 `ErrorComponent`

TanStack Router 提供了一个默认的 `ErrorComponent`。如果你选择覆盖特定路由的错误组件，建议在处理完自定义逻辑后，对于不匹配的错误依然回退到默认组件：

```tsx
import { ErrorComponent } from "@tanstack/react-router";

export const Route = createFileRoute("/posts")({
  loader: () => fetchPosts(),
  errorComponent: ({ error }) => {
    if (error instanceof MyCustomError) {
      return <div>特定业务错误：{error.message}</div>;
    }

    // 回退到 TanStack Router 默认的错误组件
    return <ErrorComponent error={error} />;
  },
});
```
