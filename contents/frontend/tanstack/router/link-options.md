---
short_title: 链接选项
---

# 链接选项 (Link Options)

你可能经常需要复用那些打算传递给 `Link`、`redirect` 或 `Maps` 的选项。在这种情况下，直接定义一个对象字面量来表示传给 `Link` 的属性似乎是个不错的主意。

```tsx
const dashboardLinkOptions = {
  to: "/dashboard",
  search: { search: "" },
};

function DashboardComponent() {
  return <Link {...dashboardLinkOptions} />;
}
```

但这样做存在几个问题：

1. **类型推断过宽**：`dashboardLinkOptions.to` 会被推断为普通的 `string` 类型。默认情况下，当你把它传给 `Link`、`Maps` 或 `redirect` 时，它会匹配到每一个路由（虽然可以通过 `as const` 解决，但不够优雅）。
2. **校验滞后**：在将该对象展开（spread）到 `Link` 之前，你根本不知道 `dashboardLinkOptions` 是否能通过类型检查。这很容易导致你创建了错误的导航选项，却直到在 `Link` 中使用时才发现类型报错。

### 使用 `linkOptions` 函数创建可复用选项

`linkOptions` 是一个辅助函数，它会校验对象字面量的类型，并原样返回推断出的输入内容。它在配置被使用**之前**就提供了与 `Link` 完全一致的类型安全性，让代码维护和复用变得更加轻松。

使用 `linkOptions` 重写上面的例子：

```tsx
const dashboardLinkOptions = linkOptions({
  to: "/dashboard",
  search: { search: "" },
});

function DashboardComponent() {
  return <Link {...dashboardLinkOptions} />;
}
```

这种方式实现了对 `dashboardLinkOptions` 的“即时类型检查”，随后你可以在任何地方复用它：

```tsx
const dashboardLinkOptions = linkOptions({
  to: "/dashboard",
  search: { search: "" },
});

export const Route = createFileRoute("/dashboard")({
  component: DashboardComponent,
  validateSearch: (input) => ({ search: input.search }),
  beforeLoad: () => {
    // 可以在 redirect 中使用
    throw redirect(dashboardLinkOptions);
  },
});

function DashboardComponent() {
  const navigate = useNavigate();

  return (
    <div>
      {/** 可以在 navigate 中使用 */}
      <button onClick={() => navigate(dashboardLinkOptions)} />

      {/** 可以在 Link 中使用 */}
      <Link {...dashboardLinkOptions} />
    </div>
  );
}
```

---

### 使用 `linkOptions` 数组

在构建导航栏时，你可能会遍历一个数组。此时，`linkOptions` 可以用来校验一个专门为 `Link` 组件准备的对象字面量数组：

```tsx
const options = linkOptions([
  {
    to: "/dashboard",
    label: "概览",
    activeOptions: { exact: true },
  },
  {
    to: "/dashboard/invoices",
    label: "发票",
  },
  {
    to: "/dashboard/users",
    label: "用户",
  },
]);

function DashboardComponent() {
  return (
    <>
      <div className="flex items-center border-b">
        <h2 className="text-xl p-2">控制台</h2>
      </div>

      <div className="flex flex-wrap divide-x">
        {options.map((option) => {
          return (
            <Link
              {...option}
              key={option.to}
              activeProps={{ className: `font-bold` }}
              className="p-2"
            >
              {option.label}
            </Link>
          );
        })}
      </div>
      <hr />

      <Outlet />
    </>
  );
}
```

注意：`linkOptions` 的输入会被完整推断并返回。正如例子中的 `label` 属性，虽然它不是 `Link` 的标准 Prop，但类型系统依然会允许并保留它，方便你在渲染时使用。
