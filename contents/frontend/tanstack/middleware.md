---
short_title: 中间件
---

# 中间件 (Middleware)

## 什么是中间件？

中间件（Middleware）允许你自定义**服务器路由**（如 GET/POST 等，包括应用程序的 SSR 请求）以及通过 `createServerFn` 创建的**服务器函数**的行为。中间件具有可组合性，甚至可以依赖其他中间件，从而创建一个按层级和顺序执行的操作链。

### 我能用中间件做些什么？

- **身份验证 (Authentication)**：在执行服务器函数之前验证用户身份。
- **权限控制 (Authorization)**：检查用户是否拥有执行特定服务器函数所需的权限。
- **日志记录 (Logging)**：记录请求、响应以及错误信息。
- **内容安全策略 (CSP)**：配置内容安全策略和其他安全防护措施。
- **可观测性 (Observability)**：收集指标 (Metrics)、追踪 (Traces) 和日志。
- **提供上下文 (Provide Context)**：将数据附加到请求对象上，供后续中间件或服务器函数使用。
- **错误处理 (Error Handling)**：以一致的方式处理异常情况。
- **以及更多！** 可能性完全取决于你的需求。

# 中间件 (Middleware)

## 中间件类型 (Middleware Types)

中间件分为两种类型：**请求中间件 (Request Middleware)** 和 **服务器函数中间件 (Server Function Middleware)**。

- **请求中间件**：用于自定义通过它的任何服务器请求的行为，包括服务器函数和 SSR 渲染请求。
- **服务器函数中间件**：专门用于自定义服务器函数 (Server Functions) 的行为。

> [!NOTE]
> 服务器函数中间件是请求中间件的一个子集，它具有专门针对服务器函数的额外功能，例如验证输入数据，或在服务器函数执行前后执行客户端逻辑。

### 核心区别 (Key Differences)

| 特性                               | 请求中间件 (Request Middleware) | 服务器函数中间件 (Server Function Middleware) |
| :--------------------------------- | :------------------------------ | :-------------------------------------------- |
| **作用范围 (Scope)**               | 所有服务器请求                  | 仅限服务器函数                                |
| **可用方法 (Methods)**             | `.server()`                     | `.client()`, `.server()`                      |
| **输入验证 (Input Validation)**    | 不支持                          | 支持 (`.inputValidator()`)                    |
| **客户端逻辑 (Client-side Logic)** | 不支持                          | 支持                                          |
| **依赖关系 (Dependencies)**        | 仅能依赖请求中间件              | 可以依赖上述两种类型的中间件                  |

> [!NOTE]
> 请求中间件不能依赖服务器函数中间件，但服务器函数中间件可以依赖请求中间件。

---

## 核心概念 (Core Concepts)

### 中间件组合 (Middleware Composition)

所有中间件都是可组合的，这意味着一个中间件可以依赖于另一个中间件。

```tsx
import { createMiddleware } from "@tanstack/react-start";

const loggingMiddleware = createMiddleware().server(() => {
  //...
});

const authMiddleware = createMiddleware()
  .middleware([loggingMiddleware])
  .server(() => {
    //...
  });
```

### 推进中间件链 (Progressing the Middleware Chain)

中间件是“可继续的 (next-able)”，这意味着你必须在 `.server` 方法（以及 `.client` 方法，如果你在创建服务器函数中间件）中调用 `next` 函数，以执行链中的下一个中间件。这允许你：

- **中断链条**：提前返回 (Short circuit) 以终止后续操作。
- **传递数据**：将数据传递给下一个中间件。
- **拦截结果**：访问并处理下一个中间件返回的结果。
- **传递上下文**：将 context 传递给外层包装的中间件。

```tsx
import { createMiddleware } from "@tanstack/react-start";

const loggingMiddleware = createMiddleware().server(async ({ next }) => {
  const result = await next(); // <-- 这将执行链中的下一个中间件
  return result;
});
```

---

## 请求中间件 (Request Middleware)

请求中间件用于自定义任何通过它的服务器请求，包括服务器路由、SSR 和服务器函数。

