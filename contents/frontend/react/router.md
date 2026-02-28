# Router & Query

## Router

复习 TanStack Router 时，最核心的直觉应该是：它是以“类型安全”和“异步数据流”为中心的。 它不仅仅是管理 URL，更像是一个状态管理工具。

为了帮你高效梳理，我将最常用的核心概念拆解为以下四个维度：

### 1. 路由定义 (The Route Tree)

这是所有功能的起点。你需要理解如何构建路由树以及如何处理参数。

- `createRootRoute`: 所有路由的根节点，通常包含全局布局（如 Navbar）和 `<Outlet />`。
- `createRoute`: 定义具体路径。
- 路径参数 (Path Params): 使用 `$ID` 语法，例如 `/posts/$postId`。
- 搜索参数 (Search Params): 这是 TanStack Router 的杀手锏。你需要定义一个 Zod 模式或校验函数来验证 URL 中的 Query String，确保它们也是类型安全的。

### 2. 核心组件 (Navigation & Outlet)

在页面中进行跳转和渲染时，最常用的是这三个：

- `<Link>`: 它是强类型的。当你输入 `to="/..."` 时，IDE 会自动补全路径。如果你错写了参数，TypeScript 会直接报错。
- `<Outlet>`: 占位符。告诉父路由：子路由的内容应该渲染在这里。
- `useNavigate`: 命令式导航。同样支持类型安全，确保你不会跳转到一个不存在的页面。

### 3. 数据加载与生命周期 (Data Loading)

这是 TanStack Router 区别于 React Router 的地方，它极大地优化了 Waterfall（瀑布流）问题。

- `loader`: 在路由渲染之前并行加载数据。它支持 Promise，可以配合 TanStack Query 使用。
- `beforeLoad`: 用于权限校验。如果用户没登录，可以在这里直接进行重定向（`throw redirect()`）。
- `pendingComponent`: 当 `loader` 还在加载时显示的 Loading 状态。
- `errorComponent`: 路由层级的错误捕获，不用担心一个页面崩溃导致全站白屏。

### 4. 常用 Hooks (Getting State)

开发中你经常需要从当前路由中提取信息：

| Hook              | 用途                                   |
| ----------------- | -------------------------------------- |
| `useParams()`     | 获取路径参数，如 `{ postId: '123' }`   |
| `useSearch()`     | 获取经过校验的搜索参数（Query Params） |
| `useLoaderData()` | 获取 `loader` 函数返回的数据           |
| `useRouter()`     | 获取全局路由实例，用于手动触发重载等   |

### 💡 开发中的一个小窍门

不要手写路由树。现在官方强烈推荐使用 File-based Routing (基于文件的路由)。
你只需要按照约定的命名方式（如 `posts.$postId.tsx`）放在 `routes/` 目录下，然后运行 `tsr watch`，它会自动帮你生成复杂的 `routeTree.gen.ts` 文件。

> 注意： 永远记得在根组件放一个 `<TanStackRouterDevtools />`，它能让你直观地看到当前的搜索参数和路由匹配情况，非常高效。

## Query

复习 TanStack Query (React Query) 时，最核心的直觉应该是：它是异步状态管理器。它不只是发请求，它是在帮你维护一个高性能、带缓存的“服务端数据快照”。

我们可以从“三驾马车”和“状态流转”这两个维度来拆解：

### 1. 核心“三驾马车”

这是你 90% 的业务代码里都会写的东西：

- `useQuery` (读): 用于获取数据。
- `queryKey`: 极为关键。它是缓存的唯一标识（通常是数组，如 `['posts', userId]`）。只要 Key 变了，Query 就会自动重新触发。
- `queryFn`: 返回 Promise 的函数（Fetch, Axios 等）。

- `useMutation` (写): 用于增删改。
- 它不自动执行，需要手动调用 `mutate`。
- 提供 `onSuccess` 回调，通常在这里执行“缓存失效”。

- `QueryClient` & `QueryClientProvider`:
- 连接 React 和缓存数据的桥梁，通常在 `App.tsx` 顶层配置。

### 2. 关键生命周期与配置

TanStack Query 自动帮你处理了复杂的缓存逻辑，你需要理解数据在什么时候是“新鲜”的：

- `staleTime` (新鲜时间): 默认是 0。这意味着数据一旦拿到即刻变“过期”。只要页面切回来，它就会重新静默请求。通常建议根据业务设置为 `5 * 1000` (5秒) 或更高。
- `gcTime` (垃圾回收时间): 默认 5分钟。当 Query 不再被使用时，数据在内存中保留多久。
- `enabled`: 用于依赖查询。比如只有拿到 `userId` 后，才去查“用户详情”。

### 3. 缓存同步的核心：Invalidation (失效)

当你用 `useMutation` 修改了数据，怎么让页面上的旧数据变新？

- `queryClient.invalidateQueries({ queryKey })`:
  这是最常用的操作。它会告诉 Query：这个 Key 的数据脏了，请立刻去重新获取，并让正在观察该数据的组件更新。

### 4. 开发利器与进阶常用项

- `useInfiniteQuery`: 处理无限滚动或“加载更多”的唯一选。
- `select`: 用于转换数据。比如后端返了整个 User 对象，你只需要姓名，可以用 `select: user => user.name`，这样只有姓名变了组件才会重绘。
- `placeholderData: keepPreviousData`: 在翻页加载时非常有用，能让用户在加载新一页时看到旧数据，避免页面闪烁。
- Devtools: 必装！`import { ReactQueryDevtools } from '@tanstack/react-query-devtools'`。

### 🛠️ TanStack Router + Query 的“梦幻联动”

在复习这两个库时，一定要关注它们的结合点：

1. 在 Router 的 `loader` 中预取数据：
   利用 `queryClient.ensureQueryData`。这样用户点击跳转的一瞬间，数据就开始加载了，甚至可能在组件渲染前就进缓存了。
2. 搜索参数同步：
   将 Router 的 `useSearch` 获取到的分页、筛选参数直接作为 Query 的 `queryKey`。
   > 效果：用户刷新页面、前进后退，URL 变了，Query 自动感知并拉取正确数据，完全不需要额外的 `useEffect`。
