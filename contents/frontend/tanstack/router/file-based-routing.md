---
short_title: 基于文件的路由
---

# 基于文件的路由 (File-Based Routing)

TanStack Router 的大部分文档都是围绕“基于文件的路由”编写的，旨在帮助您深入了解如何配置它及其背后的技术细节。虽然基于文件的路由是配置 TanStack Router 的首选且推荐方式，但如果您愿意，也可以使用 [基于代码的路由 (Code-Based Routing)](./code-based-routing.md)。

## 什么是基于文件的路由？

基于文件的路由是一种通过文件系统来配置路由的方法。您无需通过代码定义路由结构，而是通过一系列代表应用程序路由层级的文件和目录来定义。这带来了许多好处：

- **简单直观**：无论是新手还是经验丰富的开发人员，基于文件的路由在视觉上都非常直观且易于理解。
- **组织有序**：路由的组织方式与应用程序的 URL 结构一一对应。
- **可扩展性**：随着应用程序的增长，基于文件的路由使得添加新路由和维护现有路由变得轻而易举。
- **代码分割**：基于文件的路由允许 TanStack Router 自动进行路由级别的代码分割，以获得更好的性能。
- **类型安全**：它通过自动生成和管理路由的类型链接，将类型安全的上限提升到了新的高度，而这在基于代码的路由中通常是一个繁琐的过程。
- **一致性**：它强制执行一致的路由结构，使维护、更新应用以及在不同项目间切换变得更加容易。

## 使用 `/` 还是 `.`？

虽然长期以来我们习惯用目录（文件夹）来表示路由层级，但基于文件的路由引入了一个额外的概念：在文件名中使用 `.` 字符来表示路由嵌套。这让您可以避免为少量的深层嵌套路由创建繁琐的目录，同时在庞大的路由层级中继续保留目录的使用。让我们来看一些例子！

---

## 目录路由 (Directory Routes)

目录可以用来表示路由层级。这对于将多个路由组织到逻辑组中非常有用，也能缩短大型深层嵌套路由组的文件名长度。

示例如下：

| 文件名                  | 路由路径                  | 组件输出输出                      |
| ----------------------- | ------------------------- | --------------------------------- |
| ʦ `__root.tsx`          |                           | `<Root>`                          |
| ʦ `index.tsx`           | `/` (精确匹配)            | `<Root><RootIndex>`               |
| ʦ `about.tsx`           | `/about`                  | `<Root><About>`                   |
| ʦ `posts.tsx`           | `/posts`                  | `<Root><Posts>`                   |
| 📂 `posts`              |                           |                                   |
| ┄ ʦ `index.tsx`         | `/posts` (精确匹配)       | `<Root><Posts><PostsIndex>`       |
| ┄ ʦ `$postId.tsx`       | `/posts/$postId`          | `<Root><Posts><Post>`             |
| 📂 `posts_`             |                           |                                   |
| ┄ 📂 `$postId`          |                           |                                   |
| ┄ ┄ ʦ `edit.tsx`        | `/posts/$postId/edit`     | `<Root><EditPost>`                |
| ʦ `settings.tsx`        | `/settings`               | `<Root><Settings>`                |
| 📂 `settings`           |                           | `<Root><Settings>`                |
| ┄ ʦ `profile.tsx`       | `/settings/profile`       | `<Root><Settings><Profile>`       |
| ┄ ʦ `notifications.tsx` | `/settings/notifications` | `<Root><Settings><Notifications>` |
| ʦ `_pathlessLayout.tsx` |                           | `<Root><PathlessLayout>`          |
| 📂 `_pathlessLayout`    |                           |                                   |
| ┄ ʦ `route-a.tsx`       | `/route-a`                | `<Root><PathlessLayout><RouteA>`  |
| ┄ ʦ `route-b.tsx`       | `/route-b`                | `<Root><PathlessLayout><RouteB>`  |
| 📂 `files`              |                           |                                   |
| ┄ ʦ `$.tsx`             | `/files/$`                | `<Root><Files>`                   |
| 📂 `account`            |                           |                                   |
| ┄ ʦ `route.tsx`         | `/account`                | `<Root><Account>`                 |
| ┄ ʦ `overview.tsx`      | `/account/overview`       | `<Root><Account><Overview>`       |

---

## 扁平路由 (Flat Routes)

扁平路由让您能够使用 `.` 来表示路由嵌套层级。

当您有大量唯一的、深层嵌套的路由，且不想为每个路由都创建目录时，这种方式非常有用：

示例如下：

