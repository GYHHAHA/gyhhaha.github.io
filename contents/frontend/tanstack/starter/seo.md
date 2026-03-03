---
short_title: SEO
---

# SEO (搜索引擎优化)

## SEO 到底是什么？

SEO（搜索引擎优化）常被误解为仅仅是“出现在 Google 上”，或者是一个库能自动提供的复选框。实际上，SEO 是一个广泛的学科，核心在于提供人们需要的有价值内容，并让他们能够轻松找到这些内容。

**技术 SEO (Technical SEO)** 是开发者直接接触最多的 SEO 子集。它涉及使用满足搜索引擎、爬虫、排名器甚至大语言模型（LLM）技术要求的工具和 API。当有人说一个框架具有“良好的 SEO 支持”时，通常是指它提供了让这一过程变得简单直观的工具。

TanStack Start 提供了全面的技术 SEO 能力，但你仍需要投入精力去有效地使用它们。

## TanStack Start 提供的功能

TanStack Start 为你提供了技术 SEO 的基石：

- **服务器端渲染 (SSR)** —— 确保爬虫接收到完整渲染的 HTML。
- **静态预渲染 (Static Prerendering)** —— 预先生成页面，以获得最佳性能和可抓取性。
- **文档头部管理 (Document Head Management)** —— 全面控制 Meta 标签、标题和结构化数据。
- **性能优化** —— 通过代码拆分、流式传输和优化的 Bundle 体积实现快速加载。

---

## 文档头部管理 (Document Head Management)

路由上的 `head` 属性是你的主要 SEO 工具。它允许你设置页面标题、Meta 描述、Open Graph 标签等。

### 基础 Meta 标签

```tsx
// src/routes/index.tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/")({
  head: () => ({
    meta: [
      { title: "我的应用 - 首页" },
      {
        name: "description",
        content: "欢迎来到我的应用，这是一个用于...",
      },
    ],
  }),
  component: HomePage,
});
```

### 动态 Meta 标签

使用加载器数据 (Loader Data) 为内容页面生成动态 Meta 标签：

```tsx
// src/routes/posts/$postId.tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ params }) => {
    const post = await fetchPost(params.postId);
    return { post };
  },
  // 使用从服务器获取的 loaderData 填充 Meta 信息
  head: ({ loaderData }) => ({
    meta: [
      { title: loaderData.post.title },
      { name: "description", content: loaderData.post.excerpt },
    ],
  }),
  component: PostPage,
});
```

### Open Graph 与社交分享

Open Graph 标签控制你的页面在社交媒体上分享时的显示方式：

```tsx
export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ params }) => {
    const post = await fetchPost(params.postId);
    return { post };
  },
  head: ({ loaderData }) => ({
    meta: [
      { title: loaderData.post.title },
      { name: "description", content: loaderData.post.excerpt },
      // Open Graph 协议标签
      { property: "og:title", content: loaderData.post.title },
      { property: "og:description", content: loaderData.post.excerpt },
      { property: "og:image", content: loaderData.post.coverImage },
      { property: "og:type", content: "article" },
      // Twitter 卡片标签
      { name: "twitter:card", content: "summary_large_image" },
      { name: "twitter:title", content: loaderData.post.title },
      { name: "twitter:description", content: loaderData.post.excerpt },
      { name: "twitter:image", content: loaderData.post.coverImage },
    ],
  }),
  component: PostPage,
});
```

### 规范链接 (Canonical URLs)

规范链接有助于防止重复内容问题：

```tsx
export const Route = createFileRoute("/posts/$postId")({
  head: ({ params }) => ({
    links: [
      {
        rel: "canonical",
        href: `https://myapp.com/posts/${params.postId}`,
      },
    ],
  }),
  component: PostPage,
});
```

---

## 结构化数据 (JSON-LD)

结构化数据可帮助搜索引擎理解你的内容，并能在搜索结果中启用富媒体搜索结果（Rich Results）：

```tsx
export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ params }) => {
    const post = await fetchPost(params.postId);
    return { post };
  },
  head: ({ loaderData }) => ({
    meta: [{ title: loaderData.post.title }],
    scripts: [
      {
        type: "application/ld+json",
        children: JSON.stringify({
          "@context": "[https://schema.org](https://schema.org)",
          "@type": "Article",
          headline: loaderData.post.title,
          description: loaderData.post.excerpt,
          image: loaderData.post.coverImage,
          author: {
            "@type": "Person",
            name: loaderData.post.author.name,
          },
          datePublished: loaderData.post.publishedAt,
        }),
      },
    ],
  }),
  component: PostPage,
});
```

---

## 服务器端渲染 (SSR)

TanStack Start 默认启用 SSR。这确保了搜索引擎爬虫能够接收到完整渲染的 HTML 内容，这对 SEO 至关重要。

```tsx
// SSR 是自动执行的 —— 你的页面将在服务器上渲染
export const Route = createFileRoute("/about")({
  component: AboutPage,
});
```

对于不需要 SSR 的路由，你可以选择性地将其禁用。但请注意，这可能会影响这些页面的 SEO：

```tsx
// 仅对不需要 SEO 的页面禁用 SSR
export const Route = createFileRoute("/dashboard")({
  ssr: false, // 控制面板页面不需要被索引
  component: DashboardPage,
});
```

更多详情请参阅 [选择性 SSR 指南](./ssr.md)。

---

## 静态预渲染 (Static Prerendering)

对于不经常更改的内容，静态预渲染可以在构建时生成 HTML，以获得最佳性能：

```ts
// vite.config.ts
import { tanstackStart } from "@tanstack/react-start/plugin/vite";

