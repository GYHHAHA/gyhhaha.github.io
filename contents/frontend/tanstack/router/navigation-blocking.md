---
short_title: 导航拦截
---

# 导航拦截 (Navigation Blocking)

导航拦截是一种阻止导航发生的方法。这在用户尝试导航但处于以下情况时非常典型：

- 有未保存的更改
- 正在填写表单
- 正在进行支付过程中

在这些情况下，应该向用户显示提示或自定义 UI，以确认他们是否真的要离开当前页面。

- 如果用户确认，导航将正常继续。
- 如果用户取消，所有挂起的导航将被拦截。

## 导航拦截是如何工作的？

导航拦截会在整个底层 History API 上添加一层或多层“拦截器 (blockers)”。如果存在任何拦截器，导航将通过以下方式之一暂停：

- **自定义 UI**
  - 如果导航是由路由器级别控制的操作触发的，我们可以允许你执行任何任务或向用户显示任何 UI 来确认该操作。每个拦截器的 `blocker` 函数将异步且按顺序执行。如果任何拦截器函数解析 (resolve) 为或返回 `true`，则允许导航，后续拦截器将继续执行相同逻辑，直到所有拦截器都允许通过。如果任何一个拦截器解析为或返回 `false`，导航将被取消，其余的 `blocker` 函数将被忽略。
- **`onbeforeunload` 事件**
  - 对于我们无法直接控制的页面事件，我们依赖浏览器的 `onbeforeunload` 事件。如果用户尝试关闭标签页、窗口、刷新或以任何方式“卸载”页面资产，将显示浏览器通用的“确定要离开吗？”对话框。如果用户确认，所有拦截器将被跳过并卸载页面；如果用户取消，卸载操作将被取消，页面保持原样。

## 如何使用导航拦截？

有两种使用导航拦截的方式：

- 基于 Hook/逻辑的拦截
- 基于组件的拦截

## 基于 Hook/逻辑的拦截

假设我们想在表单被修改（dirty）时阻止导航。我们可以使用 `useBlocker` hook：

```tsx
import { useBlocker } from "@tanstack/react-router";

function MyComponent() {
  const [formIsDirty, setFormIsDirty] = useState(false);

  useBlocker({
    shouldBlockFn: () => {
      if (!formIsDirty) return false;

      const shouldLeave = confirm("你确定要离开吗？");
      return !shouldLeave;
    },
  });

  // ...
}
```

`shouldBlockFn` 提供了对 `current`（当前）和 `next`（下一个）位置的类型安全访问：

```tsx
import { useBlocker } from "@tanstack/react-router";

function MyComponent() {
  // 始终拦截从 /foo 到 /bar/123?hello=world 的导航
  const { proceed, reset, status } = useBlocker({
    shouldBlockFn: ({ current, next }) => {
      return (
        current.routeId === "/foo" &&
        next.fullPath === "/bar/$id" &&
        next.params.id === 123 &&
        next.search.hello === "world"
      );
    },
    withResolver: true,
  });

  // ...
}
```

请注意，即使 `shouldBlockFn` 返回 `false`，在页面刷新或关闭标签页时仍可能触发浏览器的 `beforeunload` 事件。为了控制这一点，你可以使用 `enableBeforeUnload` 选项来条件性地注册 `beforeunload` 处理程序：

```tsx
import { useBlocker } from "@tanstack/react-router";

function MyComponent() {
  const [formIsDirty, setFormIsDirty] = useState(false);

  useBlocker({
    /* ... */
    enableBeforeUnload: formIsDirty, // 或者 () => formIsDirty
  });

  // ...
}
```

你可以在 [API 参考](../api/router/useBlockerHook.md) 中找到关于 `useBlocker` hook 的更多信息。

## 基于组件的拦截

除了基于逻辑/Hook 的拦截外，你还可以使用 `Block` 组件来实现类似的结果：

```tsx
import { Block } from "@tanstack/solid-router";

function MyComponent() {
  const [formIsDirty, setFormIsDirty] = useState(false);

  return (
    <Block
      shouldBlockFn={() => {
        if (!formIsDirty) return false;

        const shouldLeave = confirm("你确定要离开吗？");
        return !shouldLeave;
      }}
      enableBeforeUnload={formIsDirty}
    />
  );

  // 或者

  return (
    <Block
      shouldBlockFn={() => formIsDirty}
      enableBeforeUnload={formIsDirty}
      withResolver
    >
      {({ status, proceed, reset }) => <>{/* ... */}</>}
    </Block>
  );
}
```

## 如何显示自定义 UI？

在大多数情况下，在 `shouldBlockFn` 函数中使用 `window.confirm` 并将 hook 设置为 `withResolver: false` 就足够了，因为它能清晰地告知用户导航已被拦截，并根据其响应解析拦截状态。

然而在某些情况下，你可能希望显示一个与应用设计集成得更好、干扰性更小的自定义 UI。

**注意：** 如果 `withResolver` 为 `true`，`shouldBlockFn` 的返回值不会直接解析拦截状态。

### 带解析器 (Resolver) 的 Hook 式自定义 UI

```tsx
import { useBlocker } from "@tanstack/react-router";

function MyComponent() {
  const [formIsDirty, setFormIsDirty] = useState(false);

  const { proceed, reset, status } = useBlocker({
    shouldBlockFn: () => formIsDirty,
    withResolver: true,
  });

  // ...

  return (
    <>
      {/* ... */}
      {status === "blocked" && (
        <div>
          <p>你确定要离开吗？</p>
          <button onClick={proceed}>是</button>
          <button onClick={reset}>否</button>
        </div>
      )}
    </>
  );
}
```

### 不带解析器的 Hook 式自定义 UI

```tsx
import { useBlocker } from "@tanstack/react-router";

function MyComponent() {
  const [formIsDirty, setFormIsDirty] = useState(false);

  useBlocker({
    shouldBlockFn: () => {
      if (!formIsDirty) {
        return false;
      }

      const shouldBlock = new Promise<boolean>((resolve) => {
        // 使用你选择的模态框管理器
        modals.open({
          title: "你确定要离开吗？",
          children: (
            <SaveBlocker
              confirm={() => {
                modals.closeAll();
                resolve(false); // 不拦截，继续导航
              }}
              reject={() => {
                modals.closeAll();
                resolve(true); // 拦截，取消导航
              }}
            />
          ),
          onClose: () => resolve(true),
        });
      });
      return shouldBlock;
    },
  });

  // ...
}
```

### 基于组件的自定义 UI

与 Hook 类似，`Block` 组件通过 render props 返回相同的状态和函数：

```tsx
import { Block } from "@tanstack/react-router";

function MyComponent() {
  const [formIsDirty, setFormIsDirty] = useState(false);

  return (
    <Block shouldBlockFn={() => formIsDirty} withResolver>
      {({ status, proceed, reset }) => (
        <>
          {/* ... */}
          {status === "blocked" && (
            <div>
              <p>你确定要离开吗？</p>
              <button onClick={proceed}>是</button>
              <button onClick={reset}>否</button>
            </div>
          )}
        </>
      )}
    </Block>
  );
}
```
