---
short_title: 导航
---

# 导航 (Navigation)

## 一切皆相对 (Everything is Relative)

信不信由你，应用内的每一次导航都是**相对**的，即使你没有使用显式的相对路径语法（如 `../../somewhere`）。每当点击链接或进行命令式导航调用时，总会有一个**起点（Origin）**路径和一个**目标（Destination）**路径，这意味着你正在**从（from）**一个路由导航**到（to）**另一个路由。

TanStack Router 在每次导航中都秉持这种“相对导航”的恒定概念，因此你会在 API 中经常看到两个属性：

- `from`：起点路由路径
- `to`：目标路由路径

> ⚠️ 如果没有提供 `from` 路由路径，路由器会假设你正从根路由 `/` 开始导航，并且此时仅能对绝对路径进行自动补全。毕竟，只有知道自己“从哪来”，才能精准地知道要“到哪去” 😉。

---

## 统一的导航 API (Shared Navigation API)

TanStack Router 中的每个导航和路由匹配 API 都使用相同的核心接口，仅根据 API 的用途有细微差别。这意味着你只需学习一次导航和路由匹配的逻辑，就可以在整个库中复用相同的语法和概念。

### `ToOptions` 接口

这是在每个导航和路由匹配 API 中都会用到的核心 `ToOptions` 接口：

```ts
type ToOptions<
  TRouteTree extends AnyRoute = AnyRoute,
  TFrom extends RoutePaths<TRouteTree> | string = string,
  TTo extends string = "",
> = {
  // `from` 是可选的路由 ID 或路径。
  // 如果不提供，则只有绝对路径会获得自动补全和类型安全。
  // 为了方便，通常建议提供你当前正在渲染的起点路由的 route.fullPath。
  // 如果你不确定起点路由，请留空并使用绝对路径或不安全的相对路径。
  from?: string;

  // `to` 可以是绝对路由路径，也可以是相对于 `from` 选项的有效路由路径。
  // ⚠️ 不要将路径参数（path params）、hash 或查询参数（search params）手动拼接进 `to` 字符串中。
  // 请使用专门的 `params`、`search` 和 `hash` 选项。
  to: string;

  // `params` 可以是一个路径参数对象，用于插入到 `to` 路径中；
  // 也可以是一个接收旧参数并返回新参数的函数。
  // 这是将动态参数插入最终 URL 的唯一方式。
  // 根据 `from` 和 `to` 的不同，你可能需要提供部分或全部路径参数。如果有缺失，TypeScript 会提醒你。
  params:
    | Record<string, unknown>
    | ((prevParams: Record<string, unknown>) => Record<string, unknown>);

  // `search` 可以是一个查询参数对象，也可以是一个接收旧搜索参数并返回新参数的函数。
  // 根据目标路由的不同，你可能需要提供特定的查询参数。TypeScript 会进行校验。
  search:
    | Record<string, unknown>
    | ((prevSearch: Record<string, unknown>) => Record<string, unknown>);

  // `hash` 可以是一个字符串，也可以是一个处理旧 hash 并返回新值的函数。
  hash?: string | ((prevHash: string) => string);

  // `state` 可以是一个状态对象或处理函数。
  // 状态存储在浏览器 History API 中，适合在路由间传递不想永久存在于 URL 中的数据。
  state?:
    | Record<string, any>
    | ((prevState: Record<string, unknown>) => Record<string, unknown>);

  // `mask` 是另一个导航对象，用于“遮罩”浏览器地址栏显示的 URL。
  mask?: ToMaskOptions<TRouteTree>;
};

type ToMaskOptions<TRouteTree extends AnyRoute = AnyRoute> = {
  // `from`, `to`, `params`, `search`, `hash`, 和 `state` 的行为与 ToOptions 一致。
  // `mask` 内部不允许再嵌套 `mask`。
  from?: string;
  to: string;
  params:
    | Record<string, unknown>
    | ((prevParams: Record<string, unknown>) => Record<string, unknown>);
  search:
    | Record<string, unknown>
    | ((prevSearch: Record<string, unknown>) => Record<string, unknown>);
  hash?: string | ((prevHash: string) => string);
  state?:
    | Record<string, any>
    | ((prevState: Record<string, unknown>) => Record<string, unknown>);

  // 如果为 true，URL 将在页面重载时解除遮罩。
  unmaskOnReload?: boolean;
};
```

