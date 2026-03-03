---
short_title: 外部数据加载
---

# 外部数据加载 (External Data Loading)

> [!IMPORTANT]
> 本指南侧重于外部状态管理库及其与 TanStack Router 在数据获取、SSR、脱水/复水（Hydration/Dehydration）以及流式传输方面的集成。如果您尚未阅读标准的 [数据加载](./data-loading.md) 指南，请先阅读该指南。

## 是“存储”还是“协调”？

虽然 TanStack Router 本身完全有能力存储和管理应用中的大部分数据需求，但有时您可能需要更强大的方案。

Router 的设计初衷是作为一个完美的**协调者（Coordinator）**，服务于外部数据获取和缓存库。这意味着您可以使用任何您喜欢的数据获取/缓存库，而路由器将协调数据的加载，使其与用户的导航路径以及对数据新鲜度的预期保持一致。

## 支持哪些数据获取库？

任何支持异步 Promise 的数据获取库都可以与 TanStack Router 配合使用，包括：

- [TanStack Query](https://tanstack.com/query/latest/docs/react/overview)
- [SWR](https://swr.vercel.app/)
- [RTK Query](https://redux-toolkit.js.org/rtk-query/overview)
- [urql](https://formidable.com/open-source/urql/)
- [Relay](https://relay.dev/)
- [Apollo](https://www.apollographql.com/docs/react/)

甚至是状态管理库...

- [Zustand](https://zustand-demo.pmnd.rs/)
- [Jotai](https://jotai.org/)
- [Recoil](https://recoiljs.org/)
- [Redux](https://redux.js.org/)

从字面上看，**任何能够返回 Promise 并且能读写数据**的库都可以集成。

---

## 使用加载器 (Loaders) 确保数据已就绪

将外部缓存/数据库集成到 Router 中最简单的方法是使用 `route.loader` 来确保路由所需的关键数据在渲染前已经加载完毕。

> ⚠️ **为什么要这么做？** 在加载器中预取关键渲染数据非常重要，原因有三：
>
> 1. **消除“加载闪烁”**：防止用户看到空白或加载状态突然闪现。
> 2. **消除请求瀑布流**：避免由组件级获取数据（Component-based fetching）引起的串行请求。
> 3. **SEO 友好**：如果数据在渲染时立即可用，它将更容易被搜索引擎索引。

这里有一个使用路由 `loader` 为缓存预填数据的初步演示（**请注意：这只是为了演示原理，实际开发中不要直接修改全局变量**）：

```tsx
// src/routes/posts.tsx
let postsCache = [];

export const Route = createFileRoute("/posts")({
  loader: async () => {
    // 预填缓存
    postsCache = await fetchPosts();
  },
  component: () => {
    return (
      <div>
        {postsCache.map((post) => (
          <Post key={post.id} post={post} />
        ))}
      </div>
    );
  },
});
```

这个例子虽然简陋，但它说明了一个核心点：你可以利用 `loader` 选项来提前为你的缓存灌入数据。

---

## 结合 TanStack Query 的实战示例

让我们来看一个使用 TanStack Query 的真实场景：

- 使用数据获取库的 **Prefetching API** 替换 `fetchPosts`。
- 使用数据获取库的 **Read-or-fetch API** 或 **Hook** 替换 `postsCache`。

```tsx
// src/routes/posts.tsx
const postsQueryOptions = queryOptions({
  queryKey: ["posts"],
  queryFn: () => fetchPosts(),
});

export const Route = createFileRoute("/posts")({
  // 1. 使用 loader 确保数据已同步到缓存中
  loader: () => queryClient.ensureQueryData(postsQueryOptions),
  component: () => {
    // 2. 从缓存中读取数据并订阅更新
    const {
      data: { posts },
    } = useSuspenseQuery(postsQueryOptions);

    return (
      <div>
        {posts.map((post) => (
          <Post key={post.id} post={post} />
        ))}
      </div>
    );
  },
});
```

### 配合 TanStack Query 的错误处理

当在使用 TanStack Query 的 `suspense` 模式发生错误时，你需要告诉查询库在重新渲染时尝试重试。这可以通过 `useQueryErrorResetBoundary` 钩子提供的 `reset` 函数来完成。

```tsx
export const Route = createFileRoute("/")({
  loader: () => queryClient.ensureQueryData(postsQueryOptions),
  errorComponent: ({ error, reset }) => {
    const router = useRouter();
    const queryErrorResetBoundary = useQueryErrorResetBoundary();

    useEffect(() => {
      // 当错误组件挂载时，重置查询错误边界
      queryErrorResetBoundary.reset();
    }, [queryErrorResetBoundary]);

    return (
      <div>
        {error.message}
        <button
          onClick={() => {
            // 使路由失效以重新触发 loader，并重置路由器错误边界
            router.invalidate();
          }}
        >
          重试
        </button>
      </div>
    );
  },
});
```

---

## SSR 脱水 (Dehydration) 与复水 (Hydration)

支持此功能的工具可以集成到 TanStack Router 的脱水/复水 API 中，以便在服务器和客户端之间传递数据。

### 关键数据的脱水与复水

**对于首屏渲染（First Paint）所需的关键数据**，TanStack Router 在配置 `Router` 时支持 `dehydrate` 和 `hydrate` 选项。这些回调函数会在路由器进行标准脱水/复水时自动调用，允许您将外部状态注入其中。

`dehydrate` 函数可以返回任何可序列化的 JSON 数据，这些数据将被合并并注入到发送给客户端的有效负载中。

例如，让我们将 TanStack Query 的 `QueryClient` 进行脱水处理，以便服务器上获取的数据在客户端复水时立即可用：

```tsx
// src/router.tsx

export function createRouter() {
  // 务必在 createRouter 函数内部创建 queryClient 或类似的存储。
  // 这能确保每个请求拥有独立的存储实例，且在服务器和客户端上都存在。
  const queryClient = new QueryClient();

  return createRouter({
    routeTree,
    // 建议将 queryClient 放入路由器上下文 (context) 以便调用
    context: {
      queryClient,
    },
    // 在服务器端，将 queryClient 的状态“脱水”
    dehydrate: () => {
      return {
        queryClientState: dehydrate(queryClient),
      };
    },
    // 在客户端，使用服务器传来的数据为 queryClient “复水”
    hydrate: (dehydrated) => {
      hydrate(queryClient, dehydrated.queryClientState);
    },
    // 使用 Wrap 选项，用 Provider 包装整个路由器
    Wrap: ({ children }) => {
      return (
        <QueryClientProvider client={queryClient}>
          {children}
        </QueryClientProvider>
      );
    },
  });
}
```
