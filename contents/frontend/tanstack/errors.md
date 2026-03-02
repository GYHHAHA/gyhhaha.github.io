---
short_title: 错误边界
---

# 错误边界 (Error Boundaries)

## 错误边界 (React Start)

TanStack Start 使用了 TanStack Router 的路由级错误边界机制。你可以：

- 通过路由器（Router）为所有路由设置全局默认配置。
- 在每个路由上通过 `errorComponent` 进行单独覆盖。

---

### 配置全局默认值

```tsx
// src/router.tsx
import { createRouter, ErrorComponent } from "@tanstack/react-router";
import { routeTree } from "./routeTree.gen";

export function getRouter() {
  const router = createRouter({
    routeTree,
    // 当错误冒泡到路由器时显示
    defaultErrorComponent: ({ error, reset }) => (
      <ErrorComponent error={error} />
    ),
  });
  return router;
}
```

---

### 按路由覆盖 (Per-route override)

```tsx
// src/routes/posts.$postId.tsx
import { createFileRoute, ErrorComponent } from "@tanstack/react-router";
import type { ErrorComponentProps } from "@tanstack/react-router";

function PostError({ error, reset }: ErrorComponentProps) {
  // reset 函数允许你尝试重新渲染该路由
  return (
    <div>
      <h3>加载帖子时出错</h3>
      <ErrorComponent error={error} />
      <button onClick={() => reset()}>重试</button>
    </div>
  );
}

export const Route = createFileRoute("/posts/$postId")({
  component: PostComponent,
  // 为此特定路由定义错误组件
  errorComponent: PostError,
});
```

---

### 注意事项

- **`ErrorComponent`**：这是一个简单的内置 UI 组件，你可以根据需要随时替换为自己的自定义 UI。
- **`reset()` 函数**：在修复状态或网络问题后，调用 `reset()` 可以尝试重新渲染该路由。
- **触发时机**：在 `beforeLoad` 或 `loader` 中抛出的错误（Errors）都会被这些错误边界捕获。