> 🧠 每个路由对象都有一个 `to` 属性，它可以作为任何导航 API 的 `to` 参数。尽可能使用它来代替纯字符串，以实现类型安全的路由引用：

```tsx
import { Route as aboutRoute } from "./routes/about.tsx";

function Comp() {
  // 使用 aboutRoute.to 确保路径永远正确
  return <Link to={aboutRoute.to}>关于我们</Link>;
}
```

---

### `NavigateOptions` 接口

这是继承自 `ToOptions` 的核心 `NavigateOptions` 接口。任何实际执行导航操作的 API 都会使用这个接口：

```ts
export type NavigateOptions<
  TRouteTree extends AnyRoute = AnyRoute,
  TFrom extends RoutePaths<TRouteTree> | string = string,
  TTo extends string = "",
> = ToOptions<TRouteTree, TFrom, TTo> & {
  // `replace` 为 true 时，导航将替换当前历史记录条目，而不是推送新条目。
  replace?: boolean;

  // `resetScroll` 为 true 时，位置提交到历史记录后，滚动位置将重置为 (0,0)。
  resetScroll?: boolean;

  // `hashScrollIntoView` 决定是否在位置提交后，自动滚动到与 hash 匹配的 ID 元素。
  hashScrollIntoView?: boolean | ScrollIntoViewOptions;

  // `viewTransition` 决定浏览器在导航时是否及如何调用 document.startViewTransition()。
  viewTransition?: boolean | ViewTransitionOptions;

  // `ignoreBlocker` 为 true 时，导航将忽略任何可能阻止它的拦截器（blockers）。
  ignoreBlocker?: boolean;

  // `reloadDocument` 为 true 时，导航将触发完整的页面加载，而不是传统的 SPA 导航。
  reloadDocument?: boolean;

  // `href` 可以替代 `to` 使用，用于导航到一个完整的生成的链接，例如指向外部目标。
  href?: string;
};
```

`NavigateOptions` 包含了 `ToOptions` 的所有字段，包括 `mask`。

---

### `LinkOptions` 接口

任何实际渲染 `<a>` 标签的地方，都会提供扩展自 `NavigateOptions` 的 `LinkOptions` 接口：

```tsx
export type LinkOptions<
  TRouteTree extends AnyRoute = AnyRoute,
  TFrom extends RoutePaths<TRouteTree> | string = string,
  TTo extends string = "",
> = NavigateOptions<TRouteTree, TFrom, TTo> & {
  // 标准 anchor 标签的 target 属性
  target?: HTMLAnchorElement["target"];

  // 链接激活时的选项，默认为 `{ exact: false, includeHash: false }`
  activeOptions?: {
    exact?: boolean;
    includeHash?: boolean;
    includeSearch?: boolean;
    explicitUndefined?: boolean;
  };

  // 如果设置，将在悬停时预加载（preload）目标路由并缓存指定毫秒数。
  preload?: false | "intent";

  // 延迟“意图预加载（intent preloading）”的毫秒数。如果在延迟结束前鼠标移出，预加载将被取消。
  preloadDelay?: number;

  // 如果为 true，将渲染不带 href 属性的链接
  disabled?: boolean;
};
```

由于 `LinkOptions` 继承了 `NavigateOptions`，它同样支持 `mask` 功能。

## 导航 API (Navigation API)

在理解了相对导航和各种接口之后，我们来看看你可以使用的几种不同形式的导航 API：

- **`<Link>` 组件**
  - 生成一个真实的 `<a>` 标签，带有有效的 `href`。可以点击，甚至支持 `cmd/ctrl + 点击` 在新标签页中打开。
- **`useNavigate()` Hook**
  - 只要可能，就应该优先使用 `Link` 组件进行导航。但有时你需要作为“副作用”的结果进行命令式导航。`useNavigate` 返回一个函数，调用它即可立即执行客户端导航。
- **`<Navigate>` 组件**
  - 不渲染任何内容，并在挂载时立即执行客户端导航。
- **`Router.navigate()` 方法**
  - 这是 TanStack Router 中最强大的导航 API。与 `useNavigate` 类似，它执行命令式导航，但只要你能访问到 `router` 实例，它在任何地方都可用。

