---
short_title: 路由树
---

# 路由树 (Route Trees)

TanStack Router 使用嵌套的路由树将 URL 与要渲染的正确定组件树进行匹配。

为了构建路由树，TanStack Router 支持：

- [基于文件的路由 (File-Based Routing)](./file-based-routing.md)
- [基于代码的路由 (Code-Based Routing)](./code-based-routing.md)

这两种方法都支持完全相同的核心特性和功能，但**基于文件的路由可以用更少的代码实现相同甚至更好的结果**。因此，**基于文件的路由是配置 TanStack Router 的首选且推荐的方式**。大部分文档都是从基于文件路由的角度编写的。

## 路由树概念

嵌套路由是一个强大的概念，它允许您使用 URL 来渲染嵌套的组件树。例如，给定 URL 为 `/blog/posts/123`，您可以创建一个如下所示的路由层级结构：

```tsx
├── blog
│   ├── posts
│   │   ├── $postId
```

并渲染一个如下所示的组件树：

```tsx
<Blog>
  <Posts>
    <Post postId="123" />
  </Posts>
</Blog>
```

让我们把这个概念扩展到一个更大的网站结构中，但现在使用文件名来表示：

```
/routes
├── __root.tsx
├── index.tsx
├── about.tsx
├── posts/
│   ├── index.tsx
│   ├── $postId.tsx
├── posts.$postId.edit.tsx
├── settings/
│   ├── profile.tsx
│   ├── notifications.tsx
├── _pathlessLayout/
│   ├── route-a.tsx
├── ├── route-b.tsx
├── files/
│   ├── $.tsx
```

以上是一个可以用于 TanStack Router 的有效路由树配置！基于文件的路由包含许多强大的功能和约定，让我们来详细拆解一下。

## 路由树配置 (Route Tree Configuration)

路由树可以通过以下几种不同的方式进行配置：

- 扁平路由 (Flat Routes)
- 目录路由 (Directories)
- 混合扁平与目录路由 (Mixed Flat Routes and Directories)
- 虚拟文件路由 (Virtual File Routes)
- 基于代码的路由 (Code-Based Routes)

请务必查看上方针对每种路由树类型的完整文档链接，或者直接进入下一章节开始学习基于文件的路由。
