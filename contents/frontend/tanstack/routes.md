---
short_title: 服务器路由
---

# 服务器路由 (Server Routes)

服务器路由是 TanStack Start 的一项强大功能，它允许你在应用程序中创建服务端端点。这对于处理原始 HTTP 请求、表单提交、用户身份验证等场景非常有用。

服务器路由可以定义在项目的 `./src/routes` 目录中，**与你的 TanStack Router 路由并排存放**，并由 TanStack Start 服务器自动处理。

以下是一个简单的服务器路由示例：

```ts
// routes/hello.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/hello")({
  server: {
    handlers: {
      GET: async ({ request }) => {
        return new Response("Hello, World!");
      },
    },
  },
});
```

---

## 服务器路由与应用路由共存

由于服务器路由可以与应用路由定义在同一个目录中，你甚至可以在同一个文件中同时编写两者！

```tsx
// routes/hello.tsx
import { useState } from "react";
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/hello")({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const body = await request.json();
        return new Response(JSON.stringify({ message: `你好, ${body.name}!` }));
      },
    },
  },
  component: HelloComponent,
});

function HelloComponent() {
  const [reply, setReply] = useState("");

  return (
    <div>
      <button
        className="px-4 py-2 bg-blue-500 text-white rounded"
        onClick={() => {
          fetch("/hello", {
            method: "POST",
            headers: {
              "Content-Type": "application/json",
            },
            body: JSON.stringify({ name: "Tanner" }),
          })
            .then((res) => res.json())
            .then((data) => setReply(data.message));
        }}
      >
        打个招呼
      </button>
      {reply && <p>{reply}</p>}
    </div>
  );
}
```

---

## 文件路由约定

TanStack Start 中的服务器路由遵循与 TanStack Router 相同的基于文件的路由约定。这意味着 `routes` 目录下的任何文件中，只要在 `createFileRoute` 调用中包含 `server` 属性，就会被视为 API 路由。以下是一些示例：

- `/routes/users.ts` -> 创建 API 路由 `/users`
- `/routes/users.index.ts` -> **同样**创建 API 路由 `/users`（但如果定义了重复的方法会报错）
- `/routes/users/$id.ts` -> 创建 API 路由 `/users/$id`
- `/routes/users/$id/posts.ts` -> 创建 API 路由 `/users/$id/posts`
- `/routes/api/file/$.ts` -> 创建 API 路由 `/api/file/$`（通配符路由）
- `/routes/my-script[.]js.ts` -> 创建 API 路由 `/my-script.js`（转义匹配）

---

## 唯一路由路径

每个路由路径只能有一个关联的处理程序文件。如果你有一个名为 `routes/users.ts` 的文件（对应路径 `/users`），你不能再拥有其他同样解析为该路径的文件。例如，以下文件如果同时存在将会导致错误：

- `/routes/users.index.ts`
- `/routes/users.ts`
- `/routes/users/index.ts`

---

## 路径无关布局路由与跳出路由

得益于统一的路由系统，服务器路由同样支持路径无关布局路由（Pathless layout routes）和跳出路由（Break-out routes），这在处理中间件时非常有用：

- **路径无关布局路由**：可用于为一组路由统一添加中间件。
- **跳出路由**：可用于从父级中间件中“跳出”。

---

## 处理服务器路由请求

默认情况下，服务器路由请求由 Start 自动处理。如果你自定义了 `src/server.ts` 入口文件，则由 `createStartHandler` 负责。

Start 处理程序负责将传入请求匹配到对应的服务器路由，并执行相关的中间件和处理函数。

如果你需要自定义服务器处理逻辑，可以创建一个自定义处理程序并将事件传递给 Start 处理程序。详情请参阅 [服务器入口点](./server-entry-point)。

---

## 定义服务器路由

通过在 `createFileRoute` 调用中添加 `server` 属性来创建服务器路由。`server` 属性包含：

