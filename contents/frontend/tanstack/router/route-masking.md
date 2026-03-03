---
short_title: 路由掩码
---

# 路由掩码 (Route Masking)

路由掩码是一种掩盖实际路由 URL 的方式，它可以改变持久化到浏览器历史记录和地址栏中的 URL。这在你想要显示一个与实际导航目的地不同的 URL 时非常有用，并且可以在分享链接或（可选地）刷新页面时回退到显示的 URL。以下是一些示例：

- 导航到模态框路由（如 `/photo/5/modal`），但将实际 URL 掩盖为 `/photos/5`。
- 导航到评论模态框（如 `/post/5/comments`），但将实际 URL 掩盖为 `/posts/5`。
- 导航到带有查询参数 `?showLogin=true` 的路由，但掩盖 URL 使其不包含该查询参数。
- 导航到带有查询参数 `?modal=settings` 的路由，但将 URL 掩盖为 `/settings`。

这些场景都可以通过路由掩码实现，甚至可以扩展以支持更高级的模式，如 [并行路由](./parallel-routes.md)。

## 路由掩码的工作原理

> [!IMPORTANT]
> 你**不需要**为了使用路由掩码而理解它的工作原理。本节内容仅供对底层原理感兴趣的开发者阅读。如果你想直接学习如何使用，请跳至[如何使用路由掩码？](如何使用路由掩码)。

路由掩码利用 `location.state` API 将运行时所需的实际位置存储在写入 URL 的位置信息中。它将运行时位置存储在 `__tempLocation` 状态属性下：

```tsx
const location = {
  pathname: "/photos/5",
  search: "",
  hash: "",
  state: {
    key: "wesdfs",
    __tempKey: "sadfasd",
    __tempLocation: {
      pathname: "/photo/5/modal",
      search: "",
      hash: "",
      state: {},
    },
  },
};
```

当路由器从历史记录中解析出一个带有 `location.state.__tempLocation` 属性的位置时，它会优先使用该位置，而不是从 URL 解析出的位置。这允许你导航到类似 `/photos/5` 的地址，而路由器实际上导航到了 `/photo/5/modal`。当这种情况发生时，历史记录中的原始位置会被保存到 `location.maskedLocation` 属性中，以备我们需要知道**真实 URL** 是什么。一个典型的应用场景是开发者工具 (Devtools)，我们会检测路由是否被掩盖，并显示真实 URL 而不是掩盖后的 URL。

请记住，你不需要担心这些细节，底层会自动为你处理好一切！

## 如何使用路由掩码？

路由掩码的 API 非常简单，可以通过两种方式使用：

- **编程式**：通过 `<Link>` 和 `Maps()` API 中提供的 `mask` 选项。
- **声明式**：通过 Router 实例的 `routeMasks` 选项。

无论使用哪种 API，`mask` 选项接受的导航对象与 `<Link>` 和 `Maps()` 接受的完全一致。这意味着你可以使用你已经熟悉的 `to`、`replace`、`state` 和 `search` 等选项。唯一的区别是，`mask` 选项定义的参数将用于掩盖正在导航的路由 URL。

> 🧠 `mask` 选项也是**类型安全**的！这意味着如果你使用 TypeScript，当你尝试传递无效的导航对象时，会收到类型错误。太棒了！

### 编程式路由掩码

`<Link>` 和 `Maps()` 接口都接受 `mask` 选项。以下是使用 `<Link>` 组件的示例：

```tsx
<Link
  to="/photos/$photoId/modal"
  params={{ photoId: 5 }}
  mask={{
    to: "/photos/$photoId",
    params: {
      photoId: 5,
    },
  }}
>
  打开照片
</Link>
```

以下是使用 `Maps()` API 的示例：

```tsx
const navigate = useNavigate();

function onOpenPhoto() {
  navigate({
    to: "/photos/$photoId/modal",
    params: { photoId: 5 },
    mask: {
      to: "/photos/$photoId",
      params: {
        photoId: 5,
      },
    },
  });
}
```

### 声明式路由掩码

除了编程式 API，你还可以使用 Router 的 `routeMasks` 选项来声明式地掩盖路由。你无需在每个 `<Link>` 或 `Maps()` 调用中传递 `mask` 选项，而是在 Router 上创建一个路由掩码，用于掩盖符合特定模式的路由。

以下是使用 `routeMasks` 实现上述相同效果的示例：

```tsx
import { createRouteMask } from "@tanstack/react-router";

const photoModalToPhotoMask = createRouteMask({
  routeTree,
  from: "/photos/$photoId/modal",
  to: "/photos/$photoId",
  params: (prev) => ({
    photoId: prev.photoId,
  }),
});

const router = createRouter({
  routeTree,
  routeMasks: [photoModalToPhotoMask],
});
```

创建路由掩码时，你需要传递一个包含以下属性的对象：

- `routeTree`：应用此掩码的路由树。
- `from`：应用掩码的目标路由 ID。
- `...navigateOptions`：标准的 `to`、`search`、`params`、`replace` 等导航选项。

> 🧠 `createRouteMask` 也是**类型安全**的！如果你传递了无效的路由掩码，TypeScript 会及时提醒你。

## 分享 URL 时的取消掩盖 (Unmasking)

当你分享 URL 时，URL 会自动取消掩盖。因为一旦 URL 脱离了浏览器的本地历史记录栈，掩码数据就不再可用。本质上，当你从地址栏复制并粘贴 URL 时，掩码数据就丢失了——毕竟，这就是“掩盖” URL 的初衷！

## 本地取消掩盖的默认行为

**默认情况下，本地刷新页面时 URL 不会取消掩盖**。掩码数据存储在历史记录位置的 `location.state` 属性中，因此只要该历史记录位置仍存在于内存的历史栈中，掩码数据就依然有效，URL 也会保持掩盖状态。

## 页面刷新时取消掩盖

**如上所述，默认情况下页面刷新不会导致 URL 取消掩盖。**

如果你希望在本地刷新页面时取消掩盖 URL，有以下三种选择（按优先级排序）：

1. 将 Router 的默认选项 `unmaskOnReload` 设置为 `true`。
2. 在使用 `createRouteMask()` 创建路由掩码时，让掩码函数返回 `unmaskOnReload: true`。
3. 在 `<Link>` 组件或 `Maps()` API 中显式传递 `unmaskOnReload: true` 选项。
