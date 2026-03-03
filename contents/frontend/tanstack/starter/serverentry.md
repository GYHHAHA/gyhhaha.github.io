---
short_title: 服务器入口点
---

# 服务器入口点 (Server Entry Point)

> [!NOTE]
> 服务器入口点在开箱即用的情况下是**可选的**。如果未手动提供，TanStack Start 将默认使用下文所述的逻辑为你自动处理。

服务器入口点支持通用 `fetch` 处理程序格式，这种格式常用于 [Cloudflare Workers](https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/) 以及其他兼容 WinterCG 规范的运行时。

为了确保互操作性，默认导出必须符合我们的 `ServerEntry` 接口规范：

```ts
export default {
  fetch(req: Request, opts?: RequestOptions): Response | Promise<Response> {
    // ...
  },
};
```

TanStack Start 提供了一个包装函数来确保创建过程的类型安全。这通常在 `src/server.ts` 文件中完成：

```tsx
// src/server.ts
import handler, { createServerEntry } from "@tanstack/react-start/server-entry";

export default createServerEntry({
  fetch(request) {
    // 将请求转发给内置的处理程序
    return handler.fetch(request);
  },
});
```

无论是静态生成 (SSG) 应用还是动态提供服务的应用，`server.ts` 文件都是执行所有 SSR 相关工作以及处理服务器路由和服务器函数请求的入口。

---

## 自定义服务器处理程序 (Custom Server Handlers)

你可以创建自定义服务器处理程序来修改应用程序的渲染方式：

```tsx
// src/server.ts
import {
  createStartHandler,
  defaultStreamHandler,
  defineHandlerCallback,
} from "@tanstack/react-start/server";
import { createServerEntry } from "@tanstack/react-start/server-entry";

// 定义一个自定义处理回调
const customHandler = defineHandlerCallback((ctx) => {
  // 在此处添加自定义逻辑（例如注入全局变量或修改渲染上下文）
  return defaultStreamHandler(ctx);
});

const fetch = createStartHandler(customHandler);

export default createServerEntry({
  fetch,
});
```

---

## 请求上下文 (Request Context)

当服务器需要将额外的类型化数据传入请求处理程序时（例如已验证的用户信息、数据库连接或每个请求的标识），可以通过 TypeScript 模块扩充（Module Augmentation）注册请求上下文类型。

注册后的上下文将作为第二个参数传递给服务器 `fetch` 处理程序，并可在整个服务端中间件链中调用 —— 包括全局中间件、请求/函数中间件、服务器路由、服务器函数以及路由器本身。

要为请求上下文添加类型，请扩充 `@tanstack/react-start` 中的 `Register` 接口，并添加 `server.requestContext` 属性。你传递给 `handler.fetch` 的运行时 `context` 将会匹配该类型。

**示例：**

```tsx
import handler, { createServerEntry } from "@tanstack/react-start/server-entry";

type MyRequestContext = {
  hello: string;
  foo: number;
};

declare module "@tanstack/react-start" {
  interface Register {
    server: {
      requestContext: MyRequestContext;
    };
  }
}

export default createServerEntry({
  async fetch(request) {
    // 手动传入 context
    return handler.fetch(request, {
      context: { hello: "world", foo: 123 },
    });
  },
});
```

---

## 服务器配置

服务器入口点是你配置服务端特定行为的核心位置，包括：

- **中间件**：请求/响应拦截处理。
- **自定义错误处理**：捕获 SSR 期间的异常。
- **身份验证逻辑**：在请求到达路由前校验 Session 或 Token。
- **数据库连接**：管理服务端连接池。
- **日志与监控**：集成服务端可观测性工具。

这种灵活性允许你在保持框架约定的同时，根据具体需求深度定制服务器端行为。

---

## Cloudflare Workers

在部署到 Cloudflare Workers 时，你可以扩展 `server.ts` 以处理额外的 Workers 特性，例如队列 (Queues)、定时任务 (Scheduled Events) 以及持久对象 (Durable Objects)。

详细指南请参考 [TanStack Start 的 Cloudflare Workers 官方文档](https://developers.cloudflare.com/workers/framework-guides/web-apps/tanstack-start/#custom-entrypoints)。
