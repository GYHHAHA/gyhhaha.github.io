---
short_title: 客户端入口点
---

# 客户端入口点 (Client Entry Point)

> [!NOTE]
> 客户端入口点在开箱即用的情况下是**可选的**。如果未手动提供，TanStack Start 将默认使用下文所述的逻辑为你自动处理。

将 HTML 发送到客户端只是成功了一半。一旦到达浏览器，我们需要在路由解析后水合（Hydrate）客户端 JavaScript。我们通过使用 `StartClient` 组件水合应用程序的根部来实现这一点：

```tsx
// src/client.tsx
import { StartClient } from "@tanstack/react-start/client";
import { StrictMode } from "react";
import { hydrateRoot } from "react-dom/client";

// 使用 StartClient 组件水合整个 document
hydrateRoot(
  document,
  <StrictMode>
    <StartClient />
  </StrictMode>,
);
```

这使得我们能够在用户的初始服务器请求完成后，立即启动客户端路由逻辑。

---

## 错误处理 (Error Handling)

你可以为客户端入口点包裹错误边界（Error Boundaries），以便优雅地处理客户端运行时错误：

```tsx
// src/client.tsx
import { StartClient } from "@tanstack/react-start/client";
import { StrictMode } from "react";
import { hydrateRoot } from "react-dom/client";
import { ErrorBoundary } from "./components/ErrorBoundary";

hydrateRoot(
  document,
  <StrictMode>
    <ErrorBoundary>
      <StartClient />
    </ErrorBoundary>
  </StrictMode>,
);
```

---

## 开发环境 vs 生产环境

你可能希望在开发环境和生产环境中使用不同的行为逻辑：

```tsx
// src/client.tsx
import { StartClient } from "@tanstack/react-start/client";
import { StrictMode } from "react";
import { hydrateRoot } from "react-dom/client";

const App = (
  <>
    {/* 仅在开发环境下显示标记 */}
    {import.meta.env.DEV && <div>开发模式</div>}
    <StartClient />
  </>
);

hydrateRoot(
  document,
  import.meta.env.DEV ? <StrictMode>{App}</StrictMode> : App,
);
```

客户端入口点让你能够完全控制应用在客户端的初始化方式，同时与 TanStack Start 的服务器端渲染无缝衔接。
