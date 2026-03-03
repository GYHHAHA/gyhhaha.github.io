---
short_title: 静态路由数据
---

# 静态路由数据 (Static Route Data)

在创建路由时，你可以选择在路由选项中指定一个 `staticData` 属性。只要该对象在创建路由时是同步可用的，它几乎可以包含任何你想要的内容。

除了可以从路由本身访问这些数据外，你还可以通过任何路由匹配项（match）的 `match.staticData` 属性来访问它。

## 示例

```tsx title='src/routes/posts.tsx'
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/posts")({
  staticData: {
    customData: "你好！",
  },
});
```

之后，你可以在任何能访问到路由的地方使用这些数据，包括可以映射回其路由的匹配项（matches）。

```tsx title='src/routes/__root.tsx'
import { createRootRoute } from "@tanstack/react-router";

export const Route = createRootRoute({
  component: () => {
    const matches = useMatches();

    return (
      <div>
        {matches.map((match) => {
          return <div key={match.id}>{match.staticData.customData}</div>;
        })}
      </div>
    );
  },
});
```

## 强制执行静态数据类型

如果你想强制要求路由必须包含某些静态数据，可以使用“声明合并（Declaration Merging）”来为路由的静态选项添加类型定义：

```tsx
declare module "@tanstack/react-router" {
  interface StaticDataRouteOption {
    customData: string;
  }
}
```

现在，如果你尝试创建一个没有 `customData` 属性的路由，将会收到类型错误：

```tsx
export const Route = createFileRoute("/posts")({
  staticData: {
    // 错误：类型 '{ ... }' 中缺少属性 'customData'，但类型 'StaticDataRouteOption' 中需要该属性。ts(2741)
  },
});
```

## 可选的静态数据

如果你希望静态数据是可选的，只需在属性后面加上 `?`：

```tsx
declare module "@tanstack/react-router" {
  interface StaticDataRouteOption {
    customData?: string;
  }
}
```

只要 `StaticDataRouteOption` 中存在任何必选属性，你就必须传入一个对象。

## 常见模式

### 控制布局可见性

使用 `staticData` 来控制哪些路由显示或隐藏特定的布局元素：

```tsx title='src/routes/admin/route.tsx'
export const Route = createFileRoute("/admin")({
  staticData: { showNavbar: false },
  component: AdminLayout,
});
```

```tsx
// routes/__root.tsx
function RootComponent() {
  const showNavbar = useMatches({
    select: (matches) =>
      // 如果没有任何一个匹配的路由明确设置 showNavbar 为 false，则显示导航栏
      !matches.some((m) => m.staticData?.showNavbar === false),
  });

  return showNavbar ? (
    <Navbar>
      <Outlet />
    </Navbar>
  ) : (
    <Outlet />
  );
}
```

### 用于面包屑的路由标题

```tsx title='src/routes/posts/$postId.tsx'
export const Route = createFileRoute("/posts/$postId")({
  staticData: {
    getTitle: () => "文章详情",
  },
});
```

```tsx title='src/components/Breadcrumbs.tsx'
function Breadcrumbs() {
  const matches = useMatches();

  return (
    <nav>
      {matches
        .filter((m) => m.staticData?.getTitle)
        .map((m) => (
          <span key={m.id}>{m.staticData.getTitle()}</span>
        ))}
    </nav>
  );
}
```

### 什么时候使用 staticData 而不是 Context？

| 特性       | staticData                   | context                           |
| :--------- | :--------------------------- | :-------------------------------- |
| **时机**   | 同步，在路由创建时定义       | 可以是异步的（通过 `beforeLoad`） |
| **可用性** | 在加载开始前即可用           | 可以依赖于 params/search 参数     |
| **共享性** | 对于同一路由的所有实例都相同 | 沿着路由树向下传递给子路由        |

**总结：** 对于静态的路由元数据（Metadata），请使用 `staticData`。对于动态数据或随请求变化的身份验证状态（Auth State），请使用 `context`。
