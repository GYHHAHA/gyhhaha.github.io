---
short_title: 路由器上下文
---

# 路由上下文 (Router Context)

TanStack Router 的路由上下文是一个非常强大的工具，可用于依赖注入等多种用途。顾名思义，路由上下文贯穿于路由器，并向下传递给每一个匹配的路由。在层级结构中的每个路由上，都可以修改或增加上下文内容。以下是路由上下文的一些实际用途：

- **依赖注入**
  - 你可以提供各种依赖项（例如 loader 函数、数据获取客户端、Mutation 服务），路由及其所有子路由无需直接导入或创建即可访问并使用它们。
- **面包屑 (Breadcrumbs)**
  - 虽然每个路由的主上下文对象在向下传递时会被合并，但每个路由独有的上下文也会被存储，这使得为每个路由的上下文附加面包屑信息或方法成为可能。
- **动态元标签 (Meta Tag) 管理**
  - 你可以将元标签附加到每个路由的上下文中，然后使用元标签管理器在用户导航时动态更新页面标签。

这些只是建议的用途，你可以根据需求随心所欲地使用路由上下文！

## 类型化路由上下文

与其它功能一样，根路由上下文是严格类型化的。该类型可以通过任何路由的 `beforeLoad` 选项在路由匹配树向下合并时进行增强。要约束根路由上下文的类型，你必须使用 `createRootRouteWithContext<YourContextTypeHere>()(routeOptions)` 函数来创建根路由，而不是使用普通的 `createRootRoute()`。示例如下：

```tsx
import {
  createRootRouteWithContext,
  createRouter,
} from "@tanstack/react-router";

interface MyRouterContext {
  user: User;
}

// 使用带有上下文定义的 routerContext 来创建你的根路由
const rootRoute = createRootRouteWithContext<MyRouterContext>()({
  component: App,
});

const routeTree = rootRoute.addChildren([
  // ...
]);

// 使用路由树创建路由器
const router = createRouter({
  routeTree,
});
```

> [!TIP]
> `MyRouterContext` 只需要包含那些将直接传递给下方 `createRouter` 的内容。所有在 `beforeLoad` 中添加的其它上下文都会被自动推导。

## 传递初始路由上下文

路由上下文在实例化时传递给路由器。你可以通过 `context` 选项传递初始上下文：

> [!TIP]
> 如果你的上下文中有任何必选属性，而在初始上下文中未传递它们，TypeScript 将会报错。如果所有属性都是可选的，则传递 `context` 也是可选的。如果你不传递路由上下文，它默认设为 `{}`。

```tsx
import { createRouter } from "@tanstack/react-router";

// 使用你创建的路由树和初始上下文来创建路由器
const router = createRouter({
  routeTree,
  context: {
    user: {
      id: "123",
      name: "John Doe",
    },
  },
});
```

### 刷新路由上下文 (Invalidating)

如果你需要刷新传递给路由器的上下文状态，可以调用 `invalidate` 方法告知路由器重新计算上下文。当你更新了上下文状态并希望路由器为所有路由重新计算上下文时，这非常有用。

```tsx
function useAuth() {
  const router = useRouter();
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    const unsubscribe = auth.onAuthStateChanged((user) => {
      setUser(user);
      // 状态变更后让路由器失效，触发重新计算
      router.invalidate();
    });

    return unsubscribe;
  }, []);

  return user;
}
```

## 使用路由上下文

定义好路由上下文类型后，你就可以在路由定义中使用它：

```tsx
// src/routes/todos.tsx
export const Route = createFileRoute("/todos")({
  component: Todos,
  loader: ({ context }) => fetchTodosByUserId(context.user.id),
});
```

你甚至可以将数据获取和 Mutation 的实现逻辑注入其中！事实上，我们强烈建议这样做 😜。

让我们尝试注入一个简单的获取 Todo 的函数：

```tsx
const fetchTodosByUserId = async ({ userId }) => {
  const response = await fetch(`/api/todos?userId=${userId}`);
  const data = await response.json();
  return data;
};

const router = createRouter({
  routeTree: rootRoute,
  context: {
    userId: "123",
    fetchTodosByUserId,
  },
});
```

然后，在你的路由中：

```tsx
// src/routes/todos.tsx
export const Route = createFileRoute("/todos")({
  component: Todos,
  loader: ({ context }) => context.fetchTodosByUserId(context.userId),
});
```

### 配合外部数据获取库使用？

```tsx
import {
  createRootRouteWithContext,
  createRouter,
} from "@tanstack/react-router";

interface MyRouterContext {
  queryClient: QueryClient;
}

const rootRoute = createRootRouteWithContext<MyRouterContext>()({
  component: App,
});

const queryClient = new QueryClient();

const router = createRouter({
  routeTree: rootRoute,
  context: {
    queryClient,
  },
});
```

然后，在你的路由中：

```tsx
// src/routes/todos.tsx
export const Route = createFileRoute("/todos")({
  component: Todos,
  loader: async ({ context }) => {
    await context.queryClient.ensureQueryData({
      queryKey: ["todos", { userId: user.id }],
      queryFn: fetchTodos,
    });
  },
});
```