要创建请求中间件，请调用 `createMiddleware` 函数。你可以将 `type` 属性设置为 `'request'`，由于这是默认值，你也可以直接省略。

```tsx
import { createMiddleware } from "@tanstack/react-start";

const loggingMiddleware = createMiddleware().server(() => {
  //...
});
```

### 可用方法 (Available Methods)

请求中间件具有以下方法：

- **`middleware`**: 向链中添加其他中间件。
- **`server`**: 定义服务器端逻辑。逻辑会在任何嵌套中间件（及最终的服务器函数）之前执行，并能处理后续中间件的结果。

### `.server` 方法

`.server` 方法接收 `next` 函数以及 `context` 和 `request` 对象：

```tsx
import { createMiddleware } from "@tanstack/react-start";

const loggingMiddleware = createMiddleware().server(
  ({ next, context, request }) => {
    return next();
  },
);
```

其交互流程如下图所示：

---

## 在服务器路由中使用请求中间件 (Using Request Middleware with Server Routes)

你可以通过两种方式在服务器路由中使用请求中间件：

#### 1. 应用于所有服务器路由方法

如果希望整个路由的所有 HTTP 方法都使用中间件，请在路由构建对象的 `middleware` 属性中传入数组：

```tsx
import { createMiddleware } from "@tanstack/react-start";

const loggingMiddleware = createMiddleware().server(() => {
  //...
});

export const Route = createFileRoute("/foo")({
  server: {
    middleware: [loggingMiddleware],
    handlers: {
      GET: () => {
        /* ... */
      },
      POST: () => {
        /* ... */
      },
    },
  },
});
```

#### 2. 应用于特定的服务器路由方法

使用 `createHandlers` 工具，并为特定的方法对象设置 `middleware` 属性：

```tsx
export const Route = createFileRoute("/foo")({
  server: {
    handlers: ({ createHandlers }) =>
      createHandlers({
        GET: {
          middleware: [loggingMiddleware],
          handler: () => {
            //...
          },
        },
      }),
  },
});
```

## 服务器函数中间件 (Server Function Middleware)

服务器函数中间件是请求中间件的一个**子集**，它专门为服务器函数提供了额外的功能，例如验证输入数据，或在服务器函数执行前后执行客户端逻辑。

要创建服务器函数中间件，请在调用 `createMiddleware` 函数时将 `type` 属性设置为 `'function'`。

```tsx
import { createMiddleware } from "@tanstack/react-start";

const loggingMiddleware = createMiddleware({ type: "function" })
  .client(() => {
    //...
  })
  .server(() => {
    //...
  });
```

### 可用方法 (Available Methods)

服务器函数中间件拥有以下方法：

- **`middleware`**: 向链中添加中间件。
- **`inputValidator`**: 在数据对象传递给当前中间件、嵌套中间件以及最终的服务器函数之前，对其进行修改或校验。
- **`client`**: 定义客户端逻辑。中间件将在客户端执行，包裹（及处理）发送到服务器的远程过程调用 (RPC)。
- **`server`**: 定义服务器端逻辑。中间件将在服务器上执行，包裹服务器函数的具体执行过程。

> [!NOTE]
> 如果你使用的是 TypeScript，这些方法的调用顺序由类型系统强制约束，以确保获得最大的类型推断能力和类型安全性。

### `.client` 方法

`.client` 方法用于定义客户端逻辑，该逻辑会包裹对服务器 RPC 调用的执行过程及其结果。

```tsx
import { createMiddleware } from "@tanstack/react-start";

const loggingMiddleware = createMiddleware({ type: "function" }).client(
  async ({ next, context, request }) => {
    // 这将执行链中的下一个中间件，并最终触发指向服务器的 RPC 调用
    const result = await next();
    return result;
  },
);
```

### `.inputValidator` 方法

`inputValidator` 方法用于在数据传递给后续流程前进行修改或验证。它接收一个函数，该函数获取原始数据并返回验证过（且可选修改过）的数据。通常我们会使用 `zod` 等校验库配合使用。

