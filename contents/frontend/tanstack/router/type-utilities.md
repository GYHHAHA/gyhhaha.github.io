---
short_title: 类型工具
---

# 类型工具 (Type Utilities)

TanStack Router 暴露的大多数类型都是内部使用的，不仅容易发生破坏性变更（Breaking changes），而且往往难以直接上手。因此，我们提供了一个专门面向外部使用的类型工具子集。这些工具在类型层面上复现了 TanStack Router 运行时的类型安全体验，并为您提供了灵活的校验切入点。

## 使用 `ValidateLinkOptions` 校验链接选项

`ValidateLinkOptions` 可以在推断位置校验对象字面量，确保它们符合 `Link` 的选项要求。例如，你可能有一个通用的 `HeadingLink` 组件，它接收一个 `title` 属性和一组 `linkOptions`，旨在让该组件可以复用于任何导航场景。

```tsx
export interface HeaderLinkProps<
  TRouter extends RegisteredRouter = RegisteredRouter,
  TOptions = unknown,
> {
  title: string;
  linkOptions: ValidateLinkOptions<TRouter, TOptions>;
}

export function HeadingLink<TRouter extends RegisteredRouter, TOptions>(
  props: HeaderLinkProps<TRouter, TOptions>,
): React.ReactNode;
export function HeadingLink(props: HeaderLinkProps): React.ReactNode {
  return (
    <>
      <h1>{props.title}</h1>
      <Link {...props.linkOptions} />
    </>
  );
}
```

这里我们使用了更宽松的 `HeadingLink` 重载，以避免在泛型签名中进行繁琐的类型断言。使用不带类型参数的宽松签名是简化 `HeadingLink` 内部实现的一个小技巧。

为了获得最佳的 TypeScript 性能，**面向外部的签名应始终指定 `TRouter`**。而在推断位置（如 `HeadingLink` 调用处），应始终使用 `TOptions` 来推断 `linkOptions`，从而准确地窄化 `params` 和 `search` 的类型。

结果就是，下方的 `linkOptions` 现在是完全类型安全的了：

```tsx
// 自动校验路径是否存在
<HeadingLink title="Posts" linkOptions={{ to: '/posts' }} />

// 自动要求传入必填的 params.postId
<HeadingLink
  title="Post"
  linkOptions={{
    to: '/posts/$postId',
    params: { postId: 'postId' }
  }}
/>
```

---

## 使用 `ValidateLinkOptionsArray` 校验链接选项数组

所有的导航类型工具都有对应的数组变体。`ValidateLinkOptionsArray` 允许对 `Link` 选项数组进行类型检查。例如，一个每个条目都是链接的通用 `Menu` 组件：

```tsx
export interface MenuProps<
  TRouter extends RegisteredRouter = RegisteredRouter,
  TItems extends ReadonlyArray<unknown> = ReadonlyArray<unknown>,
> {
  items: ValidateLinkOptionsArray<TRouter, TItems>;
}

export function Menu<
  TRouter extends RegisteredRouter = RegisteredRouter,
  TItems extends ReadonlyArray<unknown>,
>(props: MenuProps<TRouter, TItems>): React.ReactNode;
export function Menu(props: MenuProps): React.ReactNode {
  return (
    <ul>
      {props.items.map((item) => (
        <li key={item.to}>
          <Link {...item} />
        </li>
      ))}
    </ul>
  );
}
```

这样，传入的 `items` 属性就具备了完整的类型安全：

```tsx
<Menu
  items={[
    { to: "/posts" },
    { to: "/posts/$postId", params: { postId: "postId" } },
  ]}
/>
```

你甚至可以为数组中的每个链接固定一个 `from` 路径。这允许所有菜单项都相对于 `from` 进行导航。我们可以配合 `ValidateFromPath` 工具来进一步加强对 `from` 的类型检查：

```tsx
export interface MenuProps<
  TRouter extends RegisteredRouter = RegisteredRouter,
  TItems extends ReadonlyArray<unknown> = ReadonlyArray<unknown>,
  TFrom extends string = string,
> {
  from: ValidateFromPath<TRouter, TFrom>;
  items: ValidateLinkOptionsArray<TRouter, TItems, TFrom>;
}

export function Menu<
  TRouter extends RegisteredRouter = RegisteredRouter,
  TItems extends ReadonlyArray<unknown>,
  TFrom extends string = string,
>(props: MenuProps<TRouter, TItems, TFrom>): React.ReactNode;
export function Menu(props: MenuProps): React.ReactNode {
  return (
    <ul>
      {props.items.map((item) => (
        <li key={item.to}>
          {/* 将 props.from 传递给每个 Link */}
          <Link {...item} from={props.from} />
        </li>
      ))}
    </ul>
  );
}
```

通过提供额外的类型参数，`items` 数组中的链接现在支持相对于 `/posts` 的相对路径导航（如 `.` 或 `./$postId`）：

```tsx
<Menu
  from="/posts"
  items={[{ to: "." }, { to: "./$postId", params: { postId: "postId" } }]}
/>
```

---

## 使用 `ValidateRedirectOptions` 校验重定向选项

`ValidateRedirectOptions` 用于校验重定向相关的对象字面量。例如，一个通用的 `fetchOrRedirect` 函数：如果请求失败（如 401 未授权），则自动重定向到登录页。

```tsx
export async function fetchOrRedirect<
  TRouter extends RegisteredRouter = RegisteredRouter,
  TOptions,
>(
  url: string,
  redirectOptions: ValidateRedirectOptions<TRouter, TOptions>,
): Promise<unknown>;
export async function fetchOrRedirect(
  url: string,
  redirectOptions: ValidateRedirectOptions,
): Promise<unknown> {
  const response = await fetch(url);

  if (!response.ok && response.status === 401) {
    // 抛出类型安全的重定向
    throw redirect(redirectOptions);
  }

  return await response.json();
}

// 使用处：redirectOptions 会被严格校验
fetchOrRedirect("[http://example.com/](http://example.com/)", { to: "/login" });
```

---

## 使用 `ValidateNavigateOptions` 校验导航选项

`ValidateNavigateOptions` 用于校验命令式导航（Navigate）的选项。这在你编写自定义导航 Hook 时非常管用，比如创建一个支持启用/禁用状态的受控导航。

```tsx
export function useConditionalNavigate<
  TRouter extends RegisteredRouter = RegisteredRouter,
  TOptions,
>(
  navigateOptions: ValidateNavigateOptions<TRouter, TOptions>,
): UseConditionalNavigateResult;
export function useConditionalNavigate(
  navigateOptions: ValidateNavigateOptions,
): UseConditionalNavigateResult {
  const [enabled, setEnabled] = useState(false);
  const navigate = useNavigate();

  return {
    enable: () => setEnabled(true),
    disable: () => setEnabled(false),
    navigate: () => {
      if (enabled) {
        navigate(navigateOptions);
      }
    },
  };
}

// 使用处：navigateOptions 具备完整的路径补全和参数校验
const { enable, disable, navigate } = useConditionalNavigate({
  to: "/posts/$postId",
  params: { postId: "postId" },
});
```
