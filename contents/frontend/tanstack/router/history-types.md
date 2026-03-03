---
short_title: 历史类型
---

# 历史记录类型 (History Types)

虽然使用 TanStack Router 并不要求你必须掌握 `@tanstack/history` 自身的 API，但了解其工作原理是个好主意。在底层，TanStack Router 需要并使用了一个 `history` 抽象层来管理路由历史记录。

如果你没有手动创建 history 实例，路由器在初始化时会自动为你创建一个面向浏览器的实例。如果你需要特定的历史记录 API 类型，可以使用 `@tanstack/history` 包来创建你自己的实例：

- `createBrowserHistory`：默认的历史记录类型。
- `createHashHistory`：使用 URL 哈希（#）来追踪历史记录的类型。
- `createMemoryHistory`：将历史记录保留在内存中的类型。

一旦你拥有了 history 实例，就可以将其传递给 `Router` 的构造函数：

```ts
import { createMemoryHistory, createRouter } from "@tanstack/react-router";

const memoryHistory = createMemoryHistory({
  initialEntries: ["/"], // 传入你的初始 URL
});

const router = createRouter({ routeTree, history: memoryHistory });
```

## 浏览器路由 (Browser Routing)

`createBrowserHistory` 是默认的历史记录类型。它利用浏览器原生的 History API 来管理浏览器历史记录。

## 哈希路由 (Hash Routing)

如果你的服务器不支持将 HTTP 请求重写（Rewrite）到 `index.html`（或者在其他没有服务器的环境中），哈希路由会非常有用。

```ts
import { createHashHistory, createRouter } from "@tanstack/react-router";

const hashHistory = createHashHistory();

const router = createRouter({ routeTree, history: hashHistory });
```

## 内存路由 (Memory Routing)

内存路由在非浏览器环境，或者当你不想让组件与 URL 进行交互时非常有用。

```ts
import { createMemoryHistory, createRouter } from "@tanstack/react-router";

const memoryHistory = createMemoryHistory({
  initialEntries: ["/"], // 传入你的初始 URL
});

const router = createRouter({ routeTree, history: memoryHistory });
```

关于在服务器端进行服务端渲染（SSR）时的用法，请参考 [SSR 指南](./ssr.md#automatic-server-history)。
