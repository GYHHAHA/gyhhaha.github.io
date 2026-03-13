---
short_title: 搜索参数
---

# 查询参数 (Search Params)

正如 TanStack Query 让 React 和 Solid 应用处理服务器状态变得轻而易举一样，TanStack Router 旨在解锁应用中 URL 查询参数的强大力量。

> 🧠 如果您使用的是非常老旧的浏览器（如 IE11），可能需要为 `URLSearchParams` 使用 Polyfill。

## 为什么不直接使用 `URLSearchParams`？

我们理解，最近您可能听到了很多“使用平台原生能力（use the platform）”的声音，在大多数情况下我们是认同的。然而，我们也认为认识到平台在高级用例中的不足同样重要，而我们认为 `URLSearchParams` 正属于这种情况。

传统的查询参数 API 通常假设：

- 查询参数永远是字符串。
- 它们*基本*是扁平的。
- 使用 `URLSearchParams` 进行序列化和反序列化就足够了（剧透一下：其实不够）。
- 查询参数的修改与 URL 的路径名（Pathname）紧密耦合，即使路径名没有改变，也必须一起更新。

但现实情况大不相同：

- **状态管理需求**：查询参数代表了应用状态，因此我们期望它们具有与其他状态管理器相同的开发者体验（DX）。这意味着需要能够区分原始值类型，并高效地存储和操作复杂的数据结构（如嵌套数组和对象）。
- **序列化方案**：序列化和反序列化状态有许多权衡。您应该能够为应用选择最佳方案，或者至少获得一个比 `URLSearchParams` 更好的默认方案。
- **不可变性与结构共享**：每次字符串化和解析 URL 查询参数时，引用的完整性和对象标识都会丢失，因为每次解析都会创建一个具有唯一内存引用的全新数据结构。如果管理不当，这种不断的序列化和解析会导致意外的性能问题，尤其是在像 React 这样通过不可变性追踪响应式，或像 Solid 这样依赖对账（reconciliation）检测变化的框架中。
- **独立性**：查询参数虽然是 URL 的一部分，但经常独立于路径名变化。例如，用户可能只想更改分页列表的页码，而不触碰 URL 路径。

---

## 查询参数：状态管理器的“元老”

您可能在 URL 中见过像 `?page=3` 或 `?filter-name=tanner` 这样的参数。毫无疑问，这确实是存在于 URL 中的一种**全局状态**。在 URL 中存储特定状态非常有价值，因为：

- **用户可以**：
  - 通过 `Cmd/Ctrl + 点击` 在新标签页中打开链接，并可靠地看到预期的状态。
  - 将应用链接加入书签并分享给他人，确信对方看到的页面状态与分享时完全一致。
  - 刷新应用或在页面间前后跳转而不会丢失状态。
- **开发者可以**：
  - 像操作其他状态管理器一样，轻松添加、移除或修改 URL 状态。
  - 轻松校验来自 URL 的查询参数，确保其格式和类型安全，便于应用消费。
  - 读写查询参数而无需担心底层的序列化格式。

---

## JSON 优先的查询参数

为了实现上述目标，TanStack Router 内置的第一步是一个强大的查询参数解析器，它会自动将 URL 的查询字符串转换为结构化的 JSON。这意味着您可以在查询参数中存储任何可 JSON 序列化的数据结构。

例如，导航到以下路由：

```tsx
const link = (
  <Link
    to="/shop"
    search={{
      pageIndex: 3,
      includeCategories: ["electronics", "gifts"],
      sortBy: "price",
      desc: true,
    }}
  />
);
```

将生成如下 URL：

```
/shop?pageIndex=3&includeCategories=%5B%22electronics%22%2C%22gifts%22%5D&sortBy=price&desc=true
```

当解析该 URL 时，查询参数将准确地转换回以下 JSON：

```json
{
  "pageIndex": 3,
  "includeCategories": ["electronics", "gifts"],
  "sortBy": "price",
  "desc": true
}
```

注意这里发生的几件事：

- 第一层查询参数是扁平且基于字符串的，与 `URLSearchParams` 兼容。
- 第一层中非字符串的值被准确地保留为数字和布尔值。
- **嵌套数据结构**会自动转换为 URL 安全的 JSON 字符串。

