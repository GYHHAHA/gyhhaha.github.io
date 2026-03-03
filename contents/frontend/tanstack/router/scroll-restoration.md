---
short_title: 滚动恢复
---

# 滚动恢复 (Scroll Restoration)

## 哈希/页面顶部滚动 (Hash/Top-of-Page Scrolling)

TanStack Router 开箱即支持 **哈希滚动** 和 **页面顶部滚动**，无需任何额外配置。

## 滚动到顶部与嵌套可滚动区域

默认情况下，“滚动到顶部”的行为模拟了浏览器，这意味着在成功导航后，只有 `window` 自身会滚动到顶部。然而，对于许多应用来说，由于复杂的布局设计，主滚动区域通常是一个嵌套的 `div` 或类似的元素。如果你希望 TanStack Router 也为你滚动这些主滚动区域，可以使用 `routerOptions.scrollToTopSelectors` 添加选择器来定位它们：

```tsx
const router = createRouter({
  scrollToTopSelectors: ["#main-scrollable-area"],
});
```

对于无法简单使用 `document.querySelector(selector)` 解析的复杂选择器，你可以向 `routerOptions.scrollToTopSelectors` 传递返回 HTML 元素的函数：

```tsx
const selector = () =>
  document
    .querySelector("#shadowRootParent")
    ?.shadowRoot?.querySelector("#main-scrollable-area");

const router = createRouter({
  scrollToTopSelectors: [selector],
});
```

这些选择器是**在 `window` 之外额外处理的**，目前无法禁用对 `window` 的滚动处理。

## 滚动恢复 (Scroll Restoration)

滚动恢复是指当用户返回（Back）到之前访问过的页面时，恢复该页面滚动位置的过程。对于标准的基于 HTML 的网站，这通常是内置功能，但在 SPA（单页应用）中复制这一行为可能很困难，因为：

- SPA 通常使用 `history.pushState` API 进行导航，因此浏览器无法原生识别并恢复滚动位置。
- SPA 有时会异步渲染内容，因此浏览器在渲染完成前不知道页面的实际高度。
- SPA 有时会使用嵌套的滚动容器来强制执行特定的布局和功能。

不仅如此，应用中通常会有多个滚动区域（例如：聊天应用可能有一个可滚动的侧边栏和一个可滚动的聊天区域），而不只是 `body`。在这种情况下，你会希望独立恢复这两个区域的滚动位置。

为了解决这个问题，TanStack Router 提供了滚动恢复组件和 Hook，用于自动监控、缓存并为你恢复滚动位置。

它的工作原理包括：

- 监控 DOM 的滚动事件。
- 将可滚动区域注册到滚动恢复缓存中。
- 监听适当的路由器事件，以确定何时缓存和恢复滚动位置。
- 在缓存中存储每个滚动区域的滚动位置（包括 `window` 和 `body`）。
- 在 DOM 绘制之前的成功导航后恢复滚动位置。

听起来很复杂，但对你来说，只需简单的配置即可：

```tsx
const router = createRouter({
  scrollRestoration: true,
});
```

> [!NOTE]
> `<ScrollRestoration />` 组件仍然有效，但已被弃用。

## 自定义缓存键 (Custom Cache Keys)

借鉴了 Remix 的滚动恢复 API，你还可以使用 `getKey` 选项来自定义用于缓存特定滚动区域滚动位置的键。例如，这可以用于强制使用相同的滚动位置，而不管用户的浏览器历史记录如何。

`getKey` 选项接收来自 TanStack Router 的相关 `Location` 状态，并期望你返回一个字符串来唯一标识该状态的滚动测量值。

默认的 `getKey` 是 `(location) => location.state.__TSR_key!`，其中 `__TSR_key` 是为历史记录中的每个条目生成的唯一键。

> 在 `v1.121.34` 之前的旧版本中，默认使用 `state.key` 作为键，但现在已弃用，推荐使用 `state.__TSR_key`。目前 `location.state.key` 仍可用于兼容，但将在下一个主要版本中删除。

## 示例

你可以将滚动同步到路径名（pathname）：

```tsx
const router = createRouter({
  getScrollRestorationKey: (location) => location.pathname,
});
```

你可以有条件地仅同步某些路径，其余路径使用默认键：

```tsx
const router = createRouter({
  getScrollRestorationKey: (location) => {
    const paths = ["/", "/chat"];
    return paths.includes(location.pathname)
      ? location.pathname
      : location.state.__TSR_key!;
  },
});
```

## 阻止滚动恢复

有时你可能希望阻止滚动恢复的发生。为此，你可以利用以下 API 中提供的 `resetScroll` 选项：

- `<Link resetScroll={false}>`
- `Maps({ resetScroll: false })`
- `redirect({ resetScroll: false })`

当 `resetScroll` 设置为 `false` 时，下次导航的滚动位置将既不会被恢复（如果是导航到栈中已有的历史事件），也不会重置到顶部（如果是栈中的新历史事件）。

## 手动滚动恢复

大多数情况下，你不需要做任何特殊操作就能让滚动恢复工作。但在某些情况下，你可能需要手动控制滚动恢复。最常见的例子是 **虚拟化列表 (Virtualized Lists)**。

要在整个浏览器窗口内手动控制虚拟化列表的滚动恢复：

```tsx
function Component() {
  const scrollEntry = useElementScrollRestoration({
    getElement: () => window,
  })

  // 让我们使用 TanStack Virtual 来虚拟化一些内容！
  const virtualizer = useWindowVirtualizer({
    count: 10000,
    estimateSize: () => 100,
    // 我们将来自滚动恢复条目的 scrollY 传递给虚拟化器作为初始偏移量
    initialOffset: scrollEntry?.scrollY,
  })

  return (
    <div>
      {virtualizer.getVirtualItems().map(item => (
        // ...渲染逻辑
      ))}
    </div>
  )
}
```

要手动控制特定元素的滚动恢复，可以使用 `useElementScrollRestoration` Hook 和 `data-scroll-restoration-id` DOM 属性：

```tsx
function Component() {
  // 我们需要一个唯一的 ID 来对特定元素进行手动滚动恢复
  // 该 ID 在应用中对于此元素应尽可能唯一
  const scrollRestorationId = "myVirtualizedContent";

  // 我们使用该 ID 获取该元素的滚动条目
  const scrollEntry = useElementScrollRestoration({
    id: scrollRestorationId,
  });

  // 使用 TanStack Virtual 虚拟化内容
  const virtualizerParentRef = React.useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: 10000,
    getScrollElement: () => virtualizerParentRef.current,
    estimateSize: () => 100,
    // 将 scrollY 传递给虚拟化器作为初始偏移量
    initialOffset: scrollEntry?.scrollY,
  });

  return (
    <div
      ref={virtualizerParentRef}
      // 我们将滚动恢复 ID 作为自定义属性传递给元素
      // 滚动恢复观察器会识别该属性
      data-scroll-restoration-id={scrollRestorationId}
      className="flex-1 border rounded-lg overflow-auto relative"
    >
      {/* ...渲染逻辑 */}
    </div>
  );
}
```

## 滚动行为 (Scroll Behavior)

要控制在页面间导航时的滚动行为，可以使用 `scrollRestorationBehavior` 选项。这允许你使页面间的过渡变得即时，而不是平滑滚动。滚动恢复行为的全局配置拥有与浏览器支持的选项相同的选项，即 `smooth`、`instant` 和 `auto`（详见 [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollIntoView#behavior)）。

```tsx
const router = createRouter({
  scrollRestorationBehavior: "instant",
});
```
