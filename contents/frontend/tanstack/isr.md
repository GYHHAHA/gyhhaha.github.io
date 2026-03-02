---
short_title: 增量静态再生
---

# 增量静态再生 (ISR)

增量静态再生（Incremental Static Regeneration，简称 ISR）允许你从 CDN 提供静态生成的内内容，同时在后台定期对其进行再生。这让你既能享受静态站点的性能优势，又能保持动态内容的新鲜度。

## ISR 在 TanStack Start 中的工作原理

TanStack Start 的 ISR 方法非常灵活，它利用了适用于任何 CDN 的标准 HTTP 缓存响应头。与某些框架特定的 ISR 实现不同，这种方法让你能够在页面和数据层级上完全控制缓存行为。

核心概念非常简单：

1. **静态预渲染 (Static Prerendering)**：在构建时生成页面。
2. **CDN 缓存**：通过缓存响应头控制 CDN 缓存 HTML 的时长。
3. **重新验证 (Revalidation)**：缓存过期后，下一个请求将触发再生。
4. **过时而重新验证 (Stale-While-Revalidate)**：在后台获取新鲜数据时，先向用户提供过时的旧内容。

---

## 缓存响应头策略 (Cache Header Strategies)

### 基于时间的重新验证 (Time-Based Revalidation)

最常见的 ISR 模式是使用 `Cache-Control` 响应头及其 `max-age` 和 `s-maxage` 指令：

```tsx
// vite.config.ts
import { tanstackStart } from "@tanstack/react-start/plugin/vite";
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [
    tanstackStart({
      prerender: {
        // 配置需要预渲染的路由
        routes: ["/blog", "/blog/posts/*"],
        crawlLinks: true,
      },
    }),
  ],
});
```

```tsx
// routes/blog/posts/$postId.tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/blog/posts/$postId")({
  loader: async ({ params }) => {
    const post = await fetchPost(params.postId);
    return { post };
  },
  headers: () => ({
    // 在 CDN 缓存 1 小时，允许提供最长 1 天的过时内容
    "Cache-Control":
      "public, max-age=3600, s-maxage=3600, stale-while-revalidate=86400",
  }),
});

export default function BlogPost() {
  const { post } = Route.useLoaderData();
  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.content}</div>
    </article>
  );
}
```

### 理解 Cache-Control 指令

- **`public`**：响应可以被任何缓存（CDN、浏览器等）存储。
- **`max-age=3600`**：内容在 3600 秒（1 小时）内是新鲜的。
- **`s-maxage=3600`**：覆盖共享缓存（CDN）的 `max-age`。
- **`stale-while-revalidate=86400`**：在后台进行重新验证时，允许提供长达 24 小时的过时内容。
- **`immutable`**：内容永远不会改变（通常用于带哈希的资源文件）。

---

## 在服务器函数中使用 ISR

服务器函数（Server functions）同样可以为动态数据端点设置缓存响应头：

```tsx
// routes/api/products/$productId.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/api/products/$productId")({
  server: {
    handlers: {
      GET: async ({ params }) => {
        const product = await db.products.findById(params.productId);

        return Response.json(
          { product },
          {
            headers: {
              "Cache-Control":
                "public, max-age=300, stale-while-revalidate=600",
              // Cloudflare 特定的缓存控制
              "CDN-Cache-Control": "max-age=3600",
            },
          },
        );
      },
    },
  },
});
```

### 使用中间件处理缓存响应头

对于 API 路由，你可以使用中间件来统一设置缓存：

```tsx
// src/middleware/cacheMiddleware.ts
import { createMiddleware } from "@tanstack/react-start";

export const cacheMiddleware = createMiddleware().server(async ({ next }) => {
  const result = await next();

  // 向响应中添加缓存头
  result.response.headers.set(
    "Cache-Control",
    "public, max-age=3600, stale-while-revalidate=86400",
  );

  return result;
});
```

对于页面路由，直接使用 `headers` 属性会更简单直观。

---

## 按需重新验证 (On-Demand Revalidation)

虽然基于时间的重新验证能满足大多数场景，但有时你需要在内容更新时立即清除特定页面的缓存：