- **handlers**：一个将 HTTP 方法映射到处理函数的对象，或者是一个接收 `createHandlers` 以实现更高级用法的函数。
- **middleware**：可选的路由级中间件数组，适用于该路由下的所有处理函数。

```ts
// routes/hello.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/hello")({
  server: {
    handlers: {
      GET: async ({ request }) => {
        return new Response("来自 " + request.url + " 的问候！");
      },
    },
  },
});
```

## 定义服务器路由处理函数 (Handlers)

你可以通过两种方式定义处理函数：

- **简单处理函数 (Simple handlers)**：直接在 `handlers` 对象中提供处理函数。
- **带中间件的处理函数**：使用 `createHandlers` 函数来为特定方法定义带中间件的处理函数。

### 简单处理函数

对于简单的场景，你可以直接在 `handlers` 对象中提供函数。

```ts
// routes/hello.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/hello")({
  server: {
    handlers: {
      GET: async ({ request }) => {
        return new Response("来自 " + request.url + " 的问候！");
      },
    },
  },
});
```

### 为特定处理函数添加中间件

对于更复杂的场景，你可以为特定的 HTTP 方法添加中间件。这需要使用 `createHandlers` 函数：

```tsx
// routes/hello.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/hello")({
  server: {
    handlers: ({ createHandlers }) =>
      createHandlers({
        GET: {
          middleware: [loggerMiddleware], // 仅适用于 GET 请求
          handler: async ({ request }) => {
            return new Response("来自 " + request.url + " 的问候！");
          },
        },
      }),
  },
});
```

### 为所有处理函数添加中间件

你也可以在 `server` 级别使用 `middleware` 属性，添加适用于该路由下所有处理函数的中间件：

```tsx
// routes/hello.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/hello")({
  server: {
    middleware: [authMiddleware, loggerMiddleware], // 适用于所有处理函数
    handlers: {
      GET: async ({ request }) => {
        return new Response("你好，世界！来自 " + request.url);
      },
      POST: async ({ request }) => {
        const body = await request.json();
        return new Response(`你好, ${body.name}!`);
      },
    },
  },
});
```

### 结合路由级与函数级中间件

你可以结合这两种方法 —— 路由级中间件将首先运行，随后是特定处理函数的中间件：

```tsx
// routes/hello.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/hello")({
  server: {
    middleware: [authMiddleware], // 首先为所有处理函数运行
    handlers: ({ createHandlers }) =>
      createHandlers({
        GET: async ({ request }) => {
          return new Response("你好，世界！");
        },
        POST: {
          middleware: [validationMiddleware], // 在 authMiddleware 之后运行，且仅针对 POST
          handler: async ({ request }) => {
            const body = await request.json();
            return new Response(`你好, ${body.name}!`);
          },
        },
      }),
  },
});
```

---

## 处理函数上下文 (Handler Context)

每个 HTTP 方法处理函数都会接收到一个包含以下属性的对象：

