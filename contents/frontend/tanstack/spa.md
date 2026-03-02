---
short_title: SPA 模式
---

# SPA 模式

## 究竟什么是 SPA 模式？

对于那些不需要为了 SEO、爬虫或性能原因而使用 SSR 的应用程序，将静态 HTML 发送给用户可能更为理想。这个静态 HTML 包含了应用程序的“壳”（Shell）（甚至是特定路由的预渲染 HTML），其中包含必要的 `html`、`head` 和 `body` 标签，仅在客户端启动应用程序。

## 为什么在不使用 SSR 的情况下使用 Start？

**不使用 SSR 并不意味着放弃服务器端功能！** SPA 模式实际上与服务器端功能（如服务器函数 Server Functions 和/或服务器路由 Server Routes，甚至其他外部 API）配合得非常好。它**仅仅意味着初始文档在通过 JavaScript 在客户端渲染之前，不会包含应用程序完整渲染后的 HTML**。

## SPA 模式的优势

- **部署更简单** - 你只需要一个能够托管静态资源的 CDN 即可。
- **托管成本更低** - 与 Lambda 函数或长期运行的进程相比，CDN 非常便宜。
- **仅限客户端更简单** - 没有 SSR 意味着在注水（Hydration）、渲染和路由方面出错的可能性更小。

## SPA 模式的局限性

- **首屏内容显示较慢** - 达到完全内容显示的时间更长，因为在渲染“壳”以下的内容之前，必须下载并执行所有的 JS。
- **SEO 友好度较低** - 机器人、爬虫和链接预览器在索引你的应用程序时可能会更困难，除非它们被配置为执行 JS，且你的应用程序能在合理的时间内渲染完成。

## 它是如何工作的？

启用 SPA 模式后，运行 Start 构建将增加一个额外的预渲染步骤来生成“壳”。其步骤如下：

- **仅预渲染**应用程序的**根路由（Root Route）**。
- 在应用程序通常渲染匹配路由的地方，改为渲染路由器配置的 **Pending 待处理回退组件（Pending Fallback Component）**。
- 生成的 HTML 存储在一个名为 `/_shell.html`（可配置）的静态 HTML 页面中。
- 配置默认重写逻辑，将所有 404 请求重定向到该 SPA 模式的“壳”。

> [!NOTE]
> 其他路由也可以被预渲染，建议在 SPA 模式下尽可能多地预渲染，但这不是 SPA 模式运行的必要条件。

## 配置 SPA 模式

要配置 SPA 模式，可以在 Start 插件选项中添加以下配置：

```tsx
// vite.config.ts
export default defineConfig({
  plugins: [
    tanstackStart({
      spa: {
        enabled: true,
      },
    }),
  ],
});
```

## 使用必要的重定向

将纯客户端 SPA 部署到主机或 CDN 通常需要使用重定向，以确保 URL 被正确重写到 SPA 壳。任何部署的目标应按以下顺序包含这些优先级：

1. 确保静态资源（如果存在）始终被响应（例如 `/about.html`）。这通常是大多数 CDN 的默认行为。
2. （可选）允许特定的子路径路由到任何动态服务器处理器（例如 `/api/**`）。
3. 确保所有 404 请求都被重写到 SPA 壳（例如将所有未匹配请求重定向到 `/_shell.html`）。

### 基础重定向示例

让我们使用 Netlify 的 `_redirects` 文件将所有 404 请求重写到 SPA 壳：

```text
# 捕获所有其他 404 请求并将其重写到 SPA 壳
/* /_shell.html 200
```

## 允许服务器函数和服务器路由

同样，使用 Netlify 的 `_redirects` 文件，我们可以将特定子路径列入白名单，以便路由到服务器：

```text
# 允许请求 /_serverFn/* 路由到服务器
/_serverFn/* /_serverFn/:splat 200

# 允许任何对 /api/* 的请求路由到服务器
/api/* /api/:splat 200

# 捕获所有其他 404 请求并将其重写到 SPA 壳
/* /_shell.html 200
```

## 壳掩码路径 (Shell Mask Path)

用于生成 SPA 壳的默认路径名是 `/`。我们称之为**壳掩码路径**。由于壳中不包含匹配的路由，因此用于生成壳的路径名大多无关紧要，但它仍然是可配置的。

> [!NOTE]
> 建议保留默认值 `/` 作为壳掩码路径。

```tsx
// vite.config.ts
export default defineConfig({
  plugins: [
    tanstackStart({
      spa: {
        maskPath: "/app",
      },
    }),
  ],
});
```

## 预渲染选项

`prerender` 选项用于配置 SPA 壳的预渲染行为，它接受与预渲染指南中相同的选项。

**默认情况下，设置了以下 `prerender` 选项：**

- `outputPath`: `/_shell.html`
- `crawlLinks`: `false`
- `retryCount`: `0`

这意味着默认情况下，系统不会抓取壳中的链接以进行额外的预渲染，也不会重试失败的预渲染。你可以随时覆盖这些选项：

```tsx
// vite.config.ts
export default defineConfig({
  plugins: [
    tanstackStart({
      spa: {
        prerender: {
          outputPath: "/custom-shell",
          crawlLinks: true,
          retryCount: 3,
        },
      },
    }),
  ],
});
```

## 在 SPA 模式下自定义渲染 (Customized rendering in SPA mode)

在以下场景中，自定义 SPA 壳的 HTML 输出会非常有用：

- 为所有 SPA 路由提供通用的 Head 标签
- 提供自定义的 Pending 回退组件（加载占位图）
- 修改壳中 HTML、CSS 和 JS 的任何细节

为了简化这一过程，你可以通过 `router` 实例获取 `isShell()` 函数：

```tsx
// src/routes/root.tsx
export default function Root() {
  const isShell = useRouter().isShell();

  if (isShell) console.log("正在渲染壳 (Shell)！");

  // 你可以根据 isShell 的布尔值进行条件渲染
}
```

你可以利用这个布尔值根据当前是否为“壳”来渲染不同的 UI。但请注意：在壳完成注水（Hydrate）后，路由器会立即导航到首个路由，此时 `isShell()` 将返回 `false`。**如果处理不当，这可能会导致样式未加载或内容闪烁（Flashes of unstyled content）。**

---

## 在壳中使用动态数据 (Dynamic Data in your Shell)

由于壳是使用应用程序的 SSR 构建版本进行预渲染的，因此在**根路由 (Root Route)** 上定义的任何 `loader` 或服务器特定功能都会在预渲染过程中运行，并且这些数据会被包含在生成的壳中。

这意味着你可以通过 `loader` 或服务器功能在壳中使用动态数据。

```tsx
// src/routes/__root.tsx

export const RootRoute = createRootRoute({
  // 这个 loader 将在预渲染壳时在服务器端运行
  loader: async () => {
    return {
      name: "Tanner",
    };
  },
  component: Root,
});

export default function Root() {
  const { name } = useLoaderData();

  return (
    <html>
      <body>
        <h1>你好, {name}!</h1>
        {/* 在 SPA 模式下渲染壳时，
            Outlet 通常会渲染你配置的 pendingFallbackComponent 
        */}
        <Outlet />
      </body>
    </html>
  );
}
```

> [!TIP]
> 这是一个非常强大的特性：它允许你在静态 SPA 中预先注入一些“准静态”信息（如站点标题、基础配置或当前版本号），而无需等待客户端 JavaScript 启动后再去拉取。
