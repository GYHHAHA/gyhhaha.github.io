---
short_title: 静态预渲染
---

# 静态预渲染 (Static Prerendering)

静态预渲染是指为你的应用程序生成静态 HTML 文件的过程。这对于提升应用性能非常有用，因为它允许你向用户直接提供预渲染好的 HTML 文件，而无需即时生成。此外，它还适用于将静态站点部署到那些不支持服务器端渲染（SSR）的平台上。

## 预渲染配置 (Prerendering)

TanStack Start 可以将你的应用预渲染为静态 HTML 文件。要启用此功能，你可以在 `vite.config.ts` 文件的 `tanstackStart` 配置中添加 `prerender` 选项：

```ts
// vite.config.ts

import { tanstackStart } from "@tanstack/react-start/plugin/vite";
import viteReact from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [
    tanstackStart({
      prerender: {
        // 启用预渲染
        enabled: true,

        // 如果你需要页面生成在 `/page/index.html` 而不是 `/page.html`，请开启此项
        autoSubfolderIndex: true,

        // 如果禁用，则只有根路径或 pages 配置中定义的路径会被预渲染
        autoStaticPathsDiscovery: true,

        // 同时运行的预渲染任务数量（并发数）
        concurrency: 14,

        // 是否从 HTML 中提取链接并也对它们进行预渲染
        crawlLinks: true,

        // 过滤器函数：接收页面对象并返回是否应该预渲染该页面
        filter: ({ path }) => !path.startsWith("/do-not-render-me"),

        // 预渲染任务失败时的重试次数
        retryCount: 2,

        // 重试之间的延迟时间（毫秒）
        retryDelay: 1000,

        // 预渲染期间允许跟随的最大重定向次数
        maxRedirects: 5,

        // 如果预渲染过程中发生错误，则构建失败
        failOnError: true,

        // 页面渲染成功后的回调
        onSuccess: ({ page }) => {
          console.log(`已渲染 ${page.path}!`);
        },
      },
      // 针对特定页面的可选配置
      // 注意：当 autoStaticPathsDiscovery 开启时（默认），
      // 自动发现的静态路由将与下面指定的页面合并
      pages: [
        {
          path: "/my-page",
          prerender: { enabled: true, outputPath: "/my-page/index.html" },
        },
      ],
    }),
    viteReact(),
  ],
});
```

---

## 自动静态路由发现 (Automatic Static Route Discovery)

所有的静态路径都会被自动发现，并与指定的 `pages` 配置无缝合并。

在以下情况下，路由会被排除在自动发现之外：

- **带有路径参数的路由**（例如 `/users/$userId`），因为它们需要特定的参数值。
- **布局路由**（以 `_` 为前缀），因为它们不会渲染为独立的页面。
- **没有组件的路由**（例如 API 路由）。

**注意：** 如果开启了 `crawlLinks`，动态路由如果被其他页面链接到，仍然可以被预渲染。

---

## 链接爬取 (Crawling Links)

当 `crawlLinks` 开启时（默认值为 `true`），TanStack Start 会从已预渲染的页面中提取链接，并顺藤摸瓜对这些链接指向的页面也进行预渲染。

例如，如果 `/` 页面包含一个指向 `/posts` 的链接，那么 `/posts` 也会被自动预渲染。

> [!TIP]
> 结合 `autoStaticPathsDiscovery` 和 `crawlLinks` 可以最大程度地自动化预渲染流程，这对于内容驱动型的网站（如博客或文档库）非常高效。
