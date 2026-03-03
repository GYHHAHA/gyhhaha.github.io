---
short_title: 路由匹配
---

# 路由匹配 (Route Matching)

路由匹配遵循一致且可预测的模式。本指南将解释路由树是如何进行匹配的。

当 TanStack Router 处理您的路由树时，所有路由都会自动排序，优先匹配最具体的路由。这意味着无论您的路由树定义的顺序如何，路由始终按以下顺序排序：

- 索引路由 (Index Route)
- 静态路由 (Static Routes)（从最具体到最不具体）
- 动态路由 (Dynamic Routes)（从最长到最短）
- 通配符路由 (Splat/Wildcard Routes)

考虑以下伪路由树：

```
Root
  - blog
    - $postId
    - /
    - new
  - /
  - *
  - about
  - about/us
```

经过排序后，该路由树将变为：

```
Root
  - /
  - about/us
  - about
  - blog
    - /
    - new
    - $postId
  - *
```

这个最终顺序代表了根据“具体程度 (specificity)”进行路由匹配的先后顺序。

使用该路由树，让我们跟踪几个不同 URL 的匹配过程：

- `/blog`
  ```
  Root
    ❌ /
    ❌ about/us
    ❌ about
    ⏩ blog
      ✅ /
      - new
      - $postId
    - *
  ```
- `/blog/my-post`
  ```
  Root
    ❌ /
    ❌ about/us
    ❌ about
    ⏩ blog
      ❌ /
      ❌ new
      ✅ $postId
    - *
  ```
- `/`
  ```
  Root
    ✅ /
    - about/us
    - about
    - blog
      - /
      - new
      - $postId
    - *
  ```
- `/not-a-route`
  ```
  Root
    ❌ /
    ❌ about/us
    ❌ about
    ❌ blog
      - /
      - new
      - $postId
    ✅ *
  ```
