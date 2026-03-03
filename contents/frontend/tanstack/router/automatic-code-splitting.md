---
short_title: 自动代码分割
---

# 自动代码分割 (Automatic Code Splitting)

TanStack Router 的自动代码分割功能允许您通过懒加载路由组件及其关联数据来优化应用程序的包体积。这对于大型应用程序特别有用，因为您可以只加载当前路由所需的代码，从而最大限度地缩短初始加载时间。

要开启此功能，只需在捆绑器（Bundler）插件配置中将 `autoCodeSplitting` 选项设置为 `true` 即可。这使得路由器能够自动处理路由的代码分割，无需任何额外设置。

```ts
// vite.config.ts
import { defineConfig } from "vite";
import { tanstackRouter } from "@tanstack/router-plugin/vite";

export default defineConfig({
  plugins: [
    tanstackRouter({
      autoCodeSplitting: true, // 启用自动代码分割
    }),
  ],
});
```

但这仅仅是个开始！TanStack Router 的自动代码分割不仅易于启用，还提供了强大的自定义选项，可以根据您的具体需求和使用模式来调整路由如何拆分为代码块（Chunks）。

---

## 它是如何工作的？

TanStack Router 的自动代码分割通过在“开发阶段”和“构建阶段”对您的路由文件进行转换来实现。它会改写路由定义，为组件和加载器（Loaders）使用懒加载包装器，从而允许捆绑器将这些属性分组到独立的代码块中。

> [!TIP]
> **代码块 (Chunk)** 是包含应用程序一部分代码的文件，可以按需加载。这有助于减少应用程序的初始加载时间。

因此，当您的应用程序加载时，它不会包含每个路由的所有代码。相反，它仅包含最初需要的路由代码。随着用户在应用程序中导航，其他代码块会按需加载。

这个过程是无缝发生的，不需要您手动拆分代码或管理懒加载。TanStack Router 捆绑器插件会处理一切，确保您的路由在开箱即用时就已针对性能进行了优化。

### 转换过程

当您启用自动代码分割时，捆绑器插件会通过静态代码分析查看您的路由文件代码，并将其转换为优化后的输出。

在处理每个路由文件时，此转换过程会产生两个关键输出：

1. **引用文件 (Reference File)**：捆绑器插件获取您的原始路由文件（例如 `posts.route.tsx`），并将 `component` 或 `pendingComponent` 等属性的值修改为特殊的懒加载包装器。这些包装器指向一个捆绑器稍后会解析的“虚拟”文件。
2. **虚拟文件 (Virtual File)**：当捆绑器看到对这些虚拟文件的请求（例如 `posts.route.tsx?tsr-split=component`）时，它会拦截该请求并生成一个新的、临时的微型文件，其中**仅**包含所请求属性的代码（例如仅包含 `PostsComponent`）。

这一过程确保了您的原始代码保持简洁易读，而实际捆绑的输出则针对初始包体积进行了优化。

### 哪些内容会被分割？

决定将哪些内容拆分为独立代码块对于优化性能至关重要。TanStack Router 使用名为“**分割分组 (Split Groupings)**”的概念来确定路由的不同部分应如何捆绑在一起。

分割分组是一个属性数组，告诉 TanStack Router 如何将路由的不同部分捆绑在一起。每个分组都是一个属性名称列表，您希望将这些属性捆绑到一个懒加载的代码块中。

可用于分割的属性包括：

- `component`
- `errorComponent`
- `pendingComponent`
- `notFoundComponent`
- `loader`

默认情况下，TanStack Router 使用以下分割分组：

```sh
[
  ['component'],
  ['errorComponent'],
  ['notFoundComponent']
]
```

这意味着它会为每个路由创建三个独立的懒加载代码块：一个用于主组件，一个用于错误组件，还有一个用于“未找到”组件。

---

## 拆分规则 (Rules of Splitting)

为了使自动代码分割正常工作，必须遵循一些规则，以确保此过程能够可靠且可预测地发生。

#### 不要导出路由属性 (Do not export route properties)

诸如 `component`、`loader` 等路由属性**不应该**从路由文件中导出。导出这些属性会导致它们被捆绑到主应用程序包中，这意味着它们将无法被代码分割。

```tsx
export const Route = createRoute("/posts")({
  // ...
  notFoundComponent: PostsNotFoundComponent,
});

// ❌ 绝对不要这样做！
// 导出 notFoundComponent 将阻止它被代码分割，
// 导致它被包含在主包中。
export function PostsNotFoundComponent() {
  // ❌
  // ...
}

// ✅ 应该这样做
function PostsNotFoundComponent() {
  // ✅
  // ...
}
```