> ⚠️ 这些 API 都**不能**替代服务器端重定向。如果你需要在挂载应用之前立即将用户从一个路由重定向到另一个路由，请使用服务器端重定向，而不是客户端导航。

---

## `<Link>` 组件

`Link` 组件是应用内最常用的导航方式。它渲染一个带有有效 `href` 属性的 `<a>` 标签。

除了 [`LinkOptions`](linkoptions-interface) 接口外，`Link` 组件还支持以下 Props：

```tsx
export type LinkProps<
  TFrom extends RoutePaths<RegisteredRouter["routeTree"]> | string = string,
  TTo extends string = "",
> = LinkOptions<RegisteredRouter["routeTree"], TFrom, TTo> & {
  // 一个返回该链接“激活 (active)”状态额外 Props 的函数。
  // 这些 Props 会覆盖传递给 Link 的其他 Props（style 会合并，className 会拼接）。
  activeProps?:
    | FrameworkHTMLAnchorTagAttributes
    | (() => FrameworkHTMLAnchorAttributes);
  // 一个返回该链接“非激活 (inactive)”状态额外 Props 的函数。
  inactiveProps?:
    | FrameworkHTMLAnchorAttributes
    | (() => FrameworkHTMLAnchorAttributes);
};
```

### 绝对链接 (Absolute Links)

让我们创建一个简单的静态链接：

```tsx
import { Link } from "@tanstack/react-router";

const link = <Link to="/about">关于我们</Link>;
```

### 动态链接 (Dynamic Links)

动态链接是包含动态段（dynamic segments）的链接。例如，指向博客文章的链接：

```tsx
const link = (
  <Link
    to="/blog/post/$postId"
    params={{
      postId: "my-first-blog-post",
    }}
  >
    博客文章
  </Link>
);
```

请记住，通常动态段参数是 `string` 类型，但它们也可以是你路由选项中解析出的任何其他类型。无论哪种方式，类型都会在编译时进行检查。

### 相对链接 (Relative Links)

默认情况下，除非提供了 `from` 路由路径，否则所有链接都是绝对路径。这意味着无论你当前处于哪个路由，上述链接都会导航到 `/about`。

相对链接可以与 `from` 路由路径结合使用。如果未提供 `from`，相对路径默认指向当前激活的位置。

> [!NOTE]
> 请记住，当把 `useNavigate` 作为路由上的方法调用时（例如 `Route.useNavigate`），`from` 位置已预定义为调用它的那个路由。
>
> 另一个常见的坑是在**无路径布局路由 (Pathless Layout Route)** 中使用。由于无路径布局路由没有实际路径，`from` 位置被视为该布局路由的父级。因此，相对路由将从该父级开始解析。

```tsx
const postIdRoute = createRoute({
  path: "/blog/post/$postId",
});

const link = (
  <Link from={postIdRoute.fullPath} to="../categories">
    分类
  </Link>
);
```

如上所示，通常使用 `route.fullPath` 作为 `from` 路径。这是因为 `route.fullPath` 是一个引用，如果你重构应用，它会自动更新。当然，直接提供字符串路径也是可以的，类型检查依然有效。

### 特殊相对路径：`"."` 和 `".."`

你可能经常想要重新加载当前位置，例如重新运行当前或父级路由的 Loaders。这可以通过指定 `to="."` 来实现。

另一个常见的需求是导航到当前位置或另一个路径的上一级。通过指定 `to=".."`，导航将解析到当前位置之前的第一个父路由。

```tsx
export const Route = createFileRoute("/posts/$postId")({
  component: PostComponent,
});

function PostComponent() {
  return (
    <div>
      <Link to=".">重新加载当前路由 /posts/$postId</Link>
      <Link to="..">返回到 /posts</Link>

      {/* 以下方式效果等价 */}
      <Link to="/posts">返回到 /posts</Link>
      <Link from="/posts" to=".">
        返回到 /posts
      </Link>

      {/* 以下方式效果等价 */}
      <Link to="/">回到根目录</Link>
      <Link from="/posts" to="..">
        回到根目录
      </Link>
    </div>
  );
}
```

### 查询参数链接 (Search Param Links)

查询参数是为路由提供额外上下文的好方法。

```tsx
const link = (
  <Link
    to="/search"
    search={{
      query: "tanstack",
    }}
  >
    搜索
  </Link>
);
```

你也可以在不提供其他信息的情况下更新单个查询参数（例如翻页）：