> 🧠 其他工具通常假设查询参数始终是扁平的字符串。通过在第一层保持兼容性，即使 TanStack Router 将嵌套参数管理为 JSON，其他工具仍能正常读写第一层参数。

---

## 校验和类型化查询参数

尽管 TanStack Router 能将查询参数解析为可靠的 JSON，但它们终究来自**用户可见的原始文本输入**。这意味着在消费它们之前，应该将其校验为应用可以信任的格式。

### 开启校验 + TypeScript！

TanStack Router 通过路由的 `validateSearch` 选项提供了便捷的 API：

```tsx title="src/routes/shop/products.tsx"
type ProductSearchSortOptions = "newest" | "oldest" | "price";

type ProductSearch = {
  page: number;
  filter: string;
  sort: ProductSearchSortOptions;
};

export const Route = createFileRoute("/shop/products")({
  validateSearch: (search: Record<string, unknown>): ProductSearch => {
    // 将查询参数校验并解析为类型化的状态
    return {
      page: Number(search?.page ?? 1),
      filter: (search.filter as string) || "",
      sort: (search.sort as ProductSearchSortOptions) || "newest",
    };
  },
});
```

在该示例中，我们校验了路由的查询参数并返回了一个类型化的 `ProductSearch` 对象。该对象随后可用于该路由的其他选项，**以及任何子路由**！

### 校验查询参数

`validateSearch` 选项是一个函数，它接收 JSON 解析后的（但未校验的）`Record<string, unknown>`，并返回您选择的类型化对象。通常建议为格式错误或意外的参数提供合理的**默认回退值**，以确保用户体验不中断。

