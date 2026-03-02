---
short_title: Tailwind CSS
---

# Tailwind CSS 集成 (Tailwind CSS Integration)

_想在你的 TanStack Start 项目中使用 Tailwind CSS 吗？_

本指南将帮助你在 TanStack Start 项目中顺利配置并使用 Tailwind CSS。

## Tailwind CSS 版本 4（最新版）

Tailwind CSS 的最新版本是 v4。与版本 3 相比，它在配置上有一些重大变化。由于 TanStack Start 使用 Vite 作为构建工具，因此在项目中设置 Tailwind CSS v4 **更加简单且更受推荐**。

### 安装 Tailwind CSS

安装 Tailwind CSS 及其 Vite 插件：

```shell
npm install tailwindcss @tailwindcss/vite
```

### 配置 Vite 插件

将 `@tailwindcss/vite` 插件添加到你的 Vite 配置文件中：

```ts
// vite.config.ts
import { defineConfig } from "vite";
import tsConfigPaths from "vite-tsconfig-paths";
import { tanstackStart } from "@tanstack/react-start/plugin/vite";
import tailwindcss from "@tailwindcss/vite";
import viteReact from "@vitejs/plugin-react";

export default defineConfig({
  server: {
    port: 3000,
  },
  plugins: [tsConfigPaths(), tanstackStart(), viteReact(), tailwindcss()],
});
```

### 在 CSS 文件中导入 Tailwind

在版本 4 中，你需要通过 CSS 文件而不是配置文件来配置 Tailwind CSS。你可以创建一个 `src/styles/app.css` 文件（名称可以自定）。

```css
/* src/styles/app.css */
@import "tailwindcss" source("../");
```

---

## 在 `__root.tsx` 文件中导入 CSS

在你的 `__root.tsx` 文件中通过 `?url` 查询字符串导入该 CSS 文件，并确保在文件顶部添加 **三斜杠 (triple-slash)** 指令。

```tsx
// src/routes/__root.tsx
/// <reference types="vite/client" />
// 其他导入项...

import appCss from "../styles/app.css?url";

export const Route = createRootRoute({
  head: () => ({
    meta: [
      // 你的 meta 标签和站点配置
    ],
    // 将 CSS 文件添加至 links 数组中
    links: [{ rel: "stylesheet", href: appCss }],
    // 其他头部配置
  }),
  component: RootComponent,
});
```

---

## 在项目中的任何地方使用 Tailwind CSS

现在，你可以在项目的任何地方使用 Tailwind CSS 类名了：

```tsx
// src/routes/index.tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/")({
  component: Home,
});

function Home() {
  return (
    <div className="bg-red-500 text-white p-4">你好，世界 (Hello World)</div>
  );
}
```

大功告成！你现在可以在项目中尽情使用 Tailwind CSS 了 🎉。

---

## Tailwind CSS 版本 3（旧版本）

如果你希望使用 Tailwind CSS v3，请参考以下步骤：

### 安装 Tailwind CSS

安装 Tailwind CSS 及其对等依赖项：

```shell
npm install -D tailwindcss@3 postcss autoprefixer
```

然后生成 Tailwind 和 PostCSS 配置文件：

```shell
npx tailwindcss init -p
```

### 配置模板路径

在 `tailwind.config.js` 文件中添加所有模板文件的路径：

```js
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
export default {
  content: ["./src/**/*.{js,ts,jsx,tsx}"],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

### 在 CSS 文件中添加 Tailwind 指令

将 Tailwind 各个层级的 `@tailwind` 指令添加到你的 `src/styles/app.css` 文件中：

```css
/* src/styles/app.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```