```tsx
import { createMiddleware } from "@tanstack/react-start";
import { zodValidator } from "@tanstack/zod-adapter";
import { z } from "zod";

const mySchema = z.object({
  workspaceId: z.string(),
});

const workspaceMiddleware = createMiddleware({ type: "function" })
  .inputValidator(zodValidator(mySchema))
  .server(({ next, data }) => {
    // 此时 data 的类型已被 mySchema 校验并推断
    console.log("Workspace ID:", data.workspaceId);
    return next();
  });
```

### 使用服务器函数中间件 (Using Server Function Middleware)

若要让中间件包裹特定的服务器函数，只需将中间件数组传递给 `createServerFn` 函数的 `middleware` 属性即可。

```tsx
import { createServerFn } from "@tanstack/react-start";
import { loggingMiddleware } from "./middleware";

const fn = createServerFn()
  .middleware([loggingMiddleware])
  .handler(async () => {
    //...
  });
```

### 交互流程可视化

为了帮助你快速理解这种“握手”过程，请参考以下时序图：

## 上下文管理 (Context Management)

### 通过 `next` 提供上下文

你可以为 `next` 函数提供一个包含 `context` 属性的对象。你传递给 `context` 的任何属性都会被合并到父级 `context` 中，并提供给链中的下一个中间件。

```tsx
import { createMiddleware } from "@tanstack/react-start";

const awesomeMiddleware = createMiddleware({ type: "function" }).server(
  ({ next }) => {
    return next({
      context: {
        // 注入新的上下文属性
        isAwesome: Math.random() > 0.5,
      },
    });
  },
);

const loggingMiddleware = createMiddleware({ type: "function" })
  .middleware([awesomeMiddleware])
  .server(async ({ next, context }) => {
    // 访问上一个中间件注入的 context
    console.log("Is awesome?", context.isAwesome);
    return next();
  });
```

---

### 将客户端上下文发送到服务器

**默认情况下，客户端上下文不会被发送到服务器，这是为了防止无意中向服务器发送巨大的数据负载。** 如果你需要将客户端数据传递给服务器，必须在调用 `next` 时使用 `sendContext` 属性。

传递给 `sendContext` 的属性将被合并、序列化并随数据一起发送到服务器，之后可以在后续任何服务器中间件的 `context` 对象中访问。

```tsx
import { createMiddleware } from "@tanstack/react-start";

const requestLogger = createMiddleware({ type: "function" })
  .client(async ({ next, context }) => {
    return next({
      sendContext: {
        // 将 workspaceId 从客户端传输到服务器
        workspaceId: context.workspaceId,
      },
    });
  })
  .server(async ({ next, data, context }) => {
    // 哇！我们拿到了从客户端传来的 workspaceId
    console.log("Workspace ID:", context.workspaceId);
    return next();
  });
```

#### 客户端发送上下文的安全性

你可能已经注意到，虽然客户端发送的上下文是**类型安全**的，但它在运行时并不强制校验。如果通过上下文传递动态的、用户生成的数据，可能会带来安全隐患。

> [!IMPORTANT]
> **如果你正通过上下文从客户端向服务器发送动态数据，在使用这些数据之前，务必在服务器端中间件中进行校验。**

```tsx
import { createMiddleware } from "@tanstack/react-start";
import { zodValidator } from "@tanstack/zod-adapter";
import { z } from "zod";

const requestLogger = createMiddleware({ type: "function" })
  .client(async ({ next, context }) => {
    return next({
      sendContext: {
        workspaceId: context.workspaceId,
      },
    });
  })
  .server(async ({ next, data, context }) => {
    // 在使用前校验 workspaceId 的合法性
    const workspaceId = zodValidator(z.number()).parse(context.workspaceId);
    console.log("Validated Workspace ID:", workspaceId);
    return next();
  });
```

---

### 将服务器上下文发送到客户端

与发送到服务器类似，你也可以通过在服务器端的 `next` 函数中使用 `sendContext` 属性将数据传回客户端。这些数据会随响应一起被序列化，并可在后续任何客户端中间件的 `context` 对象中获取。

> [!WARNING]
> `client` 中 `next` 的返回类型只能从当前已知的中间件链中推断。因此，在中间件链末尾的中间件中，`next` 的返回类型推断是最准确的。