```tsx
const link = (
  <Link
    to="."
    search={(prev) => ({
      ...prev,
      page: prev.page + 1,
    })}
  >
    下一页
  </Link>
);
```

### 查询参数类型安全 (Search Param Type Safety)

查询参数是一种高度动态的状态管理机制，因此确保传递正确的类型至关重要。我们将在后面的章节详细讨论如何验证并确保查询参数的类型安全。

### Hash 链接 (Hash Links)

Hash 链接是链接到页面特定部分的绝佳方式：

```tsx
const link = (
  <Link
    to="/blog/post/$postId"
    params={{
      postId: "my-first-blog-post",
    }}
    hash="section-1"
  >
    第一节
  </Link>
);
```

> ⚠️ 当直接导航到带有 Hash 片段的 URL 时，该片段仅在客户端可用；浏览器不会将片段作为请求 URL 的一部分发送给服务器。
>
> 这意味着如果你使用**服务器端渲染 (SSR)**，Hash 片段在服务器端是不可见的。如果在渲染标记时使用了 Hash（例如根据 Hash 设置 Link 激活态或条件渲染），可能会发生**水合不匹配 (Hydration Mismatches)**。

### 使用可选参数进行导航 (Navigating with Optional Parameters)

可选路径参数提供了灵活的导航模式，让您可以根据需要包含或忽略某些参数。可选参数使用 `{-$paramName}` 语法，并提供对 URL 结构的精细控制。

#### 参数继承 vs 移除 (Parameter Inheritance vs Removal)

在使用可选参数进行导航时，您有两种主要策略：

**1. 继承当前参数**
使用 `params: {}` 来继承所有当前路由已有的参数：

```tsx
// 继承当前路由的所有参数
<Link to="/posts/{-$category}" params={{}}>
  所有帖子
</Link>
```

**2. 移除参数**
将参数设置为 `undefined` 以明确地将其从 URL 中移除：

```tsx
// 移除 category 参数
<Link to="/posts/{-$category}" params={{ category: undefined }}>
  所有帖子
</Link>
```

---

#### 基础可选参数导航示例

```tsx
// 带可选参数导航 (URL: /posts/tech)
<Link
  to="/posts/{-$category}"
  params={{ category: 'tech' }}
>
  技术类帖子
</Link>

// 不带可选参数导航 (URL: /posts)
<Link
  to="/posts/{-$category}"
  params={{ category: undefined }}
>
  所有帖子
</Link>

// 使用参数继承 (保持当前分类不变)
<Link
  to="/posts/{-$category}"
  params={{}}
>
  当前分类
</Link>
```

---

#### 函数式参数更新 (Function-Style Parameter Updates)

在处理可选参数时，函数式更新特别有用，因为它可以让你基于当前状态进行逻辑判断：

```tsx
// 使用函数语法移除参数
<Link
  to="/posts/{-$category}"
  params={(prev) => ({ ...prev, category: undefined })}
>
  清除分类
</Link>

// 更新一个参数同时保留其他参数
<Link
  to="/articles/{-$category}/{-$slug}"
  params={(prev) => ({ ...prev, category: 'news' })}
>
  新闻文章
</Link>

// 根据条件设置参数
<Link
  to="/posts/{-$category}"
  params={(prev) => ({
    ...prev,
    category: someCondition ? 'tech' : undefined
  })}
>
  条件分类
</Link>
```

---

#### 多个可选参数 (Multiple Optional Parameters)

当处理多个可选参数时，您可以自由组合哪些需要包含，哪些需要忽略：

```tsx
// 仅导航至部分可选参数 (URL: /posts/tech)
<Link
  to="/posts/{-$category}/{-$slug}"
  params={{ category: 'tech', slug: undefined }}
>
  技术帖子列表
</Link>

// 移除所有可选参数 (URL: /posts)
<Link
  to="/posts/{-$category}/{-$slug}"
  params={{ category: undefined, slug: undefined }}
>
  回到全部
</Link>

// 设置所有参数 (URL: /posts/tech/react-tips)
<Link
  to="/posts/{-$category}/{-$slug}"
  params={{ category: 'tech', slug: 'react-tips' }}
>
  特定帖子
</Link>
```

---

#### 混合必选参数与可选参数

可选参数可以与必选参数无缝协作：

