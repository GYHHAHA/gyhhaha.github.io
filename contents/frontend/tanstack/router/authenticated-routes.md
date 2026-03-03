---
short_title: 身份验证路由
---

# 身份验证路由 (Authenticated Routes)

身份验证是 Web 应用程序中极其常见的需求。在本指南中，我们将介绍如何使用 TanStack Router 构建受保护的路由，以及如果用户尝试访问这些路由，如何将其重定向到登录页面。

## `route.beforeLoad` 选项

`route.beforeLoad` 选项允许你指定一个在加载路由之前调用的函数。它接收与 `route.loader` 函数相同的参数。这是检查用户是否已通过身份验证，并在未验证时将其重定向到登录页面的理想场所。

`beforeLoad` 函数相对于其他路由加载函数的执行顺序如下：

- **路由匹配 (Route Matching) - 自上而下**
  - `route.params.parse`
  - `route.validateSearch`
- **路由加载 (Route Loading) - 包含预加载**
  - **`route.beforeLoad`**
  - `route.onError`
- **路由加载 (Route Loading) - 并行执行**
  - `route.component.preload?`
  - `route.load`

**重要提示：某个路由的 `beforeLoad` 函数会在其任何子路由的 `beforeLoad` 函数之前被调用。** 它本质上是该路由及其所有子路由的中间件函数。

**如果你在 `beforeLoad` 中抛出错误，它的任何子路由都不会尝试加载。**

## 重定向 (Redirecting)

虽然不是强制要求的，但某些身份验证流程需要重定向到登录页面。为此，你可以在 `beforeLoad` 中执行 **`throw redirect()`**：

```tsx
// src/routes/_authenticated.tsx
export const Route = createFileRoute("/_authenticated")({
  beforeLoad: async ({ location }) => {
    if (!isAuthenticated()) {
      throw redirect({
        to: "/login",
        search: {
          // 使用当前位置信息，以便在登录后进行重定向
          // (不要使用 `router.state.resolvedLocation`，因为它可能
          // 滞后于实际的当前位置)
          redirect: location.href,
        },
      });
    }
  },
});
```

> [!TIP]
> `redirect()` 函数接受与 `Maps` 函数完全相同的选项，因此如果你想替换当前历史记录条目而不是添加新条目，可以传递 `replace: true` 等选项。

### 处理身份验证检查失败

如果你的身份验证检查可能会抛出错误（网络故障、令牌验证等），请将其包裹在 try/catch 中：

```tsx
import { createFileRoute, redirect, isRedirect } from "@tanstack/react-router";

// src/routes/_authenticated.tsx
export const Route = createFileRoute("/_authenticated")({
  beforeLoad: async ({ location }) => {
    try {
      const user = await verifySession(); // 可能会因为网络错误抛出异常
      if (!user) {
        throw redirect({
          to: "/login",
          search: { redirect: location.href },
        });
      }
      return { user };
    } catch (error) {
      // 重新抛出重定向（它们是故意的操作，而不是错误）
      if (isRedirect(error)) throw error;

      // 验证检查失败（网络错误等）- 重定向到登录页
      throw redirect({
        to: "/login",
        search: { redirect: location.href },
      });
    }
  },
});
```

[`isRedirect()`](../api/router/isRedirectFunction.md) 辅助函数可以区分实际错误和故意的重定向。

一旦用户通过身份验证，通常的做法是将他们重定向回原本尝试访问的页面。你可以利用我们在最初重定向时添加的 `redirect` 查询参数。由于我们要用原来的完整 URL 替换当前地址，使用 `router.history.push` 比 `router.navigate` 更合适：

```tsx
router.history.push(search.redirect);
```

## 非重定向式身份验证

有些应用程序选择不将用户重定向到登录页面，而是让用户留在当前页面，并显示一个登录表单，该表单要么替换主体内容，要么通过模态框（Modal）隐藏内容。这在 TanStack Router 中也可以实现，只需短路渲染原本会渲染子路由的 `<Outlet />`：