```tsx
import { createMiddleware } from "@tanstack/react-start";

const serverTimer = createMiddleware({ type: "function" }).server(
  async ({ next }) => {
    return next({
      sendContext: {
        // 将服务器当前时间发送给客户端
        timeFromServer: new Date(),
      },
    });
  },
);

const requestLogger = createMiddleware({ type: "function" })
  .middleware([serverTimer])
  .client(async ({ next }) => {
    const result = await next();
    // 访问来自服务器的数据
    console.log("Time from the server:", result.context.timeFromServer);

    return result;
  });
```

## 全局中间件 (Global Middleware)

全局中间件会自动为应用程序中的每个请求运行。这对于身份验证、日志记录和监控等需要应用于所有请求的功能非常有用。

> [!NOTE]
> 默认的 TanStack Start 模板中不包含 `src/start.ts` 文件。当你需要配置全局中间件或其他 Start 级别的选项时，需要手动创建此文件。

### 全局请求中间件 (Global Request Middleware)

若要让某个中间件在 **Start 处理的每一个请求** 中运行，请创建 `src/start.ts` 文件，并使用 `createStart` 函数返回你的中间件配置：

```tsx
// src/start.ts
import { createStart, createMiddleware } from "@tanstack/react-start";

const myGlobalMiddleware = createMiddleware().server(() => {
  // 执行全局逻辑...
});

export const startInstance = createStart(() => {
  return {
    // 注册全局请求中间件
    requestMiddleware: [myGlobalMiddleware],
  };
});
```

> [!NOTE]
> 全局**请求**中间件会在**每一个请求之前运行，包括服务器路由、SSR（服务端渲染）和服务器函数**。

---

### 全局服务器函数中间件 (Global Server Function Middleware)

若要让某个中间件在**应用程序中的每一个服务器函数**中运行，请将其添加到 `src/start.ts` 文件中的 `functionMiddleware` 数组里：

```tsx
// src/start.ts
import { createStart } from "@tanstack/react-start";
import { loggingMiddleware } from "./middleware";

export const startInstance = createStart(() => {
  return {
    // 注册全局服务器函数中间件
    functionMiddleware: [loggingMiddleware],
  };
});
```

---

### 中间件执行顺序 (Middleware Execution Order)

中间件按照“依赖优先”的原则执行：从全局中间件开始，随后是服务器函数中间件，最后按照依赖链执行。

以下示例的日志打印顺序将是：

1. `globalMiddleware1`
2. `globalMiddleware2`
3. `a`
4. `b`
5. `c`
6. `d`
7. `fn`

```tsx
import { createMiddleware, createServerFn } from "@tanstack/react-start";

const globalMiddleware1 = createMiddleware({ type: "function" }).server(
  async ({ next }) => {
    console.log("globalMiddleware1");
    return next();
  },
);

const globalMiddleware2 = createMiddleware({ type: "function" }).server(
  async ({ next }) => {
    console.log("globalMiddleware2");
    return next();
  },
);

const a = createMiddleware({ type: "function" }).server(async ({ next }) => {
  console.log("a");
  return next();
});

const b = createMiddleware({ type: "function" })
  .middleware([a])
  .server(async ({ next }) => {
    console.log("b");
    return next();
  });

const c = createMiddleware({ type: "function" })
  .middleware()
  .server(async ({ next }) => {
    console.log("c");
    return next();
  });

const d = createMiddleware({ type: "function" })
  .middleware([b, c])
  .server(async ({ next }) => {
    console.log("d");
    return next();
  });

const fn = createServerFn()
  .middleware([d])
  .handler(async () => {
    console.log("fn");
  });
```

## 请求与响应修改 (Request and Response Modification)

### 读取/修改服务器响应 (Reading/Modifying the Server Response)