```tsx
// 'id' 是必选的，'tab' 是可选的
<Link
  to="/users/$id/{-$tab}"
  params={{ id: '123', tab: 'settings' }}
>
  用户设置
</Link>

// 保留必选参数 'id'，但移除可选参数 'tab'
<Link
  to="/users/$id/{-$tab}"
  params={{ id: '123', tab: undefined }}
>
  用户资料
</Link>

// 对混合参数使用函数式更新
<Link
  to="/users/$id/{-$tab}"
  params={(prev) => ({ ...prev, tab: 'notifications' })}
>
  用户通知
</Link>
```

---

#### 高级可选参数模式

**1. 前缀和后缀参数**
带有前缀/后缀的可选参数同样支持导航：

```tsx
// 导航至带有可选名称的文件 (URL: /files/prefix-document.txt)
<Link
  to="/files/prefix{-$name}.txt"
  params={{ name: 'document' }}
>
  文档文件
</Link>

// 导航至不带可选名称的文件 (URL: /files/prefix.txt)
<Link
  to="/files/prefix{-$name}.txt"
  params={{ name: undefined }}
>
  默认文件
</Link>
```

**2. 全可选参数路由**
甚至可以让一个路由的所有参数都是可选的：

```tsx
// 导航至具体日期 (URL: /2023/12/25)
<Link
  to="/{-$year}/{-$month}/{-$day}"
  params={{ year: '2023', month: '12', day: '25' }}
>
  2023 圣诞节
</Link>

// 导航至部分日期 (URL: /2023/12)
<Link
  to="/{-$year}/{-$month}/{-$day}"
  params={{ year: '2023', month: '12', day: undefined }}
>
  2023 十二月
</Link>

// 移除所有参数回到根路径 (URL: /)
<Link
  to="/{-$year}/{-$month}/{-$day}"
  params={{ year: undefined, month: undefined, day: undefined }}
>
  首页
</Link>
```

---

#### 查询参数 (Search Params) 与可选参数结合使用

可选路径参数与查询参数配合起来非常完美：

```tsx
// 同时结合可选路径参数和查询参数 (URL: /posts/tech?page=1&sort=newest)
<Link
  to="/posts/{-$category}"
  params={{ category: 'tech' }}
  search={{ page: 1, sort: 'newest' }}
>
  技术帖子 - 第 1 页
</Link>

// 移除路径参数但保留查询参数
<Link
  to="/posts/{-$category}"
  params={{ category: undefined }}
  search={(prev) => prev}
>
  所有帖子 - 保持原有过滤
</Link>
```

#### 命令式导航中的可选参数 (Imperative Navigation with Optional Parameters)

所有的可选参数模式在命令式导航中同样适用：

```tsx
function Component() {
  const navigate = useNavigate();

  // 清除所有过滤器
  const clearFilters = () => {
    navigate({
      to: "/posts/{-$category}/{-$tag}",
      params: { category: undefined, tag: undefined },
    });
  };

  // 仅设置分类，保持其他参数不变
  const setCategory = (category: string) => {
    navigate({
      to: "/posts/{-$category}/{-$tag}",
      params: (prev) => ({ ...prev, category }),
    });
  };

  // 同时应用多个过滤器
  const applyFilters = (category?: string, tag?: string) => {
    navigate({
      to: "/posts/{-$category}/{-$tag}",
      params: { category, tag },
    });
  };
}
```

---

### 激活与非激活 Props (Active & Inactive Props)

`Link` 组件支持两个额外的 Props：`activeProps` 和 `inactiveProps`。这两个 Props 是返回对象的函数（或直接是对象），用于在链接处于“激活”或“非激活”状态时添加额外的属性。除了样式（style）和类名（className）会进行合并外，这里传递的其他属性都会覆盖原始传给 `Link` 的属性。

示例如下：

```tsx
const link = (
  <Link
    to="/blog/post/$postId"
    params={{
      postId: "my-first-blog-post",
    }}
    activeProps={{
      style: {
        fontWeight: "bold", // 激活时加粗
      },
    }}
  >
    第一节
  </Link>
);
```

---

### `data-status` 属性

除了上述 Props 外，`Link` 组件在激活状态下还会自动向渲染的元素添加一个 `data-status` 属性。该属性的值根据状态为 `"active"` 或 `undefined`。如果你更喜欢使用 CSS 数据属性（data-attributes）而不是通过 Props 来设置链接样式，这个特性会非常方便。

