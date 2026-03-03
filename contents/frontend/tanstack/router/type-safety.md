---
short_title: 类型安全
---

# 类型安全 (Type Safety)

TanStack Router 致力于在 TypeScript 编译器和运行时的限制内实现最高级别的类型安全。这不仅意味着它由 TypeScript 编写，更意味着它能**完整推断所提供的类型，并坚持不懈地将这些类型贯穿于整个路由体验中**。

最终，这意味着作为开发者，您**编写的类型更少**，同时在代码演进过程中**对代码更有信心**。

---

## 路由定义 (Route Definitions)

### 基于文件的路由 (File-based Routing)

路由是分层级的，其定义也是如此。如果您使用基于文件的路由，大部分类型安全工作已经由系统自动为您完成了。

### 基于代码的路由 (Code-based Routing)

如果您直接使用 `Route` 类，则需要了解如何通过 `Route` 的 `getParentRoute` 选项来确保路由类型正确。这是因为子路由需要感知其**所有**父路由的类型。否则，那些您在 3 层之上的“布局路由”或“无路径布局路由”中解析出来的宝贵查询参数，将会消失在 JS 的虚无中。

所以，千万不要忘记将父路由传递给您的子路由！

```tsx
const parentRoute = createRoute({
  getParentRoute: () => parentRoute,
});
```

---

## 导出的 Hook、组件和工具

为了让路由器的类型能与顶层导出的 `Link`、`useNavigate`、`useParams` 等协同工作，它们必须渗透 TypeScript 模块边界并直接注册到库中。为此，我们对导出的 `Register` 接口使用了“声明合并（Declaration Merging）”。

```ts
const router = createRouter({
  // ...
});

declare module "@tanstack/react-router" {
  interface Register {
    router: typeof router;
  }
}
```

通过在模块中注册您的路由器，您现在可以使用带有路由器精确类型的导出 Hook、组件和工具。

---

## 解决组件上下文问题 (Fixing the Component Context Problem)

组件上下文（Component Context）在 React 等框架中是提供依赖的绝佳工具。然而，如果该上下文在组件层级移动时类型发生了变化，TypeScript 就无法知道如何推断这些变化。为了解决这个问题，基于上下文的 Hook 和组件需要您给它们一个“暗示”，说明它们在何处以及如何被使用。

```tsx
export const Route = createFileRoute("/posts")({
  component: PostsComponent,
});

function PostsComponent() {
  // 每个路由都拥有 TanStack Router 大多数内置 Hook 的类型安全版本
  const params = Route.useParams();
  const search = Route.useSearch();

  // 某些 Hook 需要来自 *整个* 路由器的上下文，而不仅仅是当前路由。
  // 为了在这里实现类型安全，我们必须传递 `from` 参数，告诉 Hook 我们在路由层级中的相对位置。
  const navigate = useNavigate({ from: Route.fullPath });
  // ... 其他 Hook
}
```

每一个需要上下文暗示的 Hook 和组件都有一个 `from` 参数，您可以在其中传入您当前渲染所在的路由 ID 或路径。

> 🧠 **小技巧**：如果您的组件是经过代码分割的，可以使用 [getRouteApi 函数](./code-splitting.md) 来获取类型化的 `useParams()` 和 `useSearch()` 钩子，从而避免手动传入 `Route.fullPath`。

### 如果我不知道当前路由是什么？如果是共享组件怎么办？

`from` 属性是可选的。这意味着如果您不传递它，路由器将根据现有的路由类型进行“最佳猜测”。通常，这意味着您会得到路由器中所有路由类型的并集。

### 如果我传错了 `from` 路径会怎样？

技术上你可以传递一个满足 TypeScript 但与运行时实际渲染路由不匹配的 `from`。在这种情况下，支持 `from` 的 Hook 和组件会检测您的预期是否与实际路由匹配，如果不匹配，将抛出运行时错误。

### 如果我无法传递 `from`，但又想避免运行时错误？