使用 `.server` 方法的中间件与服务器函数运行在相同的上下文中。因此，你可以遵循完全相同的 [服务器函数上下文工具 (Server Function Context Utilities)](./server-functions.md#server-function-context) 来读取和修改请求头、状态码等任何内容。

---

### 修改客户端请求 (Modifying the Client Request)

使用 `.client` 方法的中间件运行在与服务器函数**完全不同的客户端上下文**中，因此你不能使用相同的工具函数来读取和修改请求。不过，你仍然可以通过在调用 `next` 函数时返回额外的属性来修改请求。

#### 设置自定义请求头 (Setting Custom Headers)

你可以通过向 `next` 函数传递 `headers` 对象来为发出的请求添加 Header：

```tsx
import { createMiddleware } from "@tanstack/react-start";
import { getToken } from "my-auth-library";

const authMiddleware = createMiddleware({ type: "function" }).client(
  async ({ next }) => {
    return next({
      headers: {
        // 在客户端请求中注入授权信息
        Authorization: `Bearer ${getToken()}`,
      },
    });
  },
);
```

#### 中间件间的 Header 合并 (Header Merging Across Middleware)

当多个中间件都设置了 Header 时，它们会进行**合并**。后执行的中间件可以添加新的 Header，或者覆盖先执行中间件设置的 Header：

```tsx
import { createMiddleware } from "@tanstack/react-start";

const firstMiddleware = createMiddleware({ type: "function" }).client(
  async ({ next }) => {
    return next({
      headers: {
        "X-Request-ID": "12345",
        "X-Source": "first-middleware",
      },
    });
  },
);

const secondMiddleware = createMiddleware({ type: "function" }).client(
  async ({ next }) => {
    return next({
      headers: {
        "X-Timestamp": Date.now().toString(),
        "X-Source": "second-middleware", // 将覆盖第一个中间件设置的值
      },
    });
  },
);

// 最终发出的 Headers 将包含：
// - X-Request-ID: '12345' (来自第一个中间件)
// - X-Timestamp: '<时间戳>' (来自第二个中间件)
// - X-Source: 'second-middleware' (第二个中间件覆盖了第一个)
```

你也可以在调用处直接设置 Header：

```tsx
await myServerFn({
  data: { name: "John" },
  headers: {
    "X-Custom-Header": "call-site-value",
  },
});
```

**Header 优先级（所有 Header 都会合并，后定义的值覆盖先定义的）：**

1. 较早执行的中间件 Header
2. 较晚执行的中间件 Header（覆盖前者）
3. 调用处（Call-site）定义的 Header（覆盖所有中间件的 Header）

#### 自定义 Fetch 实现 (Custom Fetch Implementation)

对于高级用例，你可以提供自定义的 `fetch` 实现来控制服务器函数的请求方式。这在以下场景非常有用：

- 添加请求拦截器或重试逻辑
- 使用自定义 HTTP 客户端
- 进行测试和 Mock 模拟
- 添加遥测 (Telemetry) 或监控

**通过客户端中间件实现：**

```tsx
import { createMiddleware } from "@tanstack/react-start";
import type { CustomFetch } from "@tanstack/react-start";

const customFetchMiddleware = createMiddleware({ type: "function" }).client(
  async ({ next }) => {
    const customFetch: CustomFetch = async (url, init) => {
      console.log("请求开始:", url);
      const start = Date.now();

      const response = await fetch(url, init);

      console.log("请求完成，耗时", Date.now() - start, "ms");
      return response;
    };

    return next({ fetch: customFetch });
  },
);
```

**直接在调用处 (Call Site) 实现：**

```tsx
import type { CustomFetch } from "@tanstack/react-start";

const myFetch: CustomFetch = async (url, init) => {
  // 在此处添加自定义逻辑
  return fetch(url, init);
};

await myServerFn({
  data: { name: "John" },
  fetch: myFetch,
});
```

#### Fetch 覆盖优先级 (Fetch Override Precedence)

当在多个层级提供自定义 fetch 实现时，应用以下优先级（从高到低）：

| 优先级   | 来源                   | 说明                                                 |
| :------- | :--------------------- | :--------------------------------------------------- |
| 1 (最高) | **调用处 (Call site)** | `serverFn({ fetch: customFetch })`                   |
| 2        | **较晚的中间件**       | 中间件链中最后一个提供 `fetch` 的中间件              |
| 3        | **较早的中间件**       | 中间件链中第一个提供 `fetch` 的中间件                |
| 4        | **createStart**        | `createStart({ serverFns: { fetch: customFetch } })` |
| 5 (最低) | **默认**               | 全局 `fetch` 函数                                    |

**核心原则：** 调用处永远拥有最高优先级。这允许你在需要时为特定的调用覆盖中间件的行为。

```tsx
import { createMiddleware, createServerFn } from "@tanstack/react-start";
import type { CustomFetch } from "@tanstack/react-start";

// 中间件设置了一个带有日志记录的 fetch
const loggingMiddleware = createMiddleware({ type: "function" }).client(
  async ({ next }) => {
    const loggingFetch: CustomFetch = async (url, init) => {
      console.log("中间件 fetch:", url);
      return fetch(url, init);
    };
    return next({ fetch: loggingFetch });
  },
);

const myServerFn = createServerFn()
  .middleware([loggingMiddleware])
  .handler(async () => {
    return { message: "Hello" };
  });

// 情况 A: 使用中间件的 loggingFetch
await myServerFn();

// 情况 B: 为本次特定调用覆盖自定义 fetch
const testFetch: CustomFetch = async (url, init) => {
  console.log("测试专用 fetch:", url);
  return fetch(url, init);
};
await myServerFn({ fetch: testFetch }); // 使用 testFetch，而不是 loggingFetch
```

**中间件链示例：**

当多个中间件提供 fetch 时，最后一个中间件胜出：

```tsx
const firstMiddleware = createMiddleware({ type: "function" }).client(
  async ({ next }) => {
    const firstFetch: CustomFetch = (url, init) => {
      const headers = new Headers(init?.headers);
      headers.set("X-From", "first-middleware");
      return fetch(url, { ...init, headers });
    };
    return next({ fetch: firstFetch });
  },
);

const secondMiddleware = createMiddleware({ type: "function" }).client(
  async ({ next }) => {
    const secondFetch: CustomFetch = (url, init) => {
      const headers = new Headers(init?.headers);
      headers.set("X-From", "second-middleware");
      return fetch(url, { ...init, headers });
    };
    return next({ fetch: secondFetch });
  },
);

const myServerFn = createServerFn()
  .middleware([firstMiddleware, secondMiddleware])
  .handler(async () => {
    // 请求将带有 X-From: 'second-middleware'
    // 因为 secondMiddleware 的 fetch 覆盖了 firstMiddleware 的 fetch
    return { message: "Hello" };
  });
```

**通过 createStart 配置全局 Fetch：**

你可以通过在 `createStart` 的 `serverFns.fetch` 中提供自定义实现，为应用中的所有服务器函数设置默认 fetch。这对于添加全局请求拦截器、重试逻辑或遥测非常有用：

```tsx
// src/start.ts
import { createStart } from "@tanstack/react-start";
import type { CustomFetch } from "@tanstack/react-start";

const globalFetch: CustomFetch = async (url, init) => {
  console.log("全局 fetch:", url);
  // 添加重试逻辑、遥测等...
  return fetch(url, init);
};

export const startInstance = createStart(() => {
  return {
    serverFns: {
      fetch: globalFetch,
    },
  };
});
```

> [!NOTE]
> 自定义 fetch 仅适用于客户端。在 SSR（服务端渲染）期间，服务器函数是直接调用的，不会经过 fetch 过程。

## 环境与性能 (Environment and Performance)

### 环境 Tree Shaking (Environment Tree Shaking)

中间件功能会根据每个生成的 Bundle 环境自动进行 Tree Shaking（摇树优化）。

- **在服务器端 (Server)**：不会进行任何 Tree Shaking，因此中间件中使用的所有代码都会被包含在服务器 bundle 中。
- **在客户端 (Client)**：所有服务器特定的代码都会从客户端 bundle 中移除。这意味着在 `.server` 方法中使用的任何代码始终会被移除。此外，`data` 验证逻辑相关的代码也会被移除。

> [!TIP]
> 这种机制确保了敏感的服务器逻辑（如数据库查询、秘钥处理等）永远不会泄露到客户端，同时也减小了用户需要下载的 JavaScript 资源体积。
