---
short_title: 创建路由器
---

# 创建路由器 (Creating a Router)

## `createRouter` 函数

当您准备好开始使用路由器时，需要创建一个新的 `Router` 实例。路由器实例是 TanStack Router 的核心大脑，负责管理路由树、匹配路由以及协调导航和路由切换。它也是配置全局路由器设置的地方。

```tsx title="src/router.tsx"
import { createRouter } from "@tanstack/react-router";

const router = createRouter({
  // ... 配置项
});
```

## 路由树 (Route Tree)

您可能很快就会注意到，`Router` 构造函数需要一个 `routeTree` 选项。这是路由器用于匹配路由和渲染组件的路由树。

无论您使用的是 [基于文件的路由](../routing/file-based-routing.md) 还是 [基于代码的路由](../routing/code-based-routing.md)，都需要将路由树传递给 `createRouter` 函数：

### 文件系统路由树 (Filesystem Route Tree)

如果您使用了我们推荐的基于文件的路由，那么生成的路由树文件很可能位于默认的 `src/routeTree.gen.ts` 位置。如果您使用了自定义位置，则需要从该位置导入路由树。

```tsx
import { routeTree } from "./routeTree.gen";
```

### 基于代码的路由树 (Code-Based Route Tree)

如果您使用的是基于代码的路由，那么您可能是通过根路由的 `addChildren` 方法手动创建路由树的：

```tsx
const routeTree = rootRoute.addChildren([
  // ... 子路由
]);
```

---

## 路由器类型安全 (Router Type Safety)

> [!IMPORTANT]
> 请勿跳过此章节！ ⚠️

TanStack Router 对 TypeScript 提供了惊人的支持，甚至包括直接从库中进行裸导入（bare imports）等您意想不到的地方！为了实现这一点，您必须使用 TypeScript 的 [声明合并 (Declaration Merging)](https://www.typescriptlang.org/docs/handbook/declaration-merging.html) 功能来注册路由器的类型。具体做法是扩展 `@tanstack/react-router` 上的 `Register` 接口，并添加一个类型为您 `router` 实例的 `router` 属性：

```tsx title="src/router.tsx"
declare module "@tanstack/react-router" {
  interface Register {
    // 这将推断我们路由器的类型并将其注册到整个项目中
    router: typeof router;
  }
}
```

完成路由器注册后，您将在整个项目中获得与路由相关的任何内容的类型安全。

---

## 404 未找到路由 (404 Not Found Route)

正如在之前的指南中承诺的那样，我们现在来讲解 `notFoundRoute` 选项。该选项用于配置一个在找不到其他合适匹配项时渲染的路由。这对于渲染 404 页面或重定向到默认路由非常有用。

无论您使用的是基于文件的路由还是基于代码的路由，都需要在 `createRootRoute` 中添加 `notFoundComponent` 键：

```tsx
export const Route = createRootRoute({
  component: () => (
    // ... 主布局组件
  ),
  notFoundComponent: () => <div>404 页面未找到</div>,
});
```

## 其他选项 (Other Options)

`Router` 构造函数还可以接受许多其他选项。您可以在 [API 参考](../api/router/RouterOptionsType.md) 中找到它们的完整列表。
