---
short_title: 文档头部管理
---

# 文档头部管理 (Document Head Management)

文档头部管理是指管理文档中的 `head`、`title`、`meta`、`link` 和 `script` 标签的过程。TanStack Router 为使用 Start 的全栈应用以及使用 TanStack Router 的单页应用（SPA）提供了一套强大的管理方案。其特性包括：

- 自动对 `title` 和 `meta` 标签进行去重
- 根据路由的可见性自动加载/卸载标签
- 支持以组合式的方式合并嵌套路由中的 `title` 和 `meta` 标签

无论是对于全栈应用还是单页应用，管理文档头部都是至关重要的，原因如下：

- **SEO**：搜索引擎优化
- **社交媒体分享**：生成分享卡片预览
- **分析脚本**：挂载监控代码
- **资源加载**：CSS 和 JS 的动态加载与卸载

要管理文档头部，必须渲染 `<HeadContent />` 和 `<Scripts />` 组件，并使用 `routeOptions.head` 属性。该属性返回一个包含 `title`、`meta`、`links`、`styles` 和 `scripts` 属性的对象。

## 管理文档头部

```tsx
export const Route = createRootRoute({
  head: () => ({
    meta: [
      {
        name: "description",
        content: "我的应用是一个 Web 应用程序",
      },
      {
        title: "我的应用",
      },
    ],
    links: [
      {
        rel: "icon",
        href: "/favicon.ico",
      },
    ],
    styles: [
      {
        media: "all and (max-width: 500px)",
        children: `p {
                  color: blue;
                  background-color: yellow;
                }`,
      },
    ],
    scripts: [
      {
        src: "https://www.google-analytics.com/analytics.js",
      },
    ],
  }),
});
```

### 去重机制 (Deduping)

TanStack Router 开箱即支持对 `title` 和 `meta` 标签进行去重，并遵循**后到优先**原则（即优先使用嵌套路由中最后发现的标签）。

- 在嵌套路由中定义的 `title` 标签将覆盖父路由中定义的 `title`（但你可以将它们组合在一起，本指南后续章节会介绍）。
- 具有相同 `name` 或 `property` 的 `meta` 标签将被嵌套路由中最后出现的同名标签覆盖。

### `<HeadContent />`

渲染文档的头部、标题、元数据、链接和头部相关的脚本标签**必须**使用 `<HeadContent />` 组件。

它应该在根布局的 `<head>` 标签中渲染；如果你的应用无法直接管理 `<head>` 标签，则应尽可能渲染在组件树的顶端。

### Start/全栈应用示例

```tsx
import { HeadContent } from "@tanstack/react-router";

export const Route = createRootRoute({
  component: () => (
    <html>
      <head>
        <HeadContent />
      </head>
      <body>
        <Outlet />
      </body>
    </html>
  ),
});
```

### 单页应用 (SPA) 示例

首先，如果你的 `index.html` 中设置了 `<title>` 标签，请将其移除。

```tsx
import { HeadContent } from "@tanstack/react-router";

const rootRoute = createRootRoute({
  component: () => (
    <>
      <HeadContent />
      <Outlet />
    </>
  ),
});
```

## 管理 Body 脚本

除了可以在 `<head>` 中渲染脚本外，你还可以使用 `routeOptions.scripts` 属性在 `<body>` 标签中渲染脚本。这对于加载那些需要 DOM 已加载、但在应用主入口点（包括 Start 或全栈实现中的注水/Hydration）运行之前的脚本（甚至是内联脚本）非常有用。

操作步骤：

- 使用 `routeOptions` 对象的 `scripts` 属性
- 渲染 `<Scripts />` 组件

```tsx
export const Route = createRootRoute({
  scripts: () => [
    {
      children: 'console.log("Hello, world!")',
    },
  ],
});
```

### `<Scripts />`

渲染文档的 body 脚本**必须**使用 `<Scripts />` 组件。它应该在根布局的 `<body>` 标签中渲染，或者在应用无法管理 `<body>` 标签时，尽可能渲染在组件树的高处。

### 示例

```tsx
import { createRootRoute, Scripts } from "@tanstack/react-router";
export const Route = createFileRoute("/")({
  component: () => (
    <html>
      <head />
      <body>
        <Outlet />
        <Scripts />
      </body>
    </html>
  ),
});
```

## 使用 ScriptOnce 渲染内联脚本

对于必须在 React 注水（Hydration）之前运行的脚本（例如主题检测），请使用 `ScriptOnce`。这对于避免样式闪烁（FOUC）或主题切换闪烁特别有效。

```tsx
import { ScriptOnce } from "@tanstack/react-router";

const themeScript = `(function() {
  try {
    const theme = localStorage.getItem('theme') || 'auto';
    const resolved = theme === 'auto'
      ? (matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light')
      : theme;
    document.documentElement.classList.add(resolved);
  } catch (e) {}
})();`;

function ThemeProvider({ children }) {
  return (
    <>
      <ScriptOnce children={themeScript} />
      {children}
    </>
  );
}
```

### ScriptOnce 的工作原理

1. **SSR 期间**：渲染一个包含所提供代码的 `<script>` 标签。
2. **浏览器解析**：当浏览器解析 HTML 时，脚本立即执行（在 React 注水之前）。
3. **自我销毁**：执行完毕后，脚本会从 DOM 中自行移除。
4. **客户端导航**：在后续的客户端导航中，该组件不会渲染任何内容（防止重复执行）。

### 防止注水警告

如果你的脚本在注水前修改了 DOM（例如向 `<html>` 添加类名），请使用 `suppressHydrationWarning` 来防止 React 抛出警告：

```tsx
export const Route = createRootRoute({
  component: () => (
    <html lang="en" suppressHydrationWarning>
      <head>
        <HeadContent />
      </head>
      <body>
        <ThemeProvider>
          <Outlet />
        </ThemeProvider>
        <Scripts />
      </body>
    </html>
  ),
});
```

### 常见用例

- **主题/深色模式检测**：在注水前应用主题类，防止页面闪烁。
- **特性检测**：在渲染前检查浏览器能力。
- **分析初始化**：在用户交互前初始化追踪代码。
- **关键路径设置**：任何必须在注水前运行的 JavaScript 逻辑。
