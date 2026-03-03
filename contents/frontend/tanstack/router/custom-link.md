---
short_title: 自定义链接
---

# 自定义链接 (Custom Link)

虽然在很多情况下重复代码是可以接受的，但你可能会发现自己重复得太频繁了。有时，你希望创建具有额外行为或样式的跨切面（cross-cutting）组件，或者希望将第三方 UI 库与 TanStack Router 的类型安全特性结合使用。

## 使用 `createLink` 处理跨切面需求

`createLink` 可以创建一个与原生 `Link` 具有相同类型参数的自定义链接组件。这意味着你可以创建自己的组件，同时获得与官方 `Link` 相同的类型安全保障和 TypeScript 性能。

### 基础示例

如果你想创建一个基础的自定义链接组件，可以参考以下代码：

```tsx
import * as React from "react";
import { createLink, LinkComponent } from "@tanstack/react-router";

interface BasicLinkProps extends React.AnchorHTMLAttributes<HTMLAnchorElement> {
  // 在此处添加任何你想传递给 anchor 元素的额外 Props
}

const BasicLinkComponent = React.forwardRef<HTMLAnchorElement, BasicLinkProps>(
  (props, ref) => {
    return (
      <a ref={ref} {...props} className={"block px-3 py-2 text-blue-700"} />
    );
  },
);

// 使用 createLink 包装你的基础组件
const CreatedLinkComponent = createLink(BasicLinkComponent);

// 导出具备类型安全的自定义 Link
export const CustomLink: LinkComponent<typeof BasicLinkComponent> = (props) => {
  return <CreatedLinkComponent preload={"intent"} {...props} />;
};
```

现在你可以像使用普通 `Link` 一样使用你的 `CustomLink`，它会自动提示路径和参数：

```tsx
<CustomLink to={"/dashboard/invoices/$invoiceId"} params={{ invoiceId: 0 }} />
```

---

## `createLink` 与第三方库集成

以下是将 `createLink` 与流行第三方 UI 库结合使用的示例。

### React Aria Components 示例

React Aria Components (v1.11.0+) 支持 TanStack Router 的 `preload (intent)` 属性。你可以使用 `createLink` 包装任何作为链接使用的 RAC 组件。

```tsx title="RACLink.tsx"
import { createLink } from "@tanstack/react-router";
import { Link as RACLink, MenuItem } from "react-aria-components";

export const Link = createLink(RACLink);
export const MenuItemLink = createLink(MenuItem);
```

```tsx title="CustomRACLink.tsx"
import { createLink } from "@tanstack/react-router";
import { Link as RACLink, type LinkProps } from "react-aria-components";

interface MyLinkProps extends LinkProps {
  // 你的自定义属性
}

function MyLink(props: MyLinkProps) {
  return (
    <RACLink
      {...props}
      style={({ isHovered }) => ({
        color: isHovered ? "red" : "blue",
      })}
    />
  );
}

export const Link = createLink(MyLink);
```

### Chakra UI 示例

```tsx title="ChakraLinkComponent.tsx"
import * as React from "react";
import { createLink, LinkComponent } from "@tanstack/react-router";
import { Link } from "@chakra-ui/react";

interface ChakraLinkProps extends Omit<
  React.ComponentPropsWithoutRef<typeof Link>,
  "href"
> {}

const ChakraLinkComponent = React.forwardRef<
  HTMLAnchorElement,
  ChakraLinkProps
>((props, ref) => <Link ref={ref} {...props} />);

const CreatedLinkComponent = createLink(ChakraLinkComponent);

export const CustomLink: LinkComponent<typeof ChakraLinkComponent> = (
  props,
) => {
  return (
    <CreatedLinkComponent
      textDecoration={"underline"}
      _hover={{ textDecoration: "none" }}
      _focus={{ textDecoration: "none" }}
      preload={"intent"}
      {...props}
    />
  );
};
```

### MUI (Material UI) 示例

MUI 的集成非常灵活。如果只是简单的包装：

```tsx title="CustomLink.tsx"
import { createLink } from "@tanstack/react-router";
import { Link } from "@mui/material";

export const CustomLink = createLink(Link);
```

如果需要使用 MUI 的 **`Button`** 作为链接（注意设置 `component="a"`）：

```tsx title="CustomButtonLink.tsx"
import React from "react";
import { createLink, LinkComponent } from "@tanstack/react-router";
import { Button, type ButtonProps } from "@mui/material";

interface MUIButtonLinkProps extends ButtonProps<"a"> {}

const MUIButtonLinkComponent = React.forwardRef<
  HTMLAnchorElement,
  MUIButtonLinkProps
>((props, ref) => <Button ref={ref} component="a" {...props} />);

const CreatedButtonLinkComponent = createLink(MUIButtonLinkComponent);

export const CustomButtonLink: LinkComponent<typeof MUIButtonLinkComponent> = (
  props,
) => {
  return <CreatedButtonLinkComponent preload={"intent"} {...props} />;
};
```

### Mantine 示例

```tsx title="CustomLink.tsx"
import * as React from "react";
import { createLink, LinkComponent } from "@tanstack/react-router";
import { Anchor, AnchorProps } from "@mantine/core";

interface MantineAnchorProps extends Omit<AnchorProps, "href"> {}

const MantineLinkComponent = React.forwardRef<
  HTMLAnchorElement,
  MantineAnchorProps
>((props, ref) => <Anchor ref={ref} {...props} />);

const CreatedLinkComponent = createLink(MantineLinkComponent);

export const CustomLink: LinkComponent<typeof MantineLinkComponent> = (
  props,
) => {
  return <CreatedLinkComponent preload="intent" {...props} />;
};
```
