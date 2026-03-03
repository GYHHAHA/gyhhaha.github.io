---
short_title: 推迟数据加载
---

# 延迟数据加载 (Deferred Data Loading)

TanStack Router 的设计初衷是并行运行加载器（Loaders），并在渲染下一个路由之前等待所有加载器解析完成。大多数情况下这很棒，但有时，你可能希望在后台加载剩余数据的同时，先给用户展示一些内容。

**延迟数据加载**是一种允许路由器渲染下一位置的“关键数据/标记”，同时在后台解析较慢的“非关键路由数据”的模式。这一过程在客户端和服务器端（通过流式传输）均可工作，是提升应用程序“感知性能”的绝佳方式。

> [!NOTE]
> 如果你使用的是像 [TanStack Query](https://tanstack.com/query/latest) 这样的外部数据获取库，延迟加载的工作方式会有所不同。请跳至[结合外部库进行延迟数据加载](deferred-data-loading-with-external-libraries)章节。

---

## 使用 `Await` 实现延迟加载

要延迟加载缓慢或非关键的数据，只需在加载器响应中返回一个**未等待（unawaited）/未解析（unresolved）**的 Promise 即可：

```tsx
// src/routes/posts.$postId.tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/posts/$postId")({
  loader: async () => {
    // 获取一些较慢的数据，但不要 await 它
    const slowDataPromise = fetchSlowData();

    // 获取并 await 一些快速解析的数据
    const fastData = await fetchFastData();

    return {
      fastData,
      deferredSlowData: slowDataPromise,
    };
  },
});
```

一旦所有被 `await` 的 Promise 解析完成，下一个路由就会开始渲染，而延迟的 Promise 则继续在后台解析。

在组件中，可以使用 `Await` 组件来处理这些延迟的 Promise：

```tsx
// src/routes/posts.$postId.tsx
import { createFileRoute, Await } from "@tanstack/react-router";

export const Route = createFileRoute("/posts/$postId")({
  // ...
  component: PostIdComponent,
});

function PostIdComponent() {
  const { deferredSlowData, fastData } = Route.useLoaderData();

  // 处理 fastData...

  return (
    <div>
      <h1>{fastData.title}</h1>

      {/* 使用 Await 处理延迟数据 */}
      <Await promise={deferredSlowData} fallback={<div>加载中...</div>}>
        {(data) => {
          return <div>{data}</div>;
        }}
      </Await>
    </div>
  );
}
```

`Await` 组件通过触发最近的 **Suspense 边界**来解析 Promise。解析完成后，它会将结果传递给子函数进行渲染。

> [!TIP]
> 如果你使用的是 **React 19**，你可以直接使用 `use()` 钩子来代替 `Await` 组件。

---

## 结合外部库进行延迟加载

当你依赖像 [TanStack Query](https://tanstack.com) 这样的外部库进行数据获取时，延迟加载的工作方式有所不同，因为外部库在 TanStack Router 之外处理数据获取和缓存。

在这种情况下，你不需要使用 `defer` 和 `Await`，而是利用 `loader` 触发预取（Prefetch），然后使用库自带的 Hooks：

```tsx
// src/routes/posts.$postId.tsx
export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ context: { queryClient } }) => {
    // 触发慢速数据的获取，但不等待它
    queryClient.prefetchQuery(slowDataOptions());

    // 确保快速数据已获取并等待它
    await queryClient.ensureQueryData(fastDataOptions());
  },
});
```

组件内部写法：

```tsx
function PostIdComponent() {
  // 快速数据直接通过 Suspense Query 获取
  const fastData = useSuspenseQuery(fastDataOptions());

  return (
    <div>
      <Suspense fallback={<div>加载中...</div>}>
        <SlowDataComponent />
      </Suspense>
    </div>
  );
}

function SlowDataComponent() {
  // 慢速数据在子组件中触发 Suspense
  const data = useSuspenseQuery(slowDataOptions());
  return <div>{data}</div>;
}
```

---

## SSR 与流式传输延迟数据

**流式传输需要服务器支持，并正确配置 TanStack Router。**

请查阅 [流式传输 SSR 指南](./ssr.md#streaming-ssr) 以了解详细步骤。

### SSR 流式传输生命周期

以下是延迟数据流在 TanStack Router 中工作的简要流程：

- **服务器端 (Server)**
  - 路由加载器返回 Promise 并被跟踪。
  - 基础数据解析完成，延迟的 Promise 被序列化并嵌入 HTML。
  - 页面开始渲染，`<Await>` 触发 Suspense 边界，允许服务器流式传输已完成的 HTML。
- **客户端 (Client)**
  - 接收初始 HTML。
  - `<Await>` 组件进入挂起状态，等待服务器发送解析后的数据。
- **服务器端 (Server)**
  - 当延迟 Promise 解析后，结果通过内联 `<script>` 标签流式传输到客户端。
  - 相应的 HTML 片段随之发送。
- **客户端 (Client)**
  - 占位 Promise 被流式传输的数据填满，`<Await>` 组件结束挂起，渲染最终结果。
