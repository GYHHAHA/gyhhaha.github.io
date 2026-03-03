---
short_title: 预加载
---

# 预加载 (Preloading)

TanStack Router 中的预加载是一种在用户实际导航到某个路由之前就将其加载的方式。这对于用户接下来很可能访问的路由非常有用。例如，如果你有一个文章列表，且用户很有可能点击其中一篇，你可以预加载该文章路由，以便在用户点击时它已经准备就绪。

## 支持的预加载策略

- **Intent (意图)**
  - 通过监听 `<Link>` 组件上的悬停 (hover) 和触摸开始 (touch start) 事件来预加载目标路由的依赖项。
  - 这种策略适用于预加载用户接下来极有可能访问的路由。
- **Viewport Visibility (视口可见性)**
  - 利用 Intersection Observer API，当 `<Link>` 组件进入视口时，预加载目标路由的依赖项。
  - 这种策略适用于预加载位于首屏以下或屏幕外的路由。
- **Render (渲染)**
  - 一旦 `<Link>` 组件在 DOM 中渲染，就立即预加载目标路由的依赖项。
  - 这种策略适用于预加载那些总是需要的路由。

## 预加载的数据在内存中保留多久？

预加载的路由匹配项会暂时缓存在内存中，但有几点需要注意：

- **默认情况下，未使用的预加载数据会在 30 秒后被清除。** 你可以通过在 router 上设置 `defaultPreloadMaxAge` 选项来配置此时间。
- **显而易见的是，当一个路由被正式加载时，其预加载版本会被提升为路由正常的“待处理匹配 (pending matches)”状态。**

如果你需要对预加载数据的预加载、缓存和/或垃圾回收进行更多控制，你应该使用外部缓存库，如 [TanStack Query](https://tanstack.com/query)。

在应用程序中预加载路由最简单的方法是将整个 router 的 `defaultPreload` 选项设置为 `intent`：

```tsx
import { createRouter } from "@tanstack/react-router";

const router = createRouter({
  // ...
  defaultPreload: "intent",
});
```

这将为应用程序中所有的 `<Link>` 组件默认开启 `intent` 预加载。你也可以在单个 `<Link>` 组件上设置 `preload` 属性来覆盖默认行为。

## 预加载延迟 (Preload Delay)

默认情况下，预加载会在用户悬停或触摸 `<Link>` 组件 **50ms** 后开始。你可以通过在 router 上设置 `defaultPreloadDelay` 选项来更改此延迟：

```tsx
import { createRouter } from "@tanstack/react-router";

const router = createRouter({
  // ...
  defaultPreloadDelay: 100,
});
```

你也可以在单个 `<Link>` 组件上设置 `preloadDelay` 属性，以针对特定链接覆盖默认行为。

## 内置预加载与 `preloadStaleTime`

如果你使用的是内置的 loader，你可以通过将 `routerOptions.defaultPreloadStaleTime` 或 `routeOptions.preloadStaleTime` 设置为毫秒数，来控制预加载数据在触发另一次预加载之前被视为“新鲜 (fresh)”的时间。**默认情况下，预加载数据的新鲜度为 30 秒。**

要更改此设置，你可以在 router 上设置 `defaultPreloadStaleTime` 选项：

```tsx
import { createRouter } from "@tanstack/react-router";

const router = createRouter({
  // ...
  defaultPreloadStaleTime: 10_000,
});
```

或者，你可以在单个路由上使用 `routeOptions.preloadStaleTime` 选项：

```tsx
// src/routes/posts.$postId.tsx
export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ params }) => fetchPost(params.postId),
  // 如果预加载缓存早于 10 秒，则再次预加载该路由
  preloadStaleTime: 10_000,
});
```

## 配合外部库进行预加载

当集成像 React Query 这样拥有自己判定陈旧数据机制的外部缓存库时，你可能希望覆盖 TanStack Router 默认的预加载和 stale-while-revalidate 逻辑。这些库通常使用 `staleTime` 等选项来控制数据的时效性。

为了自定义 TanStack Router 中的预加载行为并充分利用外部库的缓存策略，你可以通过将 `routerOptions.defaultPreloadStaleTime` 或 `routeOptions.preloadStaleTime` 设置为 `0` 来绕过内置缓存。这确保了所有预加载在内部都被标记为陈旧，并且始终调用 loader，从而允许你的外部库（如 React Query）来管理数据加载和缓存。

例如：

```tsx
import { createRouter } from "@tanstack/react-router";

const router = createRouter({
  // ...
  defaultPreloadStaleTime: 0,
});
```

这样你就可以使用类似 React Query 的 `staleTime` 选项来控制预加载的新鲜度。

## 手动预加载

如果你需要手动预加载某个路由，可以使用 router 的 `preloadRoute` 方法。它接受标准的 TanStack `MapsOptions` 对象，并返回一个在路由预加载完成时解析 (resolve) 的 Promise。

```tsx
function Component() {
  const router = useRouter();

  useEffect(() => {
    async function preload() {
      try {
        const matches = await router.preloadRoute({
          to: postRoute,
          params: { id: 1 },
        });
      } catch (err) {
        // 预加载路由失败
      }
    }

    preload();
  }, [router]);

  return <div />;
}
```

如果你只需要预加载路由的 JS 分片 (chunk)，可以使用 router 的 `loadRouteChunk` 方法。它接受一个路由对象，并返回一个在路由分片加载完成时解析的 Promise。

```tsx
function Component() {
  const router = useRouter();

  useEffect(() => {
    async function preloadRouteChunks() {
      try {
        const postsRoute = router.routesByPath["/posts"];
        await Promise.all([
          router.loadRouteChunk(router.routesByPath["/"]),
          router.loadRouteChunk(postsRoute),
          router.loadRouteChunk(postsRoute.parentRoute),
        ]);
      } catch (err) {
        // 预加载路由分片失败
      }
    }

    preloadRouteChunks();
  }, [router]);

  return <div />;
}
```