```tsx
// routes/api/revalidate.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/api/revalidate")({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const { path, secret } = await request.json();

        // 验证密钥
        if (secret !== process.env.REVALIDATE_SECRET) {
          return Response.json({ error: "无效令牌" }, { status: 401 });
        }

        // 通过你的 CDN API 触发缓存清除（以 Cloudflare 为例）
        await fetch(
          `https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/purge_cache`,
          {
            method: "POST",
            headers: {
              Authorization: `Bearer ${CF_API_TOKEN}`,
              "Content-Type": "application/json",
            },
            body: JSON.stringify({
              files: [`https://yoursite.com${path}`],
            }),
          },
        );

        return Response.json({ revalidated: true });
      },
    },
  },
});
```

## CDN 特定配置 (CDN-Specific Configuration)

### Cloudflare Workers

Cloudflare 遵循标准的 `Cache-Control` 响应头，并提供了额外的控制字段：

```tsx
export const Route = createFileRoute("/products/$id")({
  headers: () => ({
    "Cache-Control": "public, max-age=3600",
    // Cloudflare 特有的响应头，用于实现更精细的控制
    "CDN-Cache-Control": "max-age=7200",
  }),
});
```

### Netlify

Netlify 使用 `Cache-Control` 响应头，同时也支持通过 `_headers` 文件进行配置：

```plaintext
# public/_headers
/blog/*
  Cache-Control: public, max-age=3600, stale-while-revalidate=86400

/api/*
  Cache-Control: public, max-age=300
```

### Vercel

部署到 Vercel 时，请使用其边缘网络（Edge Network）缓存响应头：

```tsx
export const Route = createFileRoute("/posts/$id")({
  headers: () => ({
    "Cache-Control": "public, s-maxage=3600, stale-while-revalidate=86400",
  }),
});
```

---

## 将 ISR 与客户端缓存结合

TanStack Router 内置的缓存控制可以与 CDN 缓存协同工作，构建强大的多级缓存体系：

```tsx
export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ params }) => {
    return fetchPost(params.postId);
  },
  // CDN 缓存（通过 headers 配置）
  headers: () => ({
    "Cache-Control": "public, max-age=3600, stale-while-revalidate=86400",
  }),
  // 客户端缓存（通过 TanStack Router 配置）
  staleTime: 60_000, // 客户端认为数据在 60 秒内是新鲜的
  gcTime: 5 * 60_000, // 在内存中保留 5 分钟
});
```

这种配置创建了一个**多级缓存策略**：

1. **CDN 边缘层**：1 小时缓存，允许 24 小时内的过时重新验证。
2. **客户端层**：60 秒新鲜期，5 分钟内存驻留。

---

## 常见的 ISR 模式 (Common ISR Patterns)

### 博客文章

```tsx
export const Route = createFileRoute("/blog/$slug")({
  loader: async ({ params }) => fetchPost(params.slug),
  headers: () => ({
    // 缓存 1 小时，允许提供最长 7 天的过时内容
    "Cache-Control": "public, max-age=3600, stale-while-revalidate=604800",
  }),
  staleTime: 5 * 60_000, // 客户端 5 分钟新鲜期
});
```

### 电商产品页

```tsx
export const Route = createFileRoute("/products/$id")({
  loader: async ({ params }) => fetchProduct(params.id),
  headers: () => ({
    // 由于库存变动频繁，缓存时间较短
    "Cache-Control": "public, max-age=300, stale-while-revalidate=3600",
  }),
  staleTime: 30_000, // 客户端 30 秒新鲜期
});
```

### 营销落地页

```tsx
export const Route = createFileRoute("/landing/$campaign")({
  loader: async ({ params }) => fetchCampaign(params.campaign),
  headers: () => ({
    // 内容稳定，设置长效缓存
    "Cache-Control": "public, max-age=86400, stale-while-revalidate=604800",
  }),
  staleTime: 60 * 60_000, // 客户端 1 小时新鲜期
});
```

### 用户特定页面

```tsx
export const Route = createFileRoute("/dashboard")({
  loader: async () => fetchUserData(),
  headers: () => ({
    // 私有缓存，禁止 CDN 缓存
    "Cache-Control": "private, max-age=60",
  }),
  staleTime: 30_000,
});
```

## 最佳实践 (Best Practices)

### 1. 从保守配置开始 (Start Conservative)

先从较短的缓存时间开始，随着对内容更新模式的了解，再逐步增加缓存时长：

```tsx
// 初始阶段建议：
'Cache-Control': 'public, max-age=300, stale-while-revalidate=600'

// 稳定后调整为：
'Cache-Control': 'public, max-age=3600, stale-while-revalidate=86400'
```

### 2. 使用 ETag 进行验证

ETag 能帮助 CDN 更高效地重新验证内容是否发生了变化：

```tsx
import { createMiddleware } from "@tanstack/react-start";
import crypto from "crypto";

const etagMiddleware = createMiddleware().server(async ({ next }) => {
  const result = await next();

  // 根据响应内容生成 ETag
  const etag = crypto
    .createHash("md5")
    .update(JSON.stringify(result.data))
    .digest("hex");

  result.response.headers.set("ETag", `"${etag}"`);

  return result;
});
```

### 3. 根据查询参数区分缓存 (Vary Cache)

当内容因查询参数而异时，确保在缓存键中包含它们：

```tsx
export const Route = createFileRoute("/search")({
  headers: () => ({
    "Cache-Control": "public, max-age=300",
    // 告知缓存服务器根据 Accept 族头部区分响应
    Vary: "Accept, Accept-Encoding",
  }),
});
```

### 4. 监控缓存命中率 (Monitor Cache Hit Rates)

通过跟踪 CDN 性能来优化缓存时间：

```tsx
const cacheMonitoringMiddleware = createMiddleware().server(
  async ({ next }) => {
    const result = await next();

    // 记录缓存状态（从 CDN 响应头中获取，以 Cloudflare 为例）
    console.log("缓存状态:", result.response.headers.get("cf-cache-status"));

    return result;
  },
);
```

### 5. 结合静态预渲染

在构建时进行预渲染以实现瞬间的首屏加载，然后利用 ISR 进行后续更新：

```tsx
// vite.config.ts
import { tanstackStart } from "@tanstack/react-start/plugin/vite";
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [
    tanstackStart({
      prerender: {
        routes: ["/blog", "/blog/posts/*"],
        crawlLinks: true,
      },
    }),
  ],
});
```

---

## 调试 ISR (Debugging ISR)

### 检查缓存响应头

使用浏览器开发者工具或 `curl` 命令检查缓存头信息：

```bash
curl -I [https://yoursite.com/blog/my-post](https://yoursite.com/blog/my-post)

# 重点关注以下字段：
# Cache-Control: public, max-age=3600, stale-while-revalidate=86400
# Age: 1234 (已在缓存中存在的时间，单位秒)
# X-Cache: HIT (表示命中 CDN 缓存)
```

### 测试重新验证逻辑

强制缓存失效以测试页面再生功能：

```bash
# Cloudflare: 绕过缓存直接请求源站
curl -H "Cache-Control: no-cache" [https://yoursite.com/page](https://yoursite.com/page)

# 或者使用 CDN 特定的缓存清除 (Purge) API
```

### 监控性能指标

跟踪以下核心指标：

- **缓存命中率 (Cache Hit Rate)**：由缓存提供服务的请求百分比。
- **重新验证时间 (Revalidation Time)**：再生过期内容所需的时间。
- **首字节时间 (TTFB)**：缓存内容的 TTFB 应该非常低。

---

## 相关资源 (Related Resources)

- [静态预渲染 (Static Prerendering)](./static-prerendering.md) —— 构建时页面生成指南。
- [托管部署 (Hosting)](./hosting.md) —— 各类 CDN 部署配置。
- [服务器函数 (Server Functions)](./server-functions.md) —— 创建动态数据端点。
- [数据加载 (Data Loading)](../../../../router/guide/data-loading.md) —— 客户端缓存控制。
- [中间件 (Middleware)](./middleware.md) —— 请求/响应自定义处理。

---

**增量静态再生 (ISR) 章节翻译已全部完成！**

接下来，你想翻译哪一个？

- file: ./contents/frontend/tanstack/middleware.md (中间件)
- file: ./contents/frontend/tanstack/protection.md (导入保护)
- file: ./contents/frontend/tanstack/server-entry-point.md (服务器入口点)
- file: ./contents/frontend/tanstack/spa-mode.md (SPA 模式)
- file: ./contents/frontend/tanstack/api-routes.md (API 路由)
- file: ./contents/frontend/tanstack/hosting.md (托管部署)
