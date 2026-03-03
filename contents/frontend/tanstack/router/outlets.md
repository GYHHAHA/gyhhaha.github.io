---
short_title: 占位组件
---

# 占位组件 (Outlets)

嵌套路由意味着路由可以嵌套在其他路由中，包括它们的渲染方式。那么，我们该如何告诉路由在哪里渲染这些嵌套的内容呢？

## `Outlet` 组件

`Outlet` 组件用于渲染下一个可能匹配的子路由。`<Outlet />` 不接受任何 Props，并且可以渲染在路由组件树中的任何位置。如果没有匹配的子路由，`<Outlet />` 将渲染为 `null`。

> [!TIP]
> 如果一个路由的 `component` 未定义，它将默认自动渲染一个 `<Outlet />`。

一个典型的例子是配置应用程序的根路由。让我们给根路由一个组件，它渲染一个标题，然后通过 `<Outlet />` 让顶级路由进行渲染。

```tsx
import { createRootRoute, Outlet } from "@tanstack/react-router";

export const Route = createRootRoute({
  component: RootComponent,
});

function RootComponent() {
  return (
    <div>
      <h1>我的应用</h1>
      <Outlet /> {/* 这里是子路由将要渲染的地方 */}
    </div>
  );
}
```
