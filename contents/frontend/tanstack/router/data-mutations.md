---
short_title: 数据变更
---

# 数据变更 (Data Mutations)

由于 TanStack Router 本身不存储或缓存数据，因此除了对外部变更事件引起的 URL 侧边效应做出反应外，它在数据变更中的角色几乎微乎其微。尽管如此，我们还是整理了一系列您可能会觉得有用的变更相关特性，以及实现这些特性的库。

在选择变更工具库时，建议寻找并使用支持以下功能的工具：

- 处理并缓存**提交状态（Submission state）**。
- 提供局部和全局的**乐观 UI（Optimistic UI）**支持。
- 内置用于触发失效（Invalidation）的 Hook。
- 支持同时处理多个正在进行的变更。
- 将变更状态组织为全局可访问的资源。
- 提交状态的历史记录及垃圾回收。

### 推荐的库：

- **[TanStack Query](https://tanstack.com/query/latest/docs/react/guides/mutations)** (强烈推荐)
- **SWR** / **RTK Query** / **urql** / **Relay** / **Apollo**

或者状态管理库：

- **Zustand** / **Jotai** / **Recoil** / **Redux**

与数据获取一样，变更状态也没有万能的方案。我们建议您多尝试几种，看看哪种最契合您的团队需求。

> ⚠️ **还没走？** 提交状态的持久化是一个很有趣的话题。你是打算永久保留每一次变更吗？你如何知道何时该清理它？如果用户离开屏幕后再回来，状态应该还在吗？让我们深入探讨！

---

## 变更后使 TanStack Router 失效

虽然 TanStack Router 不直接存储数据，但它拥有内置的短期缓存。当您进行了与路由数据相关的变更时，当前路由所展示的数据很可能已经过时了。

当发生与加载器（Loader）数据相关的变更时，我们可以使用 `router.invalidate()` 强制路由器重新加载所有当前匹配的路由：

```tsx
const router = useRouter();

const addTodo = async (todo: Todo) => {
  try {
    await api.addTodo(todo);
    // 变更成功后，让路由数据失效
    router.invalidate();
  } catch {
    // 处理错误
  }
};
```

使当前路由失效的操作是在后台进行的。这意味着在获取到新数据之前，旧数据会继续显示，就像您导航到一个新路由时的行为一样。

如果您想**等待**所有加载器完成重新加载后再继续执行后续逻辑，请向 `router.invalidate` 传入 `{ sync: true }`：

```tsx
await router.invalidate({ sync: true });
```

---

## 长期变更状态 (Long-term mutation State)

无论使用哪个变更库，变更操作通常会产生与其提交相关的状态。虽然大多数变更是“触发后即忘”，但有些状态需要长期存在，以支持乐观 UI 或向用户提供反馈（如“更新成功”）。

考虑以下交互场景：

1. 用户进入 `/posts/123/edit` 编辑帖子。
2. 用户点击保存，成功后看到一条 **“帖子更新成功”** 的消息。
3. 用户导航到 `/posts` 列表页。
4. 用户再次导航回 `/posts/123/edit`。

如果不处理路由变化，当用户回到编辑页时，可能依然会看到那条 **“帖子更新成功”** 的旧消息。这显然不符合预期——我们的本意可不是想让这条消息永久挂在那里，对吧？！

---

## 使用变更键 (Mutation Keys)

最优雅的方法是利用变更库支持的**键控机制（Keying mechanism）**。这样当路由参数（如 `roomId`）变化时，变更状态会自动重置：

```tsx
const routeApi = getRouteApi("/room/$roomId/chat");

function ChatRoom() {
  const { roomId } = routeApi.useParams();

  const sendMessageMutation = useCoolMutation({
    fn: sendMessage,
    // 当 roomId 改变时，清除该变更的状态（包括提交状态和消息提示）
    key: ["sendMessage", roomId],
  });

  return (
    <>
      {sendMessageMutation.submissions.map((submission) => (
        <div key={submission.id}>
          <div>状态: {submission.status}</div>
          <div>消息: {submission.message}</div>
        </div>
      ))}
    </>
  );
}
```

---

## 使用 `router.subscribe` 方法

对于不支持键控机制的库，我们可能需要在用户导航离开时手动重置变更状态。为此，我们可以利用 TanStack Router 的 `subscribe` 方法。

我们要监听的是 `onResolved` 事件。这个事件会在 **路径改变（而不只是重载）并最终解析完成** 时触发。这是清理旧变更状态的绝佳时机：

```tsx
const router = createRouter({ ... })
const coolMutationCache = createCoolMutationCache()

// 订阅路由解析完成事件
const unsubscribeFn = router.subscribe('onResolved', () => {
  // 当路由确实发生变化时，清理变更缓存
  coolMutationCache.clear()
})
```