---

### 激活选项 (Active Options)

`Link` 组件带有一个 `activeOptions` 属性，提供了几种确定链接是否处于激活状态的配置：

```tsx
export interface ActiveOptions {
  // 如果为 true，则仅在当前路由与 `to` 路径完全匹配（没有子路由）时，链接才处于激活状态。
  // 默认为 `false`
  exact?: boolean;
  // 如果为 true，则仅在当前 URL 的 hash 与 `hash` 属性匹配时，链接才处于激活状态。
  // 默认为 `false`
  includeHash?: boolean;
  // 如果为 true，则仅在当前 URL 的查询参数包含并匹配 `search` 属性时，链接才处于激活状态。
  // 默认为 `true`
  includeSearch?: boolean;
  // 修正 `includeSearch` 的行为。
  // 如果为 true，在 `search` 中显式设为 `undefined` 的属性必须在当前 URL 中不存在，链接才算激活。
  // 默认为 `false`
  explicitUndefined?: boolean;
}
```

默认情况下，它会检查生成的 **pathname** 是否是当前路由的前缀。如果提供了任何查询参数（Search Params），它会检查它们是否*包含式地（inclusively）*匹配当前位置。默认不检查 Hash。

例如，如果你当前处于 `/blog/post/my-first-blog-post` 路由，以下链接都将被视为激活：

```tsx
// 路径完全匹配
const link1 = (
  <Link to="/blog/post/$postId" params={{ postId: "my-first-blog-post" }}>
    博客文章
  </Link>
);
// 前缀匹配 (父级路由)
const link2 = <Link to="/blog/post">博客文章</Link>;
const link3 = <Link to="/blog">博客文章</Link>;
```

然而，以下链接将**不会**被激活：

```tsx
const link4 = (
  <Link to="/blog/post/$postId" params={{ postId: "my-second-blog-post" }}>
    另一篇博客文章
  </Link>
);
```

通常，某些链接（如首页链接）只有在精确匹配时才应显示为激活态。在这种情况下，你可以传递 `exact: true`：

```tsx
const link = (
  <Link to="/" activeOptions={{ exact: true }}>
    首页
  </Link>
);
```

这样可以确保当你处于子路由时，该首页链接不会显示为激活。

此外还有：

- 如果你想在匹配时包含 Hash，可以设置 `includeHash: true`。
- 如果你**不想**在匹配时包含查询参数，可以设置 `includeSearch: false`。

---

### 向子元素传递 `isActive` 状态

`Link` 组件接受一个函数作为其 `children`，允许你将 `isActive` 属性传递给子元素。例如，你可以根据父链接是否激活来改变图标的样式：

```tsx
const link = (
  <Link to="/blog/post">
    {({ isActive }) => {
      return (
        <>
          <span>我的博客文章</span>
          <icon className={isActive ? "active" : "inactive"} />
        </>
      );
    }}
  </Link>
);
```

### 链接预加载 (Link Preloading)

`Link` 组件支持在用户表现出“意图”时（目前指悬停 hover 或触摸开始 touchstart）自动预加载路由。这可以在路由器选项中作为默认值配置（我们稍后会详细讨论），也可以通过向 `Link` 组件传递 `preload='intent'` 属性来单独开启。示例如下：

```tsx
const link = (
  <Link to="/blog/post/$postId" preload="intent">
    博客文章
  </Link>
);
```

通过启用预加载，配合相对较快的异步路由依赖（如果有的话），这个简单的小技巧可以用极小的代价显著提升应用程序的“感知性能”。

更棒的是，通过结合使用像 `@tanstack/query` 这样“缓存优先”的库，预加载的路由数据会保留在缓存中。如果用户稍后决定导航到该路由，就能直接享受“过时即重新验证”（stale-while-revalidate）的极致加载体验。

### 链接预加载延迟 (Link Preloading Delay)

与预加载配套的是一个可配置的延迟时间，它决定了用户必须在链接上悬停多久才会触发基于意图的预加载。默认延迟为 **50 毫秒**，但你可以通过向 `Link` 组件传递 `preloadDelay` 属性（单位为毫秒）来修改它：

```tsx
const link = (
  <Link to="/blog/post/$postId" preload="intent" preloadDelay={100}>
    博客文章
  </Link>
);
```

---