```tsx
// src/routes/_authenticated.tsx
export const Route = createFileRoute("/_authenticated")({
  component: () => {
    if (!isAuthenticated()) {
      return <Login />;
    }

    return <Outlet />;
  },
});
```

这种做法能让用户保持在同一页面，同时依然允许你渲染登录表单。一旦用户通过验证，你只需渲染 `<Outlet />`，子路由就会被呈现出来。

## 使用 React Context/Hooks 进行身份验证

如果你的身份验证流程依赖于 React Context 和/或 Hooks 的交互，你需要使用 `router.context` 选项将身份验证状态传递给 TanStack Router。

> [!IMPORTANT]
> React Hooks 不能在 React 组件之外使用。如果你需要在 React 组件之外使用 Hook，你需要在包装了 `<RouterProvider />` 的组件中提取 Hook 返回的状态，然后将该值向下传递给 TanStack Router。

我们将在 [路由上下文 (Router Context)](./router-context.md) 部分详细介绍 `router.context` 选项。

以下是一个在 TanStack Router 中使用 React Context 和 Hooks 来保护验证路由的示例。你可以在 [Authenticated Routes 示例](https://github.com/TanStack/router/tree/main/examples/react/authenticated-routes) 中查看完整的运行设置。

```tsx title="src/routes/__root.tsx"
import { createRootRouteWithContext } from "@tanstack/react-router";

interface MyRouterContext {
  // 你的 useAuth Hook 的返回类型或 AuthContext 的值
  auth: AuthState;
}

export const Route = createRootRouteWithContext<MyRouterContext>()({
  component: () => <Outlet />,
});
```

```tsx title="src/router.tsx"
import { createRouter } from "@tanstack/react-router";
import { routeTree } from "./routeTree.gen";

export const router = createRouter({
  routeTree,
  context: {
    // auth 最初将是 undefined
    // 我们将从 React 组件内部向下传递 auth 状态
    auth: undefined!,
  },
});
```

```tsx title="src/App.tsx"
import { RouterProvider } from "@tanstack/react-router";
import { AuthProvider, useAuth } from "./auth";
import { router } from "./router";

function InnerApp() {
  const auth = useAuth();
  // 将 Hook 返回的值注入到路由上下文中
  return <RouterProvider router={router} context={{ auth }} />;
}

function App() {
  return (
    <AuthProvider>
      <InnerApp />
    </AuthProvider>
  );
}
```

然后在受保护的路由中，你可以使用 `beforeLoad` 函数检查验证状态，如果用户未登录，则执行 **`throw redirect()`** 跳转到 **Login 路由**。

```tsx title="src/routes/dashboard.route.tsx"
import { createFileRoute, redirect } from "@tanstack/react-router";

export const Route = createFileRoute("/dashboard")({
  beforeLoad: ({ context, location }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({
        to: "/login",
        search: {
          redirect: location.href,
        },
      });
    }
  },
});
```

你也可以*可选地*使用 [非重定向式身份验证](非重定向式身份验证) 方案来显示登录表单，而不是调用 **redirect**。

这种方法也可以与“无路径路由 (Pathless Route)”或“布局路由 (Layout Route)”结合使用，以保护其父路由下的所有子路由。

## 相关操作指南

有关详细的分步实现指南，请参阅：

- [如何设置基础身份验证](../how-to/setup-authentication.md) - 使用 React Context 和受保护路由的完整设置。
- [如何集成身份验证提供商](../how-to/setup-auth-providers.md) - 使用 Auth0、Clerk 或 Supabase。
- [如何设置基于角色的访问控制 (RBAC)](../how-to/setup-rbac.md) - 实现权限和基于角色的路由。

## 示例

代码库中提供了可运行的身份验证示例：

- [基础身份验证示例](https://github.com/TanStack/router/tree/main/examples/react/authenticated-routes) - 使用 Context 的简单身份验证。
- [Firebase 身份验证](https://github.com/TanStack/router/tree/main/examples/react/authenticated-routes-firebase) - Firebase Auth 集成。
- [TanStack Start 身份验证示例](https://github.com/TanStack/router/tree/main/examples/react) - 使用 TanStack Start 的各种身份验证实现。
