---
short_title: 渲染优化
---

# 渲染优化 (Render Optimizations)

TanStack Router 包含多项优化措施，以确保你的组件仅在必要时重新渲染。这些优化包括：

## 结构共享 (Structural Sharing)

TanStack Router 使用一种称为“结构共享”的技术，在重新渲染之间尽可能多地保留引用。这对于存储在 URL 中的状态（如搜索参数）特别有用。

例如，假设有一个包含两个搜索参数 `foo` 和 `bar` 的 `details` 路由，访问方式如下：

```tsx
const search = Route.useSearch();
```

当通过导航从 `/details?foo=f1&bar=b1` 变为 `/details?foo=f1&bar=b2` 仅改变 `bar` 时，`search.foo` 的引用将保持稳定，只有 `search.bar` 会被替换。

## 细粒度选择器 (Fine-grained Selectors)

你可以使用 `useRouterState`、`useSearch` 等各种 Hook 来访问并订阅路由器状态。如果你只希望特定组件在路由器状态的特定子集（例如搜索参数的一个子集）发生变化时才重新渲染，可以使用 `select` 属性进行部分订阅。

```tsx
// 当 `bar` 变化时，该组件不会重新渲染
const foo = Route.useSearch({ select: ({ foo }) => foo });
```

### 结合细粒度选择器的结构共享

`select` 函数可以对路由器状态执行各种计算，允许你返回不同类型的值（如对象）。例如：

```tsx
const result = Route.useSearch({
  select: (search) => {
    return {
      foo: search.foo,
      hello: `hello ${search.foo}`,
    };
  },
});
```

虽然这段代码可以运行，但会导致你的组件每次都重新渲染，因为 `select` 现在每次被调用时都会返回一个新对象。

你可以通过使用上述的“结构共享”来避免这个重新渲染问题。默认情况下，为了保持向下兼容性，结构共享是关闭的，但这可能会在 v2 版本中发生变化。

要为细粒度选择器启用结构共享，你有两种选择：

#### 1. 在路由器选项中默认启用：

```tsx
const router = createRouter({
  routeTree,
  defaultStructuralSharing: true,
});
```

#### 2. 在每个 Hook 使用处单独启用：

```tsx
const result = Route.useSearch({
  select: (search) => {
    return {
      foo: search.foo,
      hello: `hello ${search.foo}`,
    };
  },
  structuralSharing: true,
});
```

> [!IMPORTANT]
> 结构共享仅适用于 JSON 兼容的数据。这意味着如果启用了结构共享，你不能使用 `select` 返回类实例（class instances）等项。

符合 TanStack Router 类型安全的设计理念，如果你尝试以下操作，TypeScript 将会报错：

```tsx
const result = Route.useSearch({
  select: (search) => {
    return {
      date: new Date(),
    };
  },
  structuralSharing: true,
});
```

如果路由器选项中默认启用了结构共享，你可以通过设置 `structuralSharing: false` 来消除此错误。
