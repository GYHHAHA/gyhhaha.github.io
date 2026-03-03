---
short_title: Markdown 渲染
---

# Markdown 渲染 (Rendering Markdown)

本指南涵盖了在 TanStack Start 应用程序中导入和渲染 Markdown 内容的两种方法：

1. **静态 Markdown**：配合 `content-collections` 用于构建时加载（例如：博客文章）。
2. **动态 Markdown**：在运行时从 GitHub 或任何远程源获取。

这两种方法都使用基于 `unified` 生态系统的通用渲染流水线。

---

## 设置 Markdown 处理器 (Markdown Processor)

两种方法都使用相同的 Markdown 到 HTML 的处理流水线。首先，安装所需的依赖项：

```bash
npm install unified remark-parse remark-gfm remark-rehype rehype-raw rehype-slug rehype-autolink-headings rehype-stringify shiki html-react-parser gray-matter
```

创建一个 Markdown 处理器工具函数：

```tsx
// src/utils/markdown.ts
import { unified } from "unified";
import remarkParse from "remark-parse";
import remarkGfm from "remark-gfm";
import remarkRehype from "remark-rehype";
import rehypeRaw from "rehype-raw";
import rehypeSlug from "rehype-slug";
import rehypeAutolinkHeadings from "rehype-autolink-headings";
import rehypeStringify from "rehype-stringify";

export type MarkdownHeading = {
  id: string;
  text: string;
  level: number;
};

export type MarkdownResult = {
  markup: string;
  headings: Array<MarkdownHeading>;
};

export async function renderMarkdown(content: string): Promise<MarkdownResult> {
  const headings: Array<MarkdownHeading> = [];

  const result = await unified()
    .use(remarkParse) // 解析 Markdown
    .use(remarkGfm) // 支持 GitHub 风格的 Markdown (GFM)
    .use(remarkRehype, { allowDangerousHtml: true }) // 转换为 HTML AST
    .use(rehypeRaw) // 处理 Markdown 中的原始 HTML
    .use(rehypeSlug) // 为标题添加 ID
    .use(rehypeAutolinkHeadings, {
      behavior: "wrap",
      properties: { className: ["anchor"] },
    })
    .use(() => (tree) => {
      // 提取标题用于生成目录 (TOC)
      const { visit } = require("unist-util-visit");
      const { toString } = require("hast-util-to-string");

      visit(tree, "element", (node: any) => {
        if (["h1", "h2", "h3", "h4", "h5", "h6"].includes(node.tagName)) {
          headings.push({
            id: node.properties?.id || "",
            text: toString(node),
            level: parseInt(node.tagName.charAt(1), 10),
          });
        }
      });
    })
    .use(rehypeStringify) // 序列化为 HTML 字符串
    .process(content);

  return {
    markup: String(result),
    headings,
  };
}
```

---

## 创建 Markdown 组件 (Markdown Component)

创建一个 React 组件，用于渲染处理后的 HTML 并支持自定义元素处理：

```tsx
// src/components/Markdown.tsx
import { useState, useEffect } from "react";
import { Link } from "@tanstack/react-router";
import parse, {
  type HTMLReactParserOptions,
  Element,
  domToReact,
} from "html-react-parser";
import { renderMarkdown, type MarkdownResult } from "~/utils/markdown";

type MarkdownProps = {
  content: string;
  className?: string;
};

export function Markdown({ content, className }: MarkdownProps) {
  const [result, setResult] = useState<MarkdownResult | null>(null);

  useEffect(() => {
    renderMarkdown(content).then(setResult);
  }, [content]);

  if (!result) {
    return <div className={className}>正在加载...</div>;
  }

  const options: HTMLReactParserOptions = {
    replace: (domNode) => {
      if (domNode instanceof Element) {
        // 自定义特定元素的渲染
        if (domNode.name === "a") {
          // 处理链接
          const href = domNode.attribs.href;
          if (href?.startsWith("/")) {
            // 内部链接 - 使用路由器的 Link 组件
            return (
              <Link to={href}>
                {domToReact(domNode.children as any, options)}
              </Link>
            );
          }
        }

        if (domNode.name === "img") {
          // 为图片添加懒加载
          return (
            <img
              {...domNode.attribs}
              loading="lazy"
              className="rounded-lg shadow-md"
            />
          );
        }
      }
    },
  };

  return <div className={className}>{parse(result.markup, options)}</div>;
}
```

## 方法 1：使用 content-collections 处理静态 Markdown

`content-collections` 包是处理存储在代码仓库中的静态内容（如博客文章）的理想选择。它在构建时处理 Markdown 文件，并提供对内容的类型安全访问。

### 安装