## 如何使用 React Context 或 Hooks？

当你尝试在路由的 `beforeLoad` 或 `loader` 函数中使用 React Context 或 Hooks 时，务必记住 React 的 [Hooks 规则 (Rules of Hooks)](https://react.dev/reference/rules/rules-of-hooks)。你不能在非 React 函数中使用 Hook，因此无法直接在 `beforeLoad` 或 `loader` 中调用它们。

那么，该如何变通呢？我们可以**利用路由上下文 (Router Context) 将 React Context 或 Hooks 的状态传递给路由函数**。

让我们来看一个 setup 示例，我们将 `useNetworkStrength` Hook 的返回值传递给路由的 `loader` 函数：

```tsx title="src/routes/__root.tsx"
// 首先，确保根路由的上下文已定义类型
import { createRootRouteWithContext } from "@tanstack/react-router";
import { useNetworkStrength } from "@/hooks/useNetworkStrength";

interface MyRouterContext {
  networkStrength: ReturnType<typeof useNetworkStrength>;
}

export const Route = createRootRouteWithContext<MyRouterContext>()({
  component: App,
});
```

在这个例子中，我们会在使用 `<RouterProvider />` 渲染路由器之前实例化这个 Hook。这样，Hook 就会在 React 的生命周期内被调用，从而遵循 Hooks 规则。

```tsx title="src/router.tsx"
import { createRouter } from "@tanstack/react-router";
import { routeTree } from "./routeTree.gen";

export const router = createRouter({
  routeTree,
  context: {
    // 初始设为 undefined!（非空断言），稍后在 React 组件中设置
    networkStrength: undefined!,
  },
});
```

接下来，我们在 `App` 组件中调用 `useNetworkStrength` Hook，并通过 `<RouterProvider />` 的 `context` 属性将返回值注入到路由上下文中：

```tsx title="src/main.tsx"
import { RouterProvider } from "@tanstack/react-router";
import { router } from "./router";
import { useNetworkStrength } from "@/hooks/useNetworkStrength";

function App() {
  const networkStrength = useNetworkStrength();

  // 将 Hook 返回的值注入到路由上下文中
  return <RouterProvider router={router} context={{ networkStrength }} />;
}

// ...
```

现在，在路由的 `loader` 函数中，我们就可以从路由上下文中访问 `networkStrength` 了：

```tsx title="src/routes/posts.tsx"
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/posts")({
  component: Posts,
  loader: ({ context }) => {
    if (context.networkStrength === "STRONG") {
      // 执行相关操作
    }
  },
});
```

## 修改路由上下文

路由上下文会沿着路由树向下传递，并在每个路由节点进行合并。这意味着你可以在每个路由节点修改上下文，这些修改将对所有子路由可见。示例如下：

```tsx title="src/routes/__root.tsx"
import { createRootRouteWithContext } from "@tanstack/react-router";

interface MyRouterContext {
  foo: boolean;
}

export const Route = createRootRouteWithContext<MyRouterContext>()({
  component: App,
});
```

```tsx title="src/router.tsx"
import { createRouter } from "@tanstack/react-router";
import { routeTree } from "./routeTree.gen";

const router = createRouter({
  routeTree,
  context: {
    foo: true,
  },
});
```

```tsx title="src/routes/todos.tsx"
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/todos")({
  component: Todos,
  beforeLoad: () => {
    // 这里返回的对象会被合并到上下文中
    return {
      bar: true,
    };
  },
  loader: ({ context }) => {
    console.log(context.foo); // true
    console.log(context.bar); // true
  },
});
```

## 处理累积的路由上下文

上下文（特别是独立的路由 `context` 对象）使得累积和处理所有匹配路由的上下文变得非常简单。以下是一个使用所有匹配路由的上下文来生成面包屑导航（Breadcrumbs）的示例：

```tsx title="src/routes/__root.tsx"
export const Route = createRootRoute({
  component: () => {
    // 获取当前所有匹配的路由状态
    const matches = useRouterState({ select: (s) => s.matches });

    const breadcrumbs = matches
      .filter((match) => match.context.getTitle)
      .map(({ pathname, context }) => {
        return {
          title: context.getTitle(),
          path: pathname,
        };
      });

    // ... 渲染面包屑 UI
  },
});
```

利用同样的逻辑，我们还可以为页面的 `<head>` 生成 `title` 标签：

```tsx title="src/routes/__root.tsx"
export const Route = createRootRoute({
  component: () => {
    const matches = useRouterState({ select: (s) => s.matches });

    // 从后往前找，优先取最深层路由定义的标题
    const matchWithTitle = [...matches]
      .reverse()
      .find((d) => d.context.getTitle);

    const title = matchWithTitle?.context.getTitle() || "My App";

    return (
      <html>
        <head>
          <title>{title}</title>
        </head>
        <body>{/* ... */}</body>
      </html>
    );
  },
});
```