以下是使用 [Zod](https://zod.dev/) 库进行一步到位校验和类型化的示例：

```tsx title="src/routes/shop/products.tsx"
import { z } from "zod";

const productSearchSchema = z.object({
  page: z.number().catch(1),
  filter: z.string().catch(""),
  sort: z.enum(["newest", "oldest", "price"]).catch("newest"),
});

type ProductSearch = z.infer<typeof productSearchSchema>;

export const Route = createFileRoute("/shop/products")({
  validateSearch: (search) => productSearchSchema.parse(search),
});
```

由于 `validateSearch` 也接受具有 `parse` 属性的对象，因此可以简写为：

```tsx
validateSearch: productSearchSchema;
```

> [!TIP]
> 在上面的例子中，我们使用了 Zod 的 `.catch()` 而不是 `.default()`。这是因为如果 URL 参数格式错误，我们通常不希望向用户展示巨大的错误信息，而是静默回退到默认值。

如果 `validateSearch` 函数抛出错误，路由的 `onError` 将被触发（`error.routerCode` 为 `VALIDATE_SEARCH`），并且会渲染 `errorComponent`。

#### 适配器 (Adapters)

使用 Zod 等库时，您可能希望在将参数写入 URL 之前对其进行 `transform`。此时建议使用**适配器**，它们可以正确推断 `input`（写入时所需）和 `output`（解析后所得）类型。

如果您发现导航到某个路由时提示 `search` 必填（如下方的 `Link`），通常是因为校验模式中定义了必需的输出字段。

```tsx
<Link to="/shop/products" /> // 如果没有适配器处理默认值，此处可能会报错
```

### Zod

我们为 [Zod](https://zod.dev/) 提供了一个适配器，它可以自动处理并传递正确的 `input`（输入）类型和 `output`（输出）类型：

```tsx
import { zodValidator } from "@tanstack/zod-adapter";
import { z } from "zod";

const productSearchSchema = z.object({
  page: z.number().default(1),
  filter: z.string().default(""),
  sort: z.enum(["newest", "oldest", "price"]).default("newest"),
});

export const Route = createFileRoute("/shop/products/")({
  validateSearch: zodValidator(productSearchSchema),
});
```

使用适配器的关键好处在于：在使用 `Link` 组件导航时，由于有了默认值，`search` 参数不再是必填项：

```tsx
// 现在这是合法的，不会有类型错误
<Link to="/shop/products" />
```

然而，在 Zod 中直接使用 `catch` 会覆盖原始类型，导致字段变为 `unknown`，从而引发类型丢失。为了解决这个问题，我们提供了一个 `fallback` 泛型函数，它能在验证失败时提供“兜底值（fallback）”，同时保留精确的类型定义：

```tsx
import { fallback, zodValidator } from "@tanstack/zod-adapter";
import { z } from "zod";

const productSearchSchema = z.object({
  // 使用 fallback 确保类型不丢失，同时提供兜底值
  page: fallback(z.number(), 1).default(1),
  filter: fallback(z.string(), "").default(""),
  sort: fallback(z.enum(["newest", "oldest", "price"]), "newest").default(
    "newest",
  ),
});

export const Route = createFileRoute("/shop/products/")({
  validateSearch: zodValidator(productSearchSchema),
});
```

这样，在导航到该路由时，`search` 是可选的，并且在读取参数时依然能保持正确的类型。

虽然不推荐，但你也可以手动配置 `input` 和 `output` 的推断逻辑，以应对输出类型比输入类型更准确的情况：

```tsx
export const Route = createFileRoute("/shop/products/")({
  validateSearch: zodValidator({
    schema: productSearchSchema,
    input: "output",
    output: "input",
  }),
});
```

这为你在导航时推断哪种类型以及在读取参数时推断哪种类型提供了极大的灵活性。

---

### Valibot

> [!WARNING]
> 路由器要求安装 valibot 1.0 或更高版本。

使用 [Valibot](https://valibot.dev/) 时，**不需要**额外的适配器来确保类型正确。这是因为 `valibot` 已经实现了 [Standard Schema（标准模式）](https://github.com/standard-schema/standard-schema) 协议，TanStack Router 可以直接识别它。

```tsx
import * as v from "valibot";

const productSearchSchema = v.object({
  page: v.optional(v.fallback(v.number(), 1), 1),
  filter: v.optional(v.fallback(v.string(), ""), ""),
  sort: v.optional(
    v.fallback(v.picklist(["newest", "oldest", "price"]), "newest"),
    "newest",
  ),
});

export const Route = createFileRoute("/shop/products/")({
  validateSearch: productSearchSchema,
});
```

---

### ArkType

> [!WARNING]
> 路由器要求安装 arktype 2.0-rc 或更高版本。

同样地，使用 [ArkType](https://arktype.io/) 时也不需要适配器，因为它也实现了 [Standard Schema](https://github.com/standard-schema/standard-schema) 协议。

```tsx
import { type } from "arktype";

const productSearchSchema = type({
  page: "number = 1",
  filter: 'string = ""',
  sort: '"newest" | "oldest" | "price" = "newest"',
});

export const Route = createFileRoute("/shop/products/")({
  validateSearch: productSearchSchema,
});
```

---

### Effect/Schema

使用 [Effect/Schema](https://effect.website/docs/schema/introduction/) 时，由于它同样支持 [Standard Schema](https://effect.website/docs/schema/standard-schema/)，你可以直接在路由配置中使用它：

```tsx
import { Schema as S } from "effect";

const productSearchSchema = S.standardSchemaV1(
  S.Struct({
    page: S.NumberFromString.pipe(
      S.optional,
      S.withDefaults({
        constructor: () => 1,
        decoding: () => 1,
      }),
    ),
    filter: S.String.pipe(
      S.optional,
      S.withDefaults({
        constructor: () => "",
        decoding: () => "",
      }),
    ),
    sort: S.Literal("newest", "oldest", "price").pipe(
      S.optional,
      S.withDefaults({
        constructor: () => "newest" as const,
        decoding: () => "newest" as const,
      }),
    ),
  }),
);

export const Route = createFileRoute("/shop/products/")({
  validateSearch: productSearchSchema,
});
```

## 读取查询参数 (Reading Search Params)

一旦您的查询参数经过验证并完成了类型化，就可以开始读取和写入它们了。TanStack Router 提供了几种方式来实现这一点，让我们一起来看看。

### 在加载器 (Loaders) 中使用查询参数

请阅读 [加载器中的查询参数](./data-loading.md) 章节，了解更多关于如何通过 `loaderDeps` 选项在加载器中读取查询参数的信息。

### 查询参数会从父级路由继承

随着您向下遍历路由树，父级的查询参数和类型会进行合并。因此，子路由也可以访问其父路由的查询参数：

```tsx title="src/routes/shop/products.tsx"
const productSearchSchema = z.object({
  page: z.number().catch(1),
  filter: z.string().catch(""),
  sort: z.enum(["newest", "oldest", "price"]).catch("newest"),
});

type ProductSearch = z.infer<typeof productSearchSchema>;

export const Route = createFileRoute("/shop/products")({
  validateSearch: productSearchSchema,
});
```

```tsx title="src/routes/shop/products/$productId.tsx"
export const Route = createFileRoute("/shop/products/$productId")({
  beforeLoad: ({ search }) => {
    // 这里的 search 会自动继承父级的类型
    search;
    // ^? ProductSearch ✅
  },
});
```

---

### 在组件中使用查询参数

您可以在路由的 `component` 中通过 `useSearch` 钩子访问该路由验证过的查询参数。

```tsx title="src/routes/shop/products.tsx"
export const Route = createFileRoute("/shop/products")({
  validateSearch: productSearchSchema,
});

const ProductList = () => {
  // 这里的类型是自动推导且安全的
  const { page, filter, sort } = Route.useSearch();

  return <div>...</div>;
};
```

> [!TIP]
> 如果您的组件进行了代码分割（code-split），可以使用 [getRouteApi 函数](./code-splitting.md) 来获取类型化的 `useSearch()` 钩子，这样就无需为了类型而导入整个 `Route` 配置文件。

---

### 在路由组件外部使用查询参数

您可以在应用中的任何位置使用 `useSearch` 钩子访问路由验证过的查询参数。通过传入起点路由的 `from` ID 或路径，您可以获得更好的类型安全：

```tsx
// src/routes/shop.products.tsx
export const Route = createFileRoute("/shop/products")({
  validateSearch: productSearchSchema,
  // ...
});

// 在其他地方...

// src/components/product-list-sidebar.tsx
const routeApi = getRouteApi("/shop/products");

const ProductList = () => {
  // 方案 A：使用 routeApi 访问
  const routeSearch = routeApi.useSearch();

  // 或者方案 B：在全局 useSearch 中指定 from
  const { page, filter, sort } = useSearch({
    from: Route.fullPath,
  });

  return <div>...</div>;
};
```

或者，您可以放宽类型安全性要求，通过传入 `strict: false` 来获取一个可选的 `search` 对象（所有字段可能为 `undefined`）：

```tsx
function ProductList() {
  const search = useSearch({
    strict: false,
  });
  // 此时类型会变为：
  // {
  //   page: number | undefined
  //   filter: string | undefined
  //   sort: 'newest' | 'oldest' | 'price' | undefined
  // }

  return <div>...</div>;
}
```

# 写入查询参数 (Writing Search Params)

既然您已经学习了如何读取路由的查询参数，您会很高兴地发现，其实您已经见过用于修改和更新它们的主要 API 了。让我们来温习一下。

## `<Link search />`

更新查询参数的最佳方式是使用 `<Link />` 组件上的 `search` 属性。

如果要更新当前页面的查询参数并指定了 `from` 属性，则可以省略 `to` 属性。
示例如下：

```tsx title="src/routes/shop/products.tsx"
export const Route = createFileRoute("/shop/products")({
  validateSearch: productSearchSchema,
});

const ProductList = () => {
  return (
    <div>
      {/* 这里的 from 锁定了类型安全，search 函数接收旧值并返回更新后的值 */}
      <Link from={Route.fullPath} search={(prev) => ({ page: prev.page + 1 })}>
        下一页
      </Link>
    </div>
  );
};
```

如果您想在一个渲染在多个路由上的**通用组件**中更新查询参数，指定 `from` 可能会比较困难。

在这种场景下，您可以设置 `to="."`，这将允许您访问松散类型的查询参数（Loosely typed）。
以下示例说明了这一点：

```tsx
// `page` 是在 __root 路由中定义的查询参数，因此在所有路由上都可用。
const PageSelector = () => {
  return (
    <div>
      <Link to="." search={(prev) => ({ ...prev, page: prev.page + 1 })}>
        下一页
      </Link>
    </div>
  );
};
```

如果通用组件仅在路由树的特定子树中渲染，您可以使用 `from` 指定该子树。在这里，如果您愿意，也可以省略 `to='.'`。

```tsx
// `page` 是在 /posts 路由中定义的查询参数，因此在其所有子路由上都可用。
const PageSelector = () => {
  return (
    <div>
      <Link
        from="/posts"
        to="."
        search={(prev) => ({ ...prev, page: (prev.page ?? 0) + 1 })}
      >
        下一页
      </Link>
    </div>
  );
};
```

---

## 命令式导航 API

### `useNavigate()`, `Navigate({ search })`

`Navigate` 函数也接受 `search` 选项，其工作方式与 `<Link />` 上的 `search` 属性完全相同：

```tsx title="src/routes/shop/products.tsx"
export const Route = createFileRoute("/shop/products/$productId")({
  validateSearch: productSearchSchema,
});

const ProductList = () => {
  const navigate = useNavigate({ from: Route.fullPath });

  return (
    <div>
      <button
        onClick={() => {
          navigate({
            search: (prev) => ({ page: prev.page + 1 }),
          });
        }}
      >
        下一页
      </button>
    </div>
  );
};
```

### `router.navigate({ search })`

`router.navigate` 函数的工作方式与上述 `useNavigate`/`Navigate` 钩子/函数完全相同。

### `<Navigate search />` 组件

`<Navigate search />` 组件的工作方式也与上述 API 一致，但它通过 Props 接受选项，而不是函数参数。

---

## 使用查询参数中间件进行转换

默认情况下，当构建链接的 `href` 时，唯一影响查询字符串部分的是 `<Link>` 的 `search` 属性。

TanStack Router 提供了一种在生成 `href` 之前通过 **查询参数中间件 (Search Middlewares)** 操纵参数的方法。
中间件是在为路由或其后代生成新链接时转换查询参数的函数。它们也会在导航发生、查询参数通过验证后执行，以便操纵最终的查询字符串。

### 保留参数：`retainSearchParams`

以下示例展示了如何确保对于**每一个**构建的链接，如果当前参数中存在 `rootValue`，则自动将其添加进去。如果链接显式指定了 `rootValue`，则使用链接中的值。

```tsx
import { z } from "zod";
import { zodValidator } from "@tanstack/zod-adapter";

const searchSchema = z.object({
  rootValue: z.string().optional(),
});

export const Route = createRootRoute({
  validateSearch: zodValidator(searchSchema),
  search: {
    middlewares: [
      ({ search, next }) => {
        const result = next(search);
        return {
          rootValue: search.rootValue,
          ...result,
        };
      },
    ],
  },
});
```

由于这种“保留参数”的需求非常普遍，TanStack Router 提供了一个内置实现：`retainSearchParams`。

```tsx
import { z } from "zod";
import { createFileRoute, retainSearchParams } from "@tanstack/react-router";
import { zodValidator } from "@tanstack/zod-adapter";

const searchSchema = z.object({
  rootValue: z.string().optional(),
});

export const Route = createRootRoute({
  validateSearch: zodValidator(searchSchema),
  search: {
    middlewares: [retainSearchParams(["rootValue"])],
  },
});
```

### 剔除默认值：`stripSearchParams`

另一个常见的用例是：如果查询参数的值等于其默认值，则从链接中将其剔除，以保持 URL 简洁。

```tsx
import { z } from "zod";
import { createFileRoute, stripSearchParams } from "@tanstack/react-router";
import { zodValidator } from "@tanstack/zod-adapter";

const defaultValues = {
  one: "abc",
  two: "xyz",
};

const searchSchema = z.object({
  one: z.string().default(defaultValues.one),
  two: z.string().default(defaultValues.two),
});

export const Route = createFileRoute("/hello")({
  validateSearch: zodValidator(searchSchema),
  search: {
    // 剔除默认值，如果 one 是 'abc'，URL 中将不显示 ?one=abc
    middlewares: [stripSearchParams(defaultValues)],
  },
});
```

### 链式调用多个中间件

您可以组合使用多个中间件。以下示例展示了如何同时使用 `retainSearchParams` 和 `stripSearchParams`。

```tsx
import {
  Link,
  createFileRoute,
  retainSearchParams,
  stripSearchParams,
} from "@tanstack/react-router";
import { z } from "zod";
import { zodValidator } from "@tanstack/zod-adapter";

const defaultValues = ["foo", "bar"];

export const Route = createFileRoute("/search")({
  validateSearch: zodValidator(
    z.object({
      retainMe: z.string().optional(),
      arrayWithDefaults: z.string().array().default(defaultValues),
      required: z.string(),
    }),
  ),
  search: {
    middlewares: [
      retainSearchParams(["retainMe"]),
      stripSearchParams({ arrayWithDefaults: defaultValues }),
    ],
  },
});
```