如果您正在渲染一个跨多个路由共享的组件，或者组件不在路由内，您可以传递 `strict: false` 来代替 `from` 选项。这不仅会静默运行时错误，还会为您提供松散但准确的类型。

```tsx
function MyComponent() {
  const search = useSearch({ strict: false });
  // search 变量将被类型化为路由器中所有路由可能存在的查询参数的并集。
}
```

---

## 路由器上下文 (Router Context)

路由器上下文极其有用，它是终极的层级依赖注入方案。您可以为路由器以及它渲染的每一个路由提供上下文。随着您构建此上下文，TanStack Router 会将其随路由层级向下合并，以便每个路由都能访问其所有父路由的上下文。

`createRootRouteWithContext` 工厂函数创建了一个带有实例类型的路由器，这会要求您在创建路由器时履行相同的类型契约，并确保您的上下文在整个路由树中都得到正确的类型化。

```tsx
const rootRoute = createRootRouteWithContext<{ whateverYouWant: true }>()({
  component: App,
});

const routeTree = rootRoute.addChildren([
  // ... 所有子路由在其上下文中都能访问到 `whateverYouWant`
]);

const router = createRouter({
  routeTree,
  context: {
    // 现在必须传递这个属性
    whateverYouWant: true,
  },
});
```

---

## 性能建议 (Performance Recommendations)

随着应用规模扩大，TypeScript 的检查时间自然会增加。以下是一些在应用扩展时保持 TS 检查性能的建议。

### 仅推断您需要的类型

客户端数据缓存（如 TanStack Query）的一个常见模式是预取数据。

```tsx
// ❌ 可能会影响 TS 性能的写法
export const Route = createFileRoute("/posts/$postId/deep")({
  loader: ({ context: { queryClient }, params: { postId } }) =>
    queryClient.ensureQueryData(postQueryOptions(postId)),
  component: PostDeepComponent,
});
```

在这种情况下，TS 必须推断加载器（loader）的返回类型，即使它在路由中从未被使用过。如果加载器返回的是一个极其复杂的类型，且有许多路由都这样做，会拖慢编辑器的响应速度。**改进方法很简单：让 TypeScript 推断 `Promise<void>` 即可：**

```tsx
// ✅ 性能更好的写法
export const Route = createFileRoute("/posts/$postId/deep")({
  loader: async ({ context: { queryClient }, params: { postId } }) => {
    // 使用 await 但不返回结果，推断类型为 Promise<void>
    await queryClient.ensureQueryData(postQueryOptions(postId));
  },
  component: PostDeepComponent,
});
```

这样加载器数据永远不会被推断，它将推断压力从路由树移动到了你第一次真正使用数据的地方（例如在组件中使用 `useSuspenseQuery`）。

### 尽可能窄化到相关的路由 (Narrow to relevant routes)

考虑以下 `Link` 的用法：

```tsx
<Link to=".." search={{ page: 0 }} />
<Link to="." search={{ page: 0 }} />
```

**这些示例对 TypeScript 的性能非常不利。** 这是因为 `search` 属性会解析为路由器中所有路由的 `search` 参数的并集（Union），TypeScript 必须根据这个巨大的并集来检查你传递给 `search` 属性的内容。随着应用规模的扩大，这种检查时间会随路由和查询参数的数量线性增长。虽然我们已尽力对此进行了优化（TypeScript 通常会完成一次检查并缓存结果），但针对庞大并集的初始检查开销依然巨大。这也适用于 `params` 以及 `useSearch`、`useParams`、`useNavigate` 等其他 API。

相反，你应该尝试通过 `from` 或 `to` 来**窄化**相关的路由范围。

```tsx
<Link from={Route.fullPath} to=".." search={{page: 0}} />
<Link from="/posts" to=".." search={{page: 0}} />
```

记住，你始终可以向 `to` 或 `from` 传递一个并集，以缩小你感兴趣的路由范围：