| 文件名                          | 路由路径                  | 组件输出                          |
| ------------------------------- | ------------------------- | --------------------------------- |
| ʦ `__root.tsx`                  |                           | `<Root>`                          |
| ʦ `index.tsx`                   | `/` (精确匹配)            | `<Root><RootIndex>`               |
| ʦ `about.tsx`                   | `/about`                  | `<Root><About>`                   |
| ʦ `posts.tsx`                   | `/posts`                  | `<Root><Posts>`                   |
| ʦ `posts.index.tsx`             | `/posts` (精确匹配)       | `<Root><Posts><PostsIndex>`       |
| ʦ `posts.$postId.tsx`           | `/posts/$postId`          | `<Root><Posts><Post>`             |
| ʦ `posts_.$postId.edit.tsx`     | `/posts/$postId/edit`     | `<Root><EditPost>`                |
| ʦ `settings.tsx`                | `/settings`               | `<Root><Settings>`                |
| ʦ `settings.profile.tsx`        | `/settings/profile`       | `<Root><Settings><Profile>`       |
| ʦ `settings.notifications.tsx`  | `/settings/notifications` | `<Root><Settings><Notifications>` |
| ʦ `_pathlessLayout.tsx`         |                           | `<Root><PathlessLayout>`          |
| ʦ `_pathlessLayout.route-a.tsx` | `/route-a`                | `<Root><PathlessLayout><RouteA>`  |
| ʦ `_pathlessLayout.route-b.tsx` | `/route-b`                | `<Root><PathlessLayout><RouteB>`  |
| ʦ `files.$.tsx`                 | `/files/$`                | `<Root><Files>`                   |
| ʦ `account.tsx`                 | `/account`                | `<Root><Account>`                 |
| ʦ `account.overview.tsx`        | `/account/overview`       | `<Root><Account><Overview>`       |

---

## 混合使用扁平路由与目录路由

在实际项目中，100% 的目录结构或 100% 的扁平结构往往不是最优解。这就是为什么 TanStack Router 允许您将两者混合使用，在合适的场景下取长补短：

| 文件名                         | 路由路径                  | 组件输出                          |
| ------------------------------ | ------------------------- | --------------------------------- |
| ʦ `__root.tsx`                 |                           | `<Root>`                          |
| ʦ `index.tsx`                  | `/` (精确匹配)            | `<Root><RootIndex>`               |
| ʦ `about.tsx`                  | `/about`                  | `<Root><About>`                   |
| ʦ `posts.tsx`                  | `/posts`                  | `<Root><Posts>`                   |
| 📂 `posts`                     |                           |                                   |
| ┄ ʦ `index.tsx`                | `/posts` (精确匹配)       | `<Root><Posts><PostsIndex>`       |
| ┄ ʦ `$postId.tsx`              | `/posts/$postId`          | `<Root><Posts><Post>`             |
| ┄ ʦ `$postId.edit.tsx`         | `/posts/$postId/edit`     | `<Root><Posts><Post><EditPost>`   |
| ʦ `settings.tsx`               | `/settings`               | `<Root><Settings>`                |
| ʦ `settings.profile.tsx`       | `/settings/profile`       | `<Root><Settings><Profile>`       |
| ʦ `settings.notifications.tsx` | `/settings/notifications` | `<Root><Settings><Notifications>` |
| ʦ `account.tsx`                | `/account`                | `<Root><Account>`                 |
| ʦ `account.overview.tsx`       | `/account/overview`       | `<Root><Account><Overview>`       |

> [!TIP]
> 如果您发现默认的基于文件的路由结构不符合您的需求，您可以随时使用 [虚拟文件路由 (Virtual File Routes)](./virtual-file-routes.md) 来控制路由来源，同时依然享受基于文件路由带来的出色性能。

---

## 开始使用基于文件的路由

要开始使用基于文件的路由，您需要配置项目的捆绑器 (Bundler) 以使用 TanStack Router 插件或 TanStack Router CLI。

要启用此功能，您需要配合 React 以及受支持的捆绑器。请查看下方的配置指南，确认您的捆绑器是否在列：

- [通过 Vite 安装](../installation/with-vite)
- [通过 Rspack/Rsbuild 安装](../installation/with-rspack)
- [通过 Webpack 安装](../installation/with-webpack)
- [通过 Esbuild 安装](../installation/with-esbuild)

当您在受支持的捆绑器中使用 TanStack Router 的基于文件路由时，我们的插件会**通过捆绑器的开发 (dev) 和构建 (build) 过程自动生成路由配置**。这是使用 TanStack Router 路由生成功能最简单的方式。

如果您的捆绑器尚未受到支持，您可以在 Discord 或 GitHub 上联系我们。