- `request`：传入的请求对象。你可以在 [MDN Web Docs](https://developer.mozilla.org/zh-CN/docs/Web/API/Request) 中了解更多关于 `Request` 对象的信息。
- `params`：包含路由动态路径参数的对象。例如，如果路由路径是 `/users/$id`，且请求发送至 `/users/123`，那么 `params` 将是 `{ id: '123' }`。
- `context`：包含请求上下文的对象。这在中间件之间传递数据时非常有用。

处理完请求后，你可以返回一个 `Response` 对象、`Promise<Response>`，甚至可以使用 `@tanstack/react-start` 提供的任何辅助函数来操作响应。

---

## 动态路径参数 (Dynamic Path Params)

服务器路由支持动态路径参数，方式与 TanStack Router 完全一致。例如，名为 `routes/users/$id.ts` 的文件将创建一个接受动态 `id` 参数的 API 路由 `/users/$id`。

```ts
// routes/users/$id.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/users/$id")({
  server: {
    handlers: {
      GET: async ({ params }) => {
        const { id } = params; // 从参数中结构出 ID
        return new Response(`用户 ID: ${id}`);
      },
    },
  },
});

// 访问 /users/123 将看到响应：
// 用户 ID: 123
```

你也可以在单个路由中使用多个动态路径参数：

```ts
// routes/users/$id/posts/$postId.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/users/$id/posts/$postId")({
  server: {
    handlers: {
      GET: async ({ params }) => {
        const { id, postId } = params;
        return new Response(`用户 ID: ${id}, 帖子 ID: ${postId}`);
      },
    },
  },
});

// 访问 /users/123/posts/456 将看到响应：
// 用户 ID: 123, 帖子 ID: 456
```

## 通配符参数 (Wildcard/Splat Param)

服务器路由还支持在路径末尾使用通配符参数，由 `$` 后接空内容表示。例如，名为 `routes/file/$.ts` 的文件将创建一个接受通配符参数的 API 路由 `/file/$`。

```ts
// routes/file/$.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/file/$")({
  server: {
    handlers: {
      GET: async ({ params }) => {
        // 通配符匹配的内容会存储在 _splat 属性中
        const { _splat } = params;
        return new Response(`文件路径: ${_splat}`);
      },
    },
  },
});

// 访问 /file/hello.txt 将看到响应：
// 文件路径: hello.txt
```

---

## 处理带有请求体 (Body) 的请求

要处理 `POST` 请求，只需在 `handlers` 对象中添加 `POST` 处理函数即可。该函数会接收请求对象，你可以通过 `request.json()` 方法访问请求体。

```ts
// routes/hello.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/hello")({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const body = await request.json();
        return new Response(`你好, ${body.name}!`);
      },
    },
  },
});

// 向 /hello 发送一个带有 JSON 请求体（如 { "name": "Tanner" }）的 POST 请求
// 响应内容：你好, Tanner!
```

这也适用于其他 HTTP 方法，如 `PUT`、`PATCH` 和 `DELETE`。你可以添加相应的处理函数，并使用适当的方法访问请求体。

**注意**：`request.json()` 返回一个 `Promise`，因此你需要使用 `await`。根据需要，你也可以使用 `request.text()` 或 `request.formData()`。

---

## 响应 JSON 数据

使用 `Response` 对象返回 JSON 时，通常采用以下模式：

```ts
// routes/hello.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/hello")({
  server: {
    handlers: {
      GET: async ({ request }) => {
        return new Response(JSON.stringify({ message: "你好，世界！" }), {
          headers: {
            "Content-Type": "application/json",
          },
        });
      },
    },
  },
});
```

### 使用 `Response.json` 辅助函数

或者，你可以使用 [`Response.json`](https://developer.mozilla.org/zh-CN/docs/Web/API/Response/json_static) 辅助函数，它会自动为你设置 `Content-Type: application/json` 并序列化对象。

```ts
// routes/hello.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/hello")({
  server: {
    handlers: {
      GET: async ({ request }) => {
        return Response.json({ message: "你好，世界！" });
      },
    },
  },
});
```

---

## 响应状态码 (Status Code)

你可以通过 `Response` 构造函数的第二个参数来设置响应的状态码。

```ts
// routes/hello.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/hello")({
  server: {
    handlers: {
      GET: async ({ request, params }) => {
        const user = await findUser(params.id);
        if (!user) {
          return new Response("未找到用户", {
            status: 404,
          });
        }
        return Response.json(user);
      },
    },
  },
});
```

在上面的示例中，如果未找到用户，我们会返回 `404` 状态码。你可以使用此方法设置任何有效的 HTTP 状态码。

---

## 设置响应头 (Headers)

有时你需要自定义响应头。可以通过向 `Response` 构造函数的第二个参数传递 `headers` 对象来实现。

```ts
// routes/hello.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/hello")({
  server: {
    handlers: {
      GET: async ({ request }) => {
        return new Response("你好，世界！", {
          headers: {
            "Content-Type": "text/plain",
            "Cache-Control": "no-cache",
          },
        });
      },
    },
  },
});
```