```bash
npm install @content-collections/core @content-collections/vite
```

### 配置

在项目根目录下创建一个 `content-collections.ts` 文件：

```tsx
// content-collections.ts
import { defineCollection, defineConfig } from "@content-collections/core";
import matter from "gray-matter";

function extractFrontMatter(content: string) {
  const { data, content: body, excerpt } = matter(content, { excerpt: true });
  return { data, body, excerpt: excerpt || "" };
}

const posts = defineCollection({
  name: "posts",
  directory: "./src/blog", // 存放 .md 文件的目录
  include: "*.md",
  schema: (z) => ({
    title: z.string(),
    published: z.string().date(),
    description: z.string().optional(),
    authors: z.string().array(),
  }),
  transform: ({ content, ...post }) => {
    const frontMatter = extractFrontMatter(content);

    // 提取页眉图片（文档中的第一张图片）
    const headerImageMatch = content.match(/!\[([^\]]*)\]\(([^)]+)\)/);
    const headerImage = headerImageMatch ? headerImageMatch[2] : undefined;

    return {
      ...post,
      slug: post._meta.path,
      excerpt: frontMatter.excerpt,
      description: frontMatter.data.description,
      headerImage,
      content: frontMatter.body,
    };
  },
});

export default defineConfig({
  collections: [posts],
});
```

### Vite 集成

将 content-collections 插件添加到你的 Vite 配置中：

```tsx
// app.config.ts
import { defineConfig } from "@tanstack/react-start/config";
import contentCollections from "@content-collections/vite";

export default defineConfig({
  vite: {
    plugins: [contentCollections()],
  },
});
```

### 创建博客文章

在指定的目录下创建 Markdown 文件：

````markdown
---
title: 你好，世界
published: 2024-01-15
authors:
  - 张三
description: 我的第一篇博客文章
---

![页眉图片](/images/hero.jpg)

欢迎来到我的博客！这是我的第一篇文章。

## 快速开始

这里有一些包含 **加粗** 和 _斜体_ 文本的内容。

```javascript
console.log("Hello, world!");
```
````

### 使用集合 (Collection)

通过生成的集合访问你的文章：

```tsx
// src/routes/blog.index.tsx
import { createFileRoute, Link } from "@tanstack/react-router";
import { allPosts } from "content-collections";

export const Route = createFileRoute("/blog/")({
  component: BlogIndex,
});

function BlogIndex() {
  // 按发布日期对文章进行排序
  const sortedPosts = [...allPosts].sort(
    (a, b) => new Date(b.published).getTime() - new Date(a.published).getTime(),
  );

  return (
    <div>
      <h1>博客</h1>
      <ul>
        {sortedPosts.map((post) => (
          <li key={post.slug}>
            <Link to="/blog/$slug" params={{ slug: post.slug }}>
              <h2>{post.title}</h2>
              <p>{post.excerpt}</p>
              <span>{post.published}</span>
            </Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### 渲染单篇文章

```tsx
// src/routes/blog.$slug.tsx
import { createFileRoute, notFound } from "@tanstack/react-router";
import { allPosts } from "content-collections";
import { Markdown } from "~/components/Markdown";

export const Route = createFileRoute("/blog/$slug")({
  loader: ({ params }) => {
    // 在生成的集合中查找匹配的文章
    const post = allPosts.find((p) => p.slug === params.slug);
    if (!post) {
      throw notFound();
    }
    return post;
  },
  component: BlogPost,
});

function BlogPost() {
  const post = Route.useLoaderData();

  return (
    <article>
      <header>
        <h1>{post.title}</h1>
        <p>
          作者：{post.authors.join(", ")} | 发布日期：{post.published}
        </p>
      </header>
      {/* 使用我们之前创建的 Markdown 组件 */}
      <Markdown content={post.content} className="prose" />
    </article>
  );
}
```

---

## 方法 2：从远程源获取动态 Markdown

对于存储在外部（如 GitHub 仓库）的内容，你可以使用服务器函数（server functions）动态地获取并渲染 Markdown。

### 创建获取工具函数 (Fetch Utility)

```tsx
// src/utils/docs.server.ts
import { createServerFn } from "@tanstack/react-start";
import matter from "gray-matter";

type FetchDocsParams = {
  repo: string; // 例如：'tanstack/router'
  branch: string; // 例如：'main'
  filePath: string; // 例如：'docs/guide/getting-started.md'
};