**仅此而已！** 没有其他限制。您可以像平常一样在路由文件中使用任何其他 JavaScript 或 TypeScript 特性。

---

## 粒度控制 (Granular control)

对于大多数应用程序，默认行为（`autoCodeSplitting: true`）已经足够。但是，如果您有特殊需求，TanStack Router 也提供了多种自定义选项。

### 全局分割行为 (`defaultBehavior`)

您可以通过更改插件配置中的 `defaultBehavior` 选项来定义不同属性应如何捆绑。例如，将所有 UI 相关组件捆绑到单个代码块中：

```ts title="vite.config.ts"
import { tanstackRouter } from "@tanstack/router-plugin/vite";

export default defineConfig({
  plugins: [
    tanstackRouter({
      autoCodeSplitting: true,
      codeSplittingOptions: {
        defaultBehavior: [
          [
            "component",
            "pendingComponent",
            "errorComponent",
            "notFoundComponent",
          ], // 将所有 UI 组件捆绑在一起
        ],
      },
    }),
  ],
});
```

### 高级编程控制 (`splitBehavior`)

对于复杂的规则集，您可以使用 `splitBehavior` 函数根据 `routeId` 以编程方式定义拆分逻辑。

```ts title="vite.config.ts"
import { tanstackRouter } from "@tanstack/router-plugin/vite";

export default defineConfig({
  plugins: [
    tanstackRouter({
      autoCodeSplitting: true,
      codeSplittingOptions: {
        splitBehavior: ({ routeId }) => {
          // 对于 /posts 下的所有路由，将 loader 和 component 捆绑在一起
          if (routeId.startsWith("/posts")) {
            return [["loader", "component"]];
          }
          // 所有其他路由将使用默认行为
        },
      },
    }),
  ],
});
```

### 单路由覆盖 (`codeSplitGroupings`)

为了实现终极控制，您可以通过添加 `codeSplitGroupings` 属性，直接在路由文件内部覆盖全局配置。这对于那些具有独特优化需求的路由非常有用。

```tsx title="src/routes/posts.route.tsx"
import { loadPostsData } from "./-heavy-posts-utils";

export const Route = createFileRoute("/posts")({
  // 针对这个特定路由，将 loader 和 component 捆绑在一起。
  codeSplitGroupings: [["loader", "component"]],
  loader: () => loadPostsData(),
  component: PostsComponent,
});

function PostsComponent() {
  // ...
}
```

这将为该特定路由创建一个包含 `loader` 和 `component` 的单一代码块（Chunk），并会同时覆盖默认行为以及在捆绑器配置中定义的任何编程拆分行为。

---

### 配置顺序及其重要性

到目前为止，本指南介绍了三种配置 TanStack Router 路由拆分方式的方法。

为了确保不同的配置之间不发生冲突，TanStack Router 采用了以下优先级顺序：

1. **单路由覆盖 (Per-route overrides)**：路由文件内部的 `codeSplitGroupings` 属性优先级最高。这允许您为单个路由定义特定的拆分分组。
2. **编程拆分行为 (Programmatic split behavior)**：捆绑器配置中的 `splitBehavior` 函数允许您根据 `routeId` 定义自定义的路由拆分逻辑。
3. **默认行为 (Default behavior)**：捆绑器配置中的 `defaultBehavior` 选项作为后备方案。除非被上述方式覆盖，否则此基础配置将应用于所有路由。

---

### 拆分数据加载器 (Splitting the Data Loader)

`loader` 函数负责获取路由所需的数据。默认情况下，它会被捆绑到您的“引用文件”中，并随初始包（Initial Bundle）一起加载。但是，如果您想进一步优化，也可以将 `loader` 拆分到它自己的代码块中。

> [!CAUTION]
> 将 `loader` 移至其独立的代码块中是一种**性能权衡**。这会在获取数据之前引入一次额外的服务器请求往返，可能导致初始页面加载变慢。这是因为在路由能够渲染其组件之前，**必须**先获取并执行 `loader`。
> 因此，除非您有特定的理由要拆分它，否则我们建议将 `loader` 保留在初始包中。

```ts title="vite.config.ts"
import { defineConfig } from "vite";
import { tanstackRouter } from "@tanstack/router-plugin/vite";

export default defineConfig({
  plugins: [
    tanstackRouter({
      autoCodeSplitting: true,
      codeSplittingOptions: {
        defaultBehavior: [
          ["loader"], // loader 将位于其独立的代码块中
          ["component"],
          // ... 其他组件分组
        ],
      },
    }),
  ],
});
```

除非您有必须如此的特定用例，否则我们极度不建议拆分 `loader`。在大多数情况下，不拆分 `loader` 并将其保留在主包中是性能最优的选择。
