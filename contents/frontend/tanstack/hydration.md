---
short_title: 水合错误
---

# 水合错误 (Hydration Errors)

### 为什么会发生水合错误？

- **不匹配 (Mismatch)**：在水合过程中，服务器生成的 HTML 与客户端首次渲染的结果不一致。
- **常见原因**：使用 `Intl`（语言区域/时区）、`Date.now()`、随机 ID、仅响应式的逻辑（Responsive-only logic）、功能开关（Feature flags）以及用户偏好设置。

---

### 策略 1 —— 使服务器和客户端保持一致

- **在服务器上选择一个确定的语言区域/时区**，并在客户端使用相同的配置。
- **数据源**：首选 Cookie（推荐），其次是 `Accept-Language` 请求头。
- **在服务器端计算一次**并将其作为初始状态进行水合。

```tsx
// src/start.ts
import { createStart, createMiddleware } from "@tanstack/react-start";
import {
  getRequestHeader,
  getCookie,
  setCookie,
} from "@tanstack/react-start/server";

const localeTzMiddleware = createMiddleware().server(async ({ next }) => {
  const header = getRequestHeader("accept-language");
  const headerLocale = header?.split(",")[0] || "en-US";
  const cookieLocale = getCookie("locale");
  const cookieTz = getCookie("tz"); // 由客户端稍后设置（见策略 2）

  const locale = cookieLocale || headerLocale;
  const timeZone = cookieTz || "UTC"; // 在客户端发送时区之前保持确定性

  // 可选：为后续请求持久化语言区域
  setCookie("locale", locale, { path: "/", maxAge: 60 * 60 * 24 * 365 });

  return next({ context: { locale, timeZone } });
});

export const startInstance = createStart(() => ({
  requestMiddleware: [localeTzMiddleware],
}));
```

```tsx
// src/routes/index.tsx (示例)
import * as React from "react";
import { createFileRoute } from "@tanstack/react-router";
import { createServerFn } from "@tanstack/react-start";
import { getCookie } from "@tanstack/react-start/server";

export const getServerNow = createServerFn().handler(async () => {
  const locale = getCookie("locale") || "en-US";
  const timeZone = getCookie("tz") || "UTC";
  return new Intl.DateTimeFormat(locale, {
    dateStyle: "medium",
    timeStyle: "short",
    timeZone,
  }).format(new Date());
});

export const Route = createFileRoute("/")({
  loader: () => getServerNow(),
  component: () => {
    const serverNow = Route.useLoaderData() as string;
    return <time dateTime={serverNow}>{serverNow}</time>;
  },
});
```

---

### 策略 2 —— 让客户端告知其环境信息

- 在用户首次访问时，通过客户端设置一个包含时区的 Cookie；在此之前，SSR 使用 `UTC` 时区。
- 这种做法可以避免产生不匹配的风险。

```tsx
import * as React from "react";
import { ClientOnly } from "@tanstack/react-router";

function SetTimeZoneCookie() {
  React.useEffect(() => {
    const tz = Intl.DateTimeFormat().resolvedOptions().timeZone;
    document.cookie = `tz=${tz}; path=/; max-age=31536000`;
  }, []);
  return null;
}

export function AppBoot() {
  return (
    <ClientOnly fallback={null}>
      <SetTimeZoneCookie />
    </ClientOnly>
  );
}
```

---

### 策略 3 —— 采用纯客户端渲染 (Client-only)

- 使用 `<ClientOnly>` 包裹不稳定的 UI，以跳过 SSR 从而避免水合不匹配。

```tsx
import { ClientOnly } from "@tanstack/react-router";

<ClientOnly fallback={<span>—</span>}>
  <RelativeTime ts={someTs} />
</ClientOnly>;
```

---

### 策略 4 —— 为该路由禁用或限制 SSR

- 使用 **选择性 SSR (Selective SSR)** 来避免在服务器上渲染该组件。

```tsx
export const Route = createFileRoute("/unstable")({
  ssr: "data-only", // 或者设置为 false
  component: () => <ExpensiveViz />,
});
```

---

### 策略 5 —— 最后的手段：抑制警告

- 对于已知存在细微差异的小型节点，可以使用 React 的 `suppressHydrationWarning` 属性。

```tsx
<time suppressHydrationWarning>{new Date().toLocaleString()}</time>
```

---

### 检查清单 (Checklist)

- **确定性输入**：确保语言区域、时区、功能开关是确定的。
- **优先使用 Cookies** 获取客户端上下文，以 `Accept-Language` 作为备选。
- **使用 `<ClientOnly>`** 处理本质上动态的 UI。
- **使用选择性 SSR** 处理服务器 HTML 无法保持稳定的情况。
- **避免盲目抑制**：谨慎使用 `suppressHydrationWarning`。

另请参阅：[执行模型 (Execution Model)](./execution.md)、[代码执行模式 (Code Execution Patterns)](./patterns.md)、[选择性 SSR (Selective SSR)](./ssr.md)、[服务器函数 (Server Functions)](./functions.md)。