export const fetchDocs = createServerFn({ method: "GET" })
  .inputValidator((params: FetchDocsParams) => params)
  .handler(async ({ data: { repo, branch, filePath } }) => {
    const url = `https://raw.githubusercontent.com/${repo}/${branch}/${filePath}`;

    const response = await fetch(url, {
      headers: {
        // 对于私有仓库或更高的速率限制，可以添加 GitHub Token
        // Authorization: `token ${process.env.GITHUB_TOKEN}`,
      },
    });

    if (!response.ok) {
      throw new Error(`获取失败：${response.status}`);
    }

    const rawContent = await response.text();
    // 解析 Frontmatter
    const { data: frontmatter, content } = matter(rawContent);

    return {
      frontmatter,
      content,
      filePath,
    };
  });
```

### 添加缓存头 (Cache Headers)

为了生产环境的性能，建议添加适当的缓存头：

```tsx
export const fetchDocs = createServerFn({ method: "GET" })
  .inputValidator((params: FetchDocsParams) => params)
  .handler(async ({ data: { repo, branch, filePath }, context }) => {
    // 设置响应头以便 CDN 缓存
    context.response.headers.set(
      "Cache-Control",
      "public, max-age=0, must-revalidate",
    );
    context.response.headers.set(
      "CDN-Cache-Control",
      "max-age=300, stale-while-revalidate=300",
    );

    // ... 后续获取逻辑同上
  });
```

### 在路由中使用动态 Markdown

```tsx
// src/routes/docs.$path.tsx
import { createFileRoute } from "@tanstack/react-router";
import { fetchDocs } from "~/utils/docs.server";
import { Markdown } from "~/components/Markdown";

export const Route = createFileRoute("/docs/$path")({
  loader: async ({ params }) => {
    return fetchDocs({
      data: {
        repo: "你的组织/你的仓库",
        branch: "main",
        filePath: `docs/${params.path}.md`,
      },
    });
  },
  component: DocsPage,
});

function DocsPage() {
  const { frontmatter, content } = Route.useLoaderData();

  return (
    <article>
      <h1>{frontmatter.title}</h1>
      <Markdown content={content} className="prose" />
    </article>
  );
}
```

### 获取目录内容

为了根据 GitHub 目录构建导航菜单：

```tsx
// src/utils/docs.server.ts
type GitHubContent = {
  name: string;
  path: string;
  type: "file" | "dir";
};

export const fetchRepoContents = createServerFn({ method: "GET" })
  .inputValidator(
    (params: { repo: string; branch: string; path: string }) => params,
  )
  .handler(async ({ data: { repo, branch, path } }) => {
    const url = `https://api.github.com/repos/${repo}/contents/${path}?ref=${branch}`;

    const response = await fetch(url, {
      headers: {
        Accept: "application/vnd.github.v3+json",
        // 对于私有仓库，请添加 GitHub Token
        // Authorization: `token ${process.env.GITHUB_TOKEN}`,
      },
    });

    if (!response.ok) {
      throw new Error(`获取内容失败：${response.status}`);
    }

    const contents: Array<GitHubContent> = await response.json();

    // 仅过滤出 .md 文件并处理路径
    return contents
      .filter((item) => item.type === "file" && item.name.endsWith(".md"))
      .map((item) => ({
        name: item.name.replace(".md", ""),
        path: item.path,
      }));
  });
```

---

## 使用 Shiki 添加语法高亮

为了给代码块添加美观的语法高亮，可以将 Shiki 集成到你的 Markdown 处理器中：

```tsx
// src/utils/markdown.ts
import { codeToHtml } from "shiki";

// 解析后处理代码块
export async function highlightCode(
  code: string,
  language: string,
): Promise<string> {
  return codeToHtml(code, {
    lang: language,
    themes: {
      light: "github-light",
      dark: "tokyo-night",
    },
  });
}
```

接着在你的 Markdown 组件中处理代码块渲染：

```tsx
// 在 Markdown 组件的 replace 函数内部
if (domNode.name === "pre") {
  const codeElement = domNode.children.find(
    (child) => child instanceof Element && child.name === "code",
  );
  if (codeElement) {
    const className = (codeElement as Element).attribs.class || "";
    const language = className.replace("language-", "") || "text";
    const code = getText(codeElement);

    return <CodeBlock code={code} language={language} />;
  }
}
```

---

## 总结

| 方法                    | 适用场景                       | 优点                               | 缺点                           |
| :---------------------- | :----------------------------- | :--------------------------------- | :----------------------------- |
| **content-collections** | 博客文章、随应用打包的静态文档 | 类型安全、构建时处理、运行速度极快 | 内容更新需要重新构建应用       |
| **动态获取 (Dynamic)**  | 外部文档、更新频繁的内容       | 内容始终保持最新，无需重新构建     | 有运行时开销，需要处理网络错误 |

请根据你的内容更新频率和部署工作流选择最合适的方法。对于混合场景，你也可以在同一个应用程序中同时使用这两种方法。