## `useNavigate` Hook

> ⚠️ 由于 `Link` 组件内置了对 `href` 的支持、支持 `cmd/ctrl + 点击` 以及激活/非激活状态的处理，建议对于任何用户可交互的元素（如链接、按钮），优先使用 `Link` 组件。然而，在某些情况下，`useNavigate` 对于处理由**副作用**引起的导航（例如异步操作成功后的跳转）是必不可少的。

`useNavigate` 钩子返回一个 `Navigate` 函数，可用于执行命令式导航。这是在副作用中跳转路由的绝佳方式。示例如下：

```tsx
function Component() {
  // 建议传入 from，这样在下面调用 navigate 时就不用重复写了
  const navigate = useNavigate({ from: "/posts/$postId" });

  const handleSubmit = async (e: FrameworkFormEvent) => {
    e.preventDefault();

    const response = await fetch("/posts", {
      method: "POST",
      body: JSON.stringify({ title: "我的第一篇文章" }),
    });

    const { id: postId } = await response.json();

    if (response.ok) {
      // 执行跳转
      navigate({ to: "/posts/$postId", params: { postId } });
    }
  };
}
```

> 🧠 如上所示，你可以在钩子调用时传递 `from` 选项。虽然在每次调用返回的 `Navigate` 函数时也可以传递，但在这里统一设置可以减少出错，还能少打几个字！

### `Navigate` 选项

`useNavigate` 返回的 `Navigate` 函数接受 [`NavigateOptions` 接口](navigateoptions-interface) 定义的所有参数。

---

## `Navigate` 组件

偶尔你可能需要在组件**挂载（Mount）**时立即执行导航。你的第一直觉可能是想用 `useNavigate` 配合一个立即执行的副作用（如 `useEffect`），但其实没必要。你可以直接渲染 `Navigate` 组件来实现同样的效果：

```tsx
function Component() {
  return <Navigate to="/posts/$postId" params={{ postId: "my-first-post" }} />;
}
```

你可以把 `Navigate` 组件看作是组件挂载即跳转的一种声明式方式。它是处理“仅限客户端”重定向的好帮手。但请记住，它**绝对不能**替代在服务器端进行的负责任的重定向处理。

---

## `router.navigate` 方法

`router.navigate` 方法与 `useNavigate` 返回的函数功能一致，也接受相同的 [`NavigateOptions` 接口](navigateoptions-interface)。与 `useNavigate` 钩子不同的是，只要你能访问到 `router` 实例，它在**任何地方**都可用，因此它是从应用任何角落（甚至是框架外部）执行命令式导航的利器。

---

## `useMatchRoute` 和 `<MatchRoute>`

`useMatchRoute` 钩子和 `<MatchRoute>` 组件功能相同，但钩子版本更灵活。它们都接受标准的导航 `ToOptions` 接口（作为选项或 Props），如果该路由当前匹配，则返回 `true/false`。

它还有一个非常有用的 `pending` 选项：如果路由当前处于**挂起/过渡**状态（例如用户正点击跳转到该路由但数据还没加载完），它会返回 `true`。这对于在导航目标位置显示“乐观 UI（Optimistic UI）”非常有用：

```tsx
function Component() {
  return (
    <div>
      <Link to="/users">
        用户列表
        {/* 当正在跳转到 /users 时显示加载动画 */}
        <MatchRoute to="/users" pending>
          <Spinner />
        </MatchRoute>
      </Link>
    </div>
  );
}
```

组件版 `<MatchRoute>` 也可以配合函数作为子元素（render props）使用：

```tsx
function Component() {
  return (
    <div>
      <Link to="/users">
        用户列表
        <MatchRoute to="/users" pending>
          {(match) => {
            return <Spinner show={match} />;
          }}
        </MatchRoute>
      </Link>
    </div>
  );
}
```

钩子版 `useMatchRoute` 返回一个函数，可以编程方式检查匹配状态：

```tsx
function Component() {
  const matchRoute = useMatchRoute();

  useEffect(() => {
    if (matchRoute({ to: "/users", pending: true })) {
      console.info("/users 路由已匹配且处于挂起状态");
    }
  });

  return (
    <div>
      <Link to="/users">用户列表</Link>
    </div>
  );
}
```

---

呼！导航的内容真不少。到这里，你应该已经能自如地在应用里“瞬移”了。