export default defineConfig({
  plugins: [
    tanstackStart({
      prerender: {
        enabled: true,
        crawlLinks: true,
      },
    }),
  ],
});
```

预渲染的页面加载速度更快，且易于抓取。配置选项请参阅 [静态预渲染指南](./static.md)。

---

## 站点地图 (Sitemaps)

### 内置站点地图生成

当你启用带有链接爬取功能的预渲染时，TanStack Start 可以自动生成站点地图：

```ts
// vite.config.ts
import { tanstackStart } from "@tanstack/react-start/plugin/vite";

export default defineConfig({
  plugins: [
    tanstackStart({
      prerender: {
        enabled: true,
        crawlLinks: true, // 发现所有可链接的页面
      },
      sitemap: {
        enabled: true,
        host: "[https://myapp.com](https://myapp.com)",
      },
    }),
  ],
});
```

站点地图是在构建时通过爬取路由中所有可发现的页面生成的。对于静态或大部分为静态的网站，这是推荐的做法。

### 静态站点地图 (Static Sitemap)

对于简单的站点，你也可以在 `public` 目录下放置一个静态的 `sitemap.xml` 文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="[http://www.sitemaps.org/schemas/sitemap/0.9](http://www.sitemaps.org/schemas/sitemap/0.9)">
  <url>
    <loc>[https://myapp.com/](https://myapp.com/)</loc>
    <changefreq>daily</changefreq>
    <priority>1.0</priority>
  </url>
  <url>
    <loc>[https://myapp.com/about](https://myapp.com/about)</loc>
    <changefreq>monthly</changefreq>
  </url>
</urlset>
```

当你的站点结构已知且不经常变动时，这种方法非常有效。

### 动态站点地图 (Dynamic Sitemap)

对于那些包含无法在构建时发现的动态内容的站点，你可以使用 [服务器路由 (server routes)](./routes.md) 创建动态站点地图。为了性能考虑，建议在你的 CDN 层面进行缓存：

```ts
// src/routes/sitemap[.]xml.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/sitemap.xml")({
  server: {
    handlers: {
      GET: async () => {
        const posts = await fetchAllPosts();

        const sitemap = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="[http://www.sitemaps.org/schemas/sitemap/0.9](http://www.sitemaps.org/schemas/sitemap/0.9)">
  <url>
    <loc>[https://myapp.com/](https://myapp.com/)</loc>
    <changefreq>daily</changefreq>
    <priority>1.0</priority>
  </url>
  ${posts
    .map(
      (post) => `
  <url>
    <loc>[https://myapp.com/posts/$](https://myapp.com/posts/$){post.id}</loc>
    <lastmod>${post.updatedAt}</lastmod>
    <changefreq>weekly</changefreq>
  </url>`,
    )
    .join("")}
</urlset>`;

        return new Response(sitemap, {
          headers: {
            "Content-Type": "application/xml",
          },
        });
      },
    },
  },
});
```

---

## robots.txt

### 静态 robots.txt

最简单的方法是在 `public` 目录下放置一个静态的 `robots.txt` 文件：

```txt
// public/robots.txt
User-agent: *
Allow: /

Sitemap: [https://myapp.com/sitemap.xml](https://myapp.com/sitemap.xml)
```

该文件将自动在 `/robots.txt` 路径下提供服务。这是大多数站点最常用的方法。

### 动态 robots.txt

对于更复杂的场景（例如针对不同环境设置不同的规则），你可以使用 [服务器路由 (server routes)](./routes.md) 创建 `robots.txt` 文件：

```ts
// src/routes/robots[.]txt.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/robots.txt")({
  server: {
    handlers: {
      GET: async () => {
        const robots = `User-agent: *
Allow: /

Sitemap: [https://myapp.com/sitemap.xml](https://myapp.com/sitemap.xml)`;

        return new Response(robots, {
          headers: {
            "Content-Type": "text/plain",
          },
        });
      },
    },
  },
});
```

---

## 最佳实践 (Best Practices)

### 性能至关重要 (Performance Matters)

页面速度是一个排名因素。TanStack Start 通过以下方式提供帮助：

- **自动代码拆分 (Automatic code-splitting)**：仅加载每个页面所需的 JavaScript。
- **流式传输 SSR (Streaming SSR)**：立即开始向浏览器发送 HTML。
- **预加载 (Preloading)**：在用户导航之前预取路由。

### 内容为王 (Content is King)

技术 SEO 只是拼图的一块。最重要的因素包括：

- **高质量内容**：创建能为用户提供价值的内容。
- **清晰的站点结构**：逻辑清晰地组织你的路由。
- **描述性 URL**：使用具有意义的路径片段（例如 `/posts/my-great-article` 优于 `/posts/123`）。
- **内部链接**：帮助用户和爬虫发现你的内容。

### 测试你的实现 (Test Your Implementation)

使用这些工具验证你的 SEO 实现：

- [Google Search Console](https://search.google.com/search-console)：监控索引和搜索表现。
- [Google 富媒体搜索结果测试](https://search.google.com/test/rich-results)：验证结构化数据。
- [Open Graph 调试器](https://developers.facebook.com/tools/debug/)：预览社交分享卡片。
- 浏览器开发人员工具：检查渲染后的 HTML 和 Meta 标签。

### 跟踪你的排名 (Track Your Rankings)

为了长期监控你的 SEO 表现，我们推荐 [Nozzle.io](https://nozzle.io?utm_source=tanstack)。Nozzle 提供企业级的排名跟踪，让你监控无限数量的关键词、跟踪 SERP 特性，并分析你相对于竞争对手的可见度。与传统的排名跟踪器不同，Nozzle 为每次查询存储完整的 SERP，提供完整的数据来了解你的页面在搜索结果中的表现。