```tsx
const from: '/posts/$postId/deep' | '/posts/' = '/posts/'
<Link from={from} to='..' />
```

你也可以向 `from` 传递路由分支，使其仅解析来自该分支任何后代的 `search` 或 `params`：

```tsx
const from = '/posts'
<Link from={from} to='..' />
```

`/posts` 可以是一个拥有许多后代的分支，这些后代共享相同的 `search` 或 `params`。

---

### 考虑使用 `addChildren` 的对象语法

路由通常拥有 `params`、`search`、`loaders` 或 `context`，甚至可以引用外部依赖，这些都会对 TS 推断造成沉重负担。对于此类应用，在创建路由树时，**使用对象（Object）语法比使用元组（Tuple）性能更高**。

`createChildren` 也可以接受一个对象。对于具有复杂路由和外部库的大型路由树，使用对象进行类型检查的速度要比大型元组快得多。性能提升的程度取决于你的项目、外部依赖项以及这些库的类型定义编写方式。

```tsx
const routeTree = rootRoute.addChildren({
  postsRoute: postsRoute.addChildren({ postRoute, postsIndexRoute }),
  indexRoute,
});
```

注意，这种语法虽然略显冗长，但 TS 性能更好。在使用基于文件的路由时，路由树是自动生成的，因此无需担心手动编写冗长路由树的问题。

---

### 避免在不窄化的情况下使用内部类型

你可能想复用库中暴露的类型。例如，你可能会倾向于像这样直接使用 `LinkProps`：

```tsx
// ❌ 这样做对 TS 性能非常、非常不利
const props: LinkProps = {
  to: '/posts/',
}

return (
  <Link {...props}>
)
```

**这对 TS 性能来说是非常糟糕的。** 问题在于 `LinkProps` 没有任何类型参数，因此它是一个**极其庞大**的类型。它包含了所有路由 `search` 参数的并集，包含了所有 `params` 的并集。当将此对象与 `Link` 合并时，它会对这个庞然大物进行结构化比较。

相反，你应该使用 `as const satisfies` 来推断精确的类型，而不是直接使用 `LinkProps`，从而避开繁重的检查：

```tsx
const props = {
  to: '/posts/',
} as const satisfies LinkProps

return (
  <Link {...props}>
)
```

由于 `props` 不是泛型的 `LinkProps` 类型，它的类型更加精确，因此检查成本更低。你还可以通过进一步窄化 `LinkProps` 来优化类型检查：

```tsx
const props = {
  to: '/posts/',
} as const satisfies LinkProps<RegisteredRouter, string, '/posts/'>

return (
  <Link {...props}>
)
```

这甚至更快，因为我们是在针对窄化后的 `LinkProps` 类型进行检查。你也可以使用这种方法将 `LinkProps` 窄化为特定类型，用作组件的 Prop 或函数参数：

```tsx
export const myLinkProps = [
  {
    to: "/posts",
  },
  {
    to: "/posts/$postId",
    params: { postId: "postId" },
  },
] as const satisfies ReadonlyArray<LinkProps>;

export type MyLinkProps = (typeof myLinkProps)[number];

const MyComponent = (props: { linkProps: MyLinkProps }) => {
  return <Link {...props.linkProps} />;
};
```

这比直接在组件中使用 `LinkProps` 快得多，因为 `MyLinkProps` 是一个精确得多的类型。

#### 另一种方案：控制反转 (Render Props)

另一种解决方案是不使用 `LinkProps`，而是采用**控制反转**，渲染一个窄化到特定路由的 `Link` 组件。Render Props 是将控制权移交给组件使用者的好方法：

```tsx
export interface MyComponentProps {
  readonly renderLink: () => React.ReactNode;
}

const MyComponent = (props: MyComponentProps) => {
  return <div>{props.renderLink()}</div>;
};

const Page = () => {
  // 这里非常快，因为 Link 已被窄化到我们要导航到的具体路由
  return <MyComponent renderLink={() => <Link to="/absolute" />} />;
};
```
