---
short_title: 虚拟文件路由
---

# 虚拟文件路由 (Virtual File Routes)

> 我们要感谢 Remix 团队 [率先提出了虚拟文件路由的概念](https://www.youtube.com/watch?v=fjTX8hQTlEc&t=730s)。我们从他们的工作中汲取了灵感，并将其适配到 TanStack Router 现有的基于文件的路由树生成逻辑中。

虚拟文件路由是一个强大的概念，它允许您使用引用项目中真实文件的代码，以编程方式构建路由树。这在以下情况下非常有用：

- 您希望保留现有的路由组织结构。
- 您希望自定义路由文件的存放位置。
- 您希望完全覆盖 TanStack Router 的基于文件的路由生成逻辑，并构建自己的约定。

下面是一个使用虚拟文件路由将路由树映射到项目中一组真实文件的快速示例：

```tsx
// routes.ts
import {
  rootRoute,
  route,
  index,
  layout,
  physical,
} from "@tanstack/virtual-file-routes";

export const routes = rootRoute("root.tsx", [
  index("index.tsx"),
  layout("pathlessLayout.tsx", [
    route("/dashboard", "app/dashboard.tsx", [
      index("app/dashboard-index.tsx"),
      route("/invoices", "app/dashboard-invoices.tsx", [
        index("app/invoices-index.tsx"),
        route("$id", "app/invoice-detail.tsx"),
      ]),
    ]),
    physical("/posts", "posts"),
  ]),
]);
```

## 配置 (Configuration)

虚拟文件路由可以通过以下两种方式进行配置：

- 用于 Vite/Rspack/Webpack 的 `TanStackRouter` 插件
- 用于 TanStack Router CLI 的 `tsr.config.json` 文件

## 通过 TanStackRouter 插件配置

如果您正在使用用于 Vite/Rspack/Webpack 的 `TanStackRouter` 插件，可以在设置插件时通过将路由文件的路径传递给 `virtualRoutesConfig` 选项来配置虚拟文件路由：

```tsx title="vite.config.ts"
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { tanstackRouter } from "@tanstack/router-plugin/vite";

export default defineConfig({
  plugins: [
    tanstackRouter({
      target: "react",
      virtualRouteConfig: "./routes.ts",
    }),
    react(),
  ],
});
```

或者，您也可以选择直接在配置中定义虚拟路由：

```tsx
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { tanstackRouter } from "@tanstack/router-plugin/vite";
import { rootRoute } from "@tanstack/virtual-file-routes";

const routes = rootRoute("root.tsx", [
  // ... 虚拟路由树的其余部分
]);

export default defineConfig({
  plugins: [tanstackRouter({ virtualRouteConfig: routes }), react()],
});
```

```tsx title="vite.config.ts"
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { tanstackRouter } from "@tanstack/router-plugin/vite";

const routes = rootRoute("root.tsx", [
  // ... 虚拟路由树的其余部分
]);

export default defineConfig({
  plugins: [
    tanstackRouter({ virtualRouteConfig: routes, target: "react" }),
    react(),
  ],
});
```

## 创建虚拟文件路由

要创建虚拟文件路由，您需要安装并导入 `@tanstack/virtual-file-routes` 包。该包提供了一组函数，允许您创建引用项目中真实文件的虚拟路由。该包导出了以下几个实用函数：

- `rootRoute` - 创建一个虚拟根路由。
- `route` - 创建一个虚拟路由。
- `index` - 创建一个虚拟索引路由。
- `layout` - 创建一个虚拟无路径布局路由。
- `physical` - 创建一个物理虚拟路由（稍后会详细介绍）。

## 虚拟根路由 (Virtual Root Route)

`rootRoute` 函数用于创建虚拟根路由。它接受一个文件名和一个子路由数组。下面是一个虚拟根路由的示例：

```tsx
// routes.ts
import { rootRoute } from "@tanstack/virtual-file-routes";

export const routes = rootRoute("root.tsx", [
  // ... 子路由
]);
```

## 虚拟路由 (Virtual Route)

`route` 函数用于创建虚拟路由。它接受一个路径、一个文件名和一个子路由数组。下面是一个虚拟路由的示例：

```tsx
// routes.ts
import { route } from "@tanstack/virtual-file-routes";

export const routes = rootRoute("root.tsx", [
  route("/about", "about.tsx", [
    // ... 子路由
  ]),
]);
```

您也可以定义一个没有文件名的虚拟路由。这允许为其子路由设置一个共同的路径前缀：

```tsx
// routes.ts
import { route } from "@tanstack/virtual-file-routes";

export const routes = rootRoute("root.tsx", [
  route("/hello", [
    route("/world", "world.tsx"), // 完整路径将为 "/hello/world"
    route("/universe", "universe.tsx"), // 完整路径将为 "/hello/universe"
  ]),
]);
```

## 虚拟索引路由 (Virtual Index Route)

`index` 函数用于创建虚拟索引路由。它接受一个文件名。下面是一个虚拟索引路由的示例：

```tsx
import { index } from "@tanstack/virtual-file-routes";

const routes = rootRoute("root.tsx", [index("index.tsx")]);
```

## 虚拟无路径路由 (Virtual Pathless Route)

`layout` 函数用于创建虚拟无路径路由。它接受一个文件名、一个子路由数组以及一个可选的无路径 ID。下面是一个虚拟无路径路由的示例：

```tsx
// routes.ts
import { layout } from "@tanstack/virtual-file-routes";

export const routes = rootRoute("root.tsx", [
  layout("pathlessLayout.tsx", [
    // ... 子路由
  ]),
]);
```

您还可以指定一个无路径 ID，为该路由提供一个不同于文件名的唯一标识符：

```tsx
// routes.ts
import { layout } from "@tanstack/virtual-file-routes";

export const routes = rootRoute("root.tsx", [
  layout("my-pathless-layout-id", "pathlessLayout.tsx", [
    // ... 子路由
  ]),
]);
```

## 物理虚拟路由 (Physical Virtual Routes)

物理虚拟路由是一种将“老牌”的 TanStack Router 基于文件的路由约定目录“挂载 (mount)”到特定 URL 路径下的方法。如果您正在使用虚拟路由来自定义路由树顶层的个别部分，但希望对子路由和目录保留标准的基于文件的路由约定，那么这个功能会非常有用。

考虑以下文件结构：

```
/routes
├── root.tsx
├── index.tsx
├── pathlessLayout.tsx
├── app
│   ├── dashboard.tsx
│   ├── dashboard-index.tsx
│   ├── dashboard-invoices.tsx
│   ├── invoices-index.tsx
│   ├── invoice-detail.tsx
└── posts
    ├── index.tsx
    ├── $postId.tsx
    ├── $postId.edit.tsx
    ├── comments/
    │   ├── index.tsx
    │   ├── $commentId.tsx
    └── likes/
        ├── index.tsx
        ├── $likeId.tsx
```

让我们使用虚拟路由来自定义除 `posts` 之外的所有路由树，然后使用物理虚拟路由将 `posts` 目录挂载到 `/posts` 路径下：

```tsx
// routes.ts
export const routes = rootRoute("root.tsx", [
  // 正常设置您的虚拟路由
  index("index.tsx"),
  layout("pathlessLayout.tsx", [
    route("/dashboard", "app/dashboard.tsx", [
      index("app/dashboard-index.tsx"),
      route("/invoices", "app/dashboard-invoices.tsx", [
        index("app/invoices-index.tsx"),
        route("$id", "app/invoice-detail.tsx"),
      ]),
    ]),
    // 将 `posts` 目录挂载到 `/posts` 路径下
    physical("/posts", "posts"),
  ]),
]);
```

### 在当前层级合并物理路由

您还可以使用不带路径前缀（或仅带一个参数）的 `physical` 函数，将物理目录中的路由直接合并到当前层级，而不添加路径前缀。当您希望将路由组织到不同的目录中，但希望它们出现在相同的 URL 层级时，这非常有用。

考虑以下文件结构：

```
/routes
├── __root.tsx
├── about.tsx
└── features
    ├── index.tsx
    └── contact.tsx
```

您可以将 `features` 目录下的路由合并到根层级：

```tsx
// routes.ts
import { physical, rootRoute, route } from "@tanstack/virtual-file-routes";

export const routes = rootRoute("__root.tsx", [
  route("/about", "about.tsx"),
  // 在根层级合并 features/ 目录下的路由（无路径前缀）
  physical("features"),
  // 或者等效于：physical('', 'features')
]);
```

这将生成以下路由：

- `/about` - 来自 `about.tsx`
- `/` - 来自 `features/index.tsx`
- `/contact` - 来自 `features/contact.tsx`

> **注意：** 在同一层级合并时，请确保您的虚拟路由和物理目录路由之间没有冲突的路由路径。如果发生冲突（例如，两者都有 `/about` 路由），生成器将抛出错误。

## 在基于文件的路由中使用虚拟路由

前一节展示了如何在虚拟路由配置中使用 TanStack Router 的基于文件的路由约定。
然而，反向操作也是可能的。
您可以使用 TanStack Router 的基于文件的路由约定来配置应用程序路由树的主要部分，并针对特定的子树选择使用虚拟路由配置。

考虑以下文件结构：

```
/routes
├── __root.tsx
├── foo
│   ├── bar
│   │   ├── __virtual.ts
│   │   ├── details.tsx
│   │   ├── home.tsx
│   │   └── route.ts
│   └── bar.tsx
└── index.tsx
```

让我们看看 `bar` 目录，它包含一个名为 `__virtual.ts` 的特殊文件。该文件指示生成器为该目录（及其子目录）切换到虚拟文件路由配置。

`__virtual.ts` 配置该特定子树的虚拟路由。它使用与上述相同的 API，唯一的区别是该子树不需要定义 `rootRoute`：

```tsx
// routes/foo/bar/__virtual.ts
import {
  defineVirtualSubtreeConfig,
  index,
  route,
} from "@tanstack/virtual-file-routes";

export default defineVirtualSubtreeConfig([
  index("home.tsx"),
  route("$id", "details.tsx"),
]);
```

辅助函数 `defineVirtualSubtreeConfig` 紧密模仿了 Vite 的 `defineConfig`，允许您通过默认导出定义子树配置。默认导出可以是：

- 一个子树配置对象
- 一个返回子树配置对象的函数
- 一个返回子树配置对象的异步函数

## 盗梦空间 (Inception / 嵌套混合)

您可以根据喜好随心所欲地混搭 TanStack Router 的基于文件的路由约定和虚拟路由配置。
让我们深入一点！
查看以下示例：它首先使用基于文件的路由约定，然后针对 `/posts` 切换到虚拟路由配置，接着仅针对 `/posts/lets-go` 切换回基于文件的路由约定，最后针对 `/posts/lets-go/deeper` 再次切换到虚拟路由配置。

```
├── __root.tsx
├── index.tsx
├── posts
│   ├── __virtual.ts
│   ├── details.tsx
│   ├── home.tsx
│   └── lets-go
│       ├── deeper
│       │   ├── __virtual.ts
│       │   └── home.tsx
│       └── index.tsx
└── posts.tsx
```

## 通过 TanStack Router CLI 进行配置

如果您正在使用 TanStack Router CLI，可以通过在 `tsr.config.json` 文件中定义路由文件的路径来配置虚拟文件路由：

```json
// tsr.config.json
{
  "virtualRouteConfig": "./routes.ts"
}
```

或者，您可以直接在配置中定义虚拟路由。虽然这种情况不常见，但它允许您通过在 `tsr.config.json` 文件中添加 `virtualRouteConfig` 对象来配置它们，方法是定义您的虚拟路由并传递通过调用 `@tanstack/virtual-file-routes` 包中的 `rootRoute`/`route`/`index` 等函数生成的 JSON 结果。

```json
// tsr.config.json
{
  "virtualRouteConfig": {
    "type": "root",
    "file": "root.tsx",
    "children": [
      {
        "type": "index",
        "file": "home.tsx"
      },
      {
        "type": "route",
        "file": "posts/posts.tsx",
        "path": "/posts",
        "children": [
          {
            "type": "index",
            "file": "posts/posts-home.tsx"
          },
          {
            "type": "route",
            "file": "posts/posts-detail.tsx",
            "path": "$postId"
          }
        ]
      },
      {
        "type": "layout",
        "id": "first",
        "file": "layout/first-pathless-layout.tsx",
        "children": [
          {
            "type": "layout",
            "id": "second",
            "file": "layout/second-pathless-layout.tsx",
            "children": [
              {
                "type": "route",
                "file": "a.tsx",
                "path": "/route-a"
              },
              {
                "type": "route",
                "file": "b.tsx",
                "path": "/route-b"
              }
            ]
          }
        ]
      }
    ]
  }
}
```
