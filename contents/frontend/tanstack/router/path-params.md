---
short_title: 路径参数
---

# 路径参数 (Path Params)

路径参数用于匹配 URL 中的单个分段（即两个 `/` 之间的文本），并将其值作为**命名变量**返回给您。路径参数通过在路径中使用 `$` 字符前缀来定义，后面紧跟用于赋值的变量名。以下是有效的路径参数路径示例：

- `$postId`
- `$name`
- `$teamId`
- `about/$name`
- `team/$teamId`
- `blog/$postId`

由于路径参数路由仅匹配到下一个 `/` 为止，因此可以通过创建子路由来继续表达层级结构。

让我们创建一个使用路径参数来匹配帖子 ID 的路由文件：

```tsx title="src/routes/posts.$postId.tsx"
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ params }) => {
    // params.postId 将会自动获得类型定义
    return fetchPost(params.postId);
  },
});
```

## 子路由可以继承路径参数

一旦路径参数被解析，它对所有子路由都是可见的。这意味着如果我们为 `postRoute` 定义一个子路由，我们可以在子路由的路径中直接使用 URL 里的 `postId` 变量！

## 加载器 (Loaders) 中的路径参数

路径参数会作为一个 `params` 对象传递给加载器。该对象的键是路径参数的名称，值是从实际 URL 路径中解析出来的字符串。例如，如果我们访问 `/blog/123` URL，`params` 对象将是 `{ postId: '123' }`：

```tsx title="src/routes/posts.$postId.tsx"
export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ params }) => {
    return fetchPost(params.postId);
  },
});
```

`params` 对象也会被传递给 `beforeLoad` 选项：

```tsx title="src/routes/posts.$postId.tsx"
export const Route = createFileRoute("/posts/$postId")({
  beforeLoad: async ({ params }) => {
    // 使用 params.postId 进行权限检查或其他操作
  },
});
```

## 组件中的路径参数

如果我们为 `postRoute` 添加一个组件，可以通过使用路由的 `useParams` Hook 来访问 URL 中的 `postId` 变量：

```tsx title="src/routes/posts.$postId.tsx"
export const Route = createFileRoute("/posts/$postId")({
  component: PostComponent,
});

function PostComponent() {
  const { postId } = Route.useParams();
  return <div>帖子 ID: {postId}</div>;
}
```

> 🧠 **小技巧**：如果您的组件是经过代码分割（code-split）的，可以使用 [getRouteApi 函数](./code-splitting.md) 来避免为了获取类型化的 `useParams()` Hook 而导入整个路由配置。

## 在路由外部使用路径参数

您也可以使用全局导出的 `useParams` Hook 从应用中的任何组件访问已解析的路径参数。此时您需要将 `strict: false` 选项传递给 `useParams`，表示您正在从一个模糊的位置访问参数：

```tsx title="src/components/PostComponent.tsx"
import { useParams } from "@tanstack/react-router";

function PostComponent() {
  const { postId } = useParams({ strict: false });
  return <div>帖子 ID: {postId}</div>;
}
```

## 使用路径参数进行导航

当导航到带有路径参数的路由时，TypeScript 会要求您以对象形式或返回对象函数的形式传递 `params`。

让我们看看对象形式的写法：

```tsx
function Component() {
  return (
    <Link to="/blog/$postId" params={{ postId: "123" }}>
      帖子 123
    </Link>
  );
}
```

以下是函数形式的写法：

```tsx
function Component() {
  return (
    <Link to="/blog/$postId" params={(prev) => ({ ...prev, postId: "123" })}>
      帖子 123
    </Link>
  );
}
```

请注意，当您需要保留 URL 中已有的其他路由参数时，函数形式非常有用。它会接收当前的参数作为参数，允许您根据需要修改它们并返回最终的参数对象。

---

## 路径参数的前缀与后缀

您还可以为路径参数使用**前缀**和**后缀**来创建更复杂的路由模式。这允许您在捕获动态段的同时匹配特定的 URL 结构。

使用前缀或后缀时，可以通过将路径参数包裹在花括号 `{}` 中，并在变量名之前或之后放置文本来定义。

### 定义前缀

前缀定义在花括号外部、变量名之前。例如，如果您想匹配以 `post-` 开头后跟帖子 ID 的 URL，可以这样定义：

```tsx title="src/routes/posts/post-{$postId}.tsx"
export const Route = createFileRoute("/posts/post-{$postId}")({
  component: PostComponent,
});

function PostComponent() {
  const { postId } = Route.useParams();
  // postId 将会是 'post-' 之后的值
  return <div>帖子 ID: {postId}</div>;
}
```

您甚至可以将前缀与通配符路由（wildcard routes）结合使用：

```tsx title="src/routes/on-disk/storage-{$postId}/$.tsx"
export const Route = createFileRoute("/on-disk/storage-{$postId}/$")({
  component: StorageComponent,
});

function StorageComponent() {
  const { _splat } = Route.useParams();
  // _splat 将会是 'storage-' 之后的所有路径值
  // 例如：my-drive/documents/foo.txt
  return <div>存储位置: /{_splat}</div>;
}
```

### 定义后缀

后缀定义在花括号外部、变量名之后。例如，如果您想匹配以 `.txt` 结尾的文件名，可以这样定义：

```tsx title="src/routes/files/{$fileName}[.]txt.tsx"
// 注意：在文件名中使用 [.] 来转义点号
export const Route = createFileRoute("/files/{$fileName}.txt")({
  component: FileComponent,
});

function FileComponent() {
  const { fileName } = Route.useParams();
  // fileName 将会是 '.txt' 之前的值
  return <div>文件名: {fileName}</div>;
}
```

同样可以结合通配符使用：

```tsx title="src/routes/files/{$}[.]txt.tsx"
export const Route = createFileRoute("/files/{$}.txt")({
  component: FileComponent,
});

function FileComponent() {
  const { _splat } = Route.useParams();
  // _splat 将会是 '.txt' 之前的所有路径值
  return <div>文件路径通配符: {_splat}</div>;
}
```

### 结合前缀与后缀

您可以同时使用前缀和后缀。例如，匹配以 `user-` 开头并以 `.json` 结尾的 URL：

```tsx title="src/routes/users/user-{$userId}.json"
export const Route = createFileRoute("/users/user-{$userId}.json")({
  component: UserComponent,
});

function UserComponent() {
  const { userId } = Route.useParams();
  // userId 将会是 'user-' 和 '.json' 之间的值
  return <div>用户 ID: {userId}</div>;
}
```

就像之前的例子一样，您可以对前缀、后缀和通配符进行任意组合。尽情发挥吧！

## 可选路径参数 (Optional Path Parameters)

可选路径参数允许您定义在 URL 中可能存在也可能不存在的路由段。它们使用 `{-$paramName}` 语法，并提供了灵活的路由模式，使某些参数成为可选项。

### 定义可选参数

可选路径参数通过带有短横线前缀的花括号来定义：`{-$paramName}`。

```tsx
// 单个可选参数
// src/routes/posts/{-$category}.tsx
export const Route = createFileRoute("/posts/{-$category}")({
  component: PostsComponent,
});

// 多个可选参数
// src/routes/posts/{-$category}/{-$slug}.tsx
export const Route = createFileRoute("/posts/{-$category}/{-$slug}")({
  component: PostComponent,
});

// 混合必选参数和可选参数
// src/routes/users/$id/{-$tab}.tsx
export const Route = createFileRoute("/users/$id/{-$tab}")({
  component: UserComponent,
});
```

### 可选参数的工作原理

可选参数可以创建灵活的 URL 匹配模式：

- `/posts/{-$category}` 既匹配 `/posts` 也匹配 `/posts/tech`。
- `/posts/{-$category}/{-$slug}` 匹配 `/posts`、`/posts/tech` 以及 `/posts/tech/hello-world`。
- `/users/$id/{-$tab}` 匹配 `/users/123` 和 `/users/123/settings`。

当 URL 中不存在可选参数时，其值在路由处理器（Handlers）和组件中将为 `undefined`。

### 访问可选参数

在组件中访问可选参数的方式与普通参数完全相同，但它们的值可能是 `undefined`：

```tsx title="src/routes/posts/{-$category}.tsx"
function PostsComponent() {
  const { category } = Route.useParams();

  return (
    <div>{category ? `正在查看 ${category} 分类下的帖子` : "所有帖子"}</div>
  );
}
```

### 加载器 (Loaders) 中的可选参数

可选参数在加载器中可用，且值可能为 `undefined`：

```tsx
export const Route = createFileRoute("/posts/{-$category}")({
  loader: async ({ params }) => {
    // params.category 可能是 undefined
    return fetchPosts({ category: params.category });
  },
});
```

### beforeLoad 中的可选参数

可选参数在 `beforeLoad` 处理器中同样有效：

```tsx
export const Route = createFileRoute("/posts/{-$category}")({
  beforeLoad: async ({ params }) => {
    if (params.category) {
      // 验证分类是否存在
      await validateCategory(params.category);
    }
  },
});
```

---

### 高级可选参数模式

#### 配合前缀与后缀

可选参数支持前缀和后缀模式：

```tsx title="src/routes/files/prefix{-$name}.txt"
// 路由: /files/prefix{-$name}.txt
// 匹配: /files/prefix.txt 和 /files/prefixdocument.txt
export const Route = createFileRoute("/files/prefix{-$name}.txt")({
  component: FileComponent,
});

function FileComponent() {
  const { name } = Route.useParams();
  return <div>文件名: {name || "默认文件"}</div>;
}
```

#### 全可选参数

您可以创建所有参数均为可选的路由：

```tsx title="src/routes/{-$year}/{-$month}/{-$day}.tsx"
// 路由: /{-$year}/{-$month}/{-$day}
// 匹配: /、/2023、/2023/12、/2023/12/25
export const Route = createFileRoute("/{-$year}/{-$month}/{-$day}")({
  component: DateComponent,
});

function DateComponent() {
  const { year, month, day } = Route.useParams();

  if (!year) return <div>请选择年份</div>;
  if (!month) return <div>年份: {year}</div>;
  if (!day)
    return (
      <div>
        月份: {year}/{month}
      </div>
    );

  return (
    <div>
      日期: {year}/{month}/{day}
    </div>
  );
}
```

#### 可选参数配合通配符

可选参数可以与通配符结合，用于复杂的路由模式：

```tsx title="src/routes/docs/{-$version}/$.tsx"
// 路由: /docs/{-$version}/$
// 匹配: /docs/extra/path, /docs/v2/extra/path
export const Route = createFileRoute("/docs/{-$version}/$")({
  component: DocsComponent,
});

function DocsComponent() {
  const { version } = Route.useParams();
  const { _splat } = Route.useParams();

  return (
    <div>
      版本: {version || "最新版本"}
      路径: {_splat}
    </div>
  );
}
```

---

### 使用可选参数进行导航

在导航到带有可选参数的路由时，您可以精细控制包含哪些参数：

```tsx
function Navigation() {
  return (
    <div>
      {/* 携带可选参数进行导航 */}
      <Link to="/posts/{-$category}" params={{ category: "tech" }}>
        技术类文章
      </Link>

      {/* 不带可选参数进行导航 (移除参数) */}
      <Link to="/posts/{-$category}" params={{ category: undefined }}>
        全部文章
      </Link>

      {/* 携带多个可选参数进行导航 */}
      <Link
        to="/posts/{-$category}/{-$slug}"
        params={{ category: "tech", slug: "react-tips" }}
      >
        特定文章
      </Link>
    </div>
  );
}
```

### 可选参数的类型安全

TypeScript 为可选参数提供完整的类型安全支持：

```tsx title="src/routes/posts/{-$category}.tsx"
function PostsComponent() {
  // TypeScript 知道 category 可能是 undefined
  const { category } = Route.useParams() // 类型推断为: string | undefined

  // 安全访问
  const categoryUpper = category?.toUpperCase()

  return <div>{categoryUpper || '所有分类'}</div>
}

// 导航同样具备类型安全和灵活性
<Link
  to="/posts/{-$category}"
  params={{ category: 'tech' }} // ✅ 有效 - 字符串
>
  技术文章
</Link>

<Link
  to="/posts/{-$category}"
  params={{ category: 123 }} // ✅ 有效 - 数字（将自动转换为字符串）
>
  分类 123
</Link>
```

## 结合可选路径参数实现国际化 (i18n)

可选路径参数是实现国际化 (i18n) 路由模式的绝佳选择。您可以利用前缀模式来处理多种语言，同时保持 URL 的简洁和 SEO 友好。

### 基于前缀的 i18n

使用可选的语言前缀来支持诸如 `/en/about`、`/fr/about` 这样的 URL，或者直接用 `/about`（代表默认语言）：

```tsx title="src/routes/{-$locale}/about.tsx"
// 路由: /{-$locale}/about
export const Route = createFileRoute("/{-$locale}/about")({
  component: AboutComponent,
});

function AboutComponent() {
  const { locale } = Route.useParams();
  const currentLocale = locale || "en"; // 默认为英语

  const content = {
    en: { title: "About Us", description: "Learn more about our company." },
    fr: {
      title: "À Propos",
      description: "En savoir plus sur notre entreprise.",
    },
    es: {
      title: "Acerca de",
      description: "Conoce más sobre nuestra empresa.",
    },
  };

  return (
    <div>
      <h1>{content[currentLocale]?.title}</h1>
      <p>{content[currentLocale]?.description}</p>
    </div>
  );
}
```

此模式可匹配：

- `/about` (默认区域设置)
- `/en/about` (显式指定英语)
- `/fr/about` (法语)
- `/es/about` (西班牙语)

---

### 复杂的 i18n 模式

结合多个可选参数来实现更复杂的 i18n 路由：

```tsx title="src/routes/{-$locale}/blog/{-$category}/$slug.tsx"
// 路由: /{-$locale}/blog/{-$category}/$slug
export const Route = createFileRoute("/{-$locale}/blog/{-$category}/$slug")({
  beforeLoad: async ({ params }) => {
    const locale = params.locale || "en";
    const category = params.category;

    // 验证区域设置和分类
    const validLocales = ["en", "fr", "es", "de"];
    if (locale && !validLocales.includes(locale)) {
      throw new Error("无效的区域设置");
    }

    return { locale, category };
  },
  loader: async ({ params, context }) => {
    const { locale } = context;
    const { slug, category } = params;

    return fetchBlogPost({ slug, category, locale });
  },
  component: BlogPostComponent,
});

function BlogPostComponent() {
  const { locale, category, slug } = Route.useParams();
  const data = Route.useLoaderData();

  return (
    <article>
      <h1>{data.title}</h1>
      <p>
        分类: {category || "全部"} | 语言: {locale || "en"}
      </p>
      <div>{data.content}</div>
    </article>
  );
}
```

这支持如下 URL：

- `/blog/tech/my-post` (默认区域设置，技术分类)
- `/fr/blog/my-post` (法语，无分类)
- `/en/blog/tech/my-post` (显式英语，技术分类)
- `/es/blog/tecnologia/mi-post` (西班牙语，西班牙语分类)

---

### 语言切换 (Language Navigation)

使用函数式参数配合可选的 i18n 参数来创建语言切换器：

```tsx title="src/components/LanguageSwitcher.tsx"
function LanguageSwitcher() {
  const currentParams = useParams({ strict: false });

  const languages = [
    { code: "en", name: "English" },
    { code: "fr", name: "Français" },
    { code: "es", name: "Español" },
  ];

  return (
    <div className="language-switcher">
      {languages.map(({ code, name }) => (
        <Link
          key={code}
          to="/{-$locale}/blog/{-$category}/$slug"
          params={(prev) => ({
            ...prev,
            locale: code === "en" ? undefined : code, // 英语不显示前缀，以保持 URL 简洁
          })}
          className={currentParams.locale === code ? "active" : ""}
        >
          {name}
        </Link>
      ))}
    </div>
  );
}
```

您还可以创建更复杂的语言切换逻辑：

```tsx
function AdvancedLanguageSwitcher() {
  const currentParams = useParams({ strict: false });

  const handleLanguageChange = (newLocale: string) => {
    return (prev: any) => {
      // 保留所有现有参数，仅更新区域设置
      const updatedParams = { ...prev };

      if (newLocale === "en") {
        // 移除英语的区域前缀，以保持 URL 简洁
        delete updatedParams.locale;
      } else {
        updatedParams.locale = newLocale;
      }

      return updatedParams;
    };
  };

  return (
    <div className="language-switcher">
      <Link
        to="/{-$locale}/blog/{-$category}/$slug"
        params={handleLanguageChange("fr")}
      >
        Français
      </Link>

      <Link
        to="/{-$locale}/blog/{-$category}/$slug"
        params={handleLanguageChange("es")}
      >
        Español
      </Link>

      <Link
        to="/{-$locale}/blog/{-$category}/$slug"
        params={handleLanguageChange("en")}
      >
        English
      </Link>
    </div>
  );
}
```

### 使用可选参数的高级 i18n (Advanced i18n with Optional Parameters)

利用可选参数组织 i18n 路由，可以实现极其灵活的语言处理，同时保持简洁的文件结构：

```tsx
// 路由结构示例：
// routes/
//   {-$locale}/
//     index.tsx        // 对应路径：/, /en, /fr
//     about.tsx        // 对应路径：/about, /en/about, /fr/about
//     blog/
//       index.tsx      // 对应路径：/blog, /en/blog, /fr/blog
//       $slug.tsx      // 对应路径：/blog/post, /en/blog/post, /fr/blog/post

// routes/{-$locale}/index.tsx
export const Route = createFileRoute("/{-$locale}/")({
  component: HomeComponent,
});

function HomeComponent() {
  const { locale } = Route.useParams();
  // 检查是否为从右向左（RTL）阅读的语言
  const isRTL = ["ar", "he", "fa"].includes(locale || "");

  return (
    <div dir={isRTL ? "rtl" : "ltr"}>
      <h1>欢迎 ({locale || "en"})</h1>
      {/* 国际化内容 */}
    </div>
  );
}

// routes/{-$locale}/about.tsx
export const Route = createFileRoute("/{-$locale}/about")({
  component: AboutComponent,
});
```

---

### SEO 与规范链接 (SEO and Canonical URLs)

正确处理 i18n 路由的 SEO 对搜索引擎排名至关重要：

```tsx title="src/routes/{-$locale}/products/$id.tsx"
export const Route = createFileRoute("/{-$locale}/products/$id")({
  component: ProductComponent,
  head: ({ params, loaderData }) => {
    const locale = params.locale || "en";
    const product = loaderData;

    return {
      title: product.title[locale] || product.title.en,
      meta: [
        {
          name: "description",
          content: product.description[locale] || product.description.en,
        },
        {
          property: "og:locale",
          content: locale,
        },
      ],
      links: [
        // 规范链接 (Canonical URL)：通常始终指向默认语言格式
        {
          rel: "canonical",
          href: `https://example.com/products/${params.id}`,
        },
        // 备用语言版本 (Alternate language versions)
        {
          rel: "alternate",
          hreflang: "en",
          href: `https://example.com/products/${params.id}`,
        },
        {
          rel: "alternate",
          hreflang: "fr",
          href: `https://example.com/fr/products/${params.id}`,
        },
        {
          rel: "alternate",
          hreflang: "es",
          href: `https://example.com/es/products/${params.id}`,
        },
      ],
    };
  },
});
```

---

### i18n 的类型安全 (Type Safety for i18n)

确保您的 i18n 实现具备严格的类型校验，防止非法语言代码进入系统：

```tsx title="src/routes/{-$locale}/shop/{-$category}.tsx"
// 定义受支持的区域设置
type Locale = "en" | "fr" | "es" | "de";

// 类型安全的区域设置验证函数
function validateLocale(locale: string | undefined): locale is Locale {
  return ["en", "fr", "es", "de"].includes(locale as Locale);
}

export const Route = createFileRoute("/{-$locale}/shop/{-$category}")({
  beforeLoad: async ({ params }) => {
    const { locale } = params;

    // 类型安全的区域设置验证
    if (locale && !validateLocale(locale)) {
      // 如果语言代码非法，重定向到默认路径
      throw redirect({
        to: "/shop/{-$category}",
        params: { category: params.category },
      });
    }

    return {
      locale: (locale as Locale) || "en",
      isDefaultLocale: !locale || locale === "en",
    };
  },
  component: ShopComponent,
});

function ShopComponent() {
  const { locale, category } = Route.useParams();
  const { isDefaultLocale } = Route.useRouteContext();

  // TypeScript 现在知道 locale 是 Locale | undefined
  // 并且我们已经在 beforeLoad 中对其进行了验证

  return (
    <div>
      <h1>商店 {category ? `- ${category}` : ""}</h1>
      <p>语言: {locale || "en"}</p>
      {!isDefaultLocale && (
        <Link to="/shop/{-$category}" params={{ category }}>
          查看英文版
        </Link>
      )}
    </div>
  );
}
```

可选路径参数为您在 TanStack Router 应用中实现国际化提供了强大而灵活的基础。无论您偏好基于前缀的方法还是组合方法，都可以在保持卓越开发体验和类型安全的同时，创建简洁且 SEO 友好的 URL。

---

## 允许的字符 (Allowed Characters)

默认情况下，路径参数会使用 `encodeURIComponent` 进行转义。如果您希望在路径中允许其他有效的 URI 字符（例如 `@` 或 `+`），可以在 [RouterOptions](../api/router/RouterOptionsType.md#pathparamsallowedcharacters-property) 中进行指定。

使用示例：

```tsx
const router = createRouter({
  // ...
  // 允许在路径参数中直接显示 @ 符号
  pathParamsAllowedCharacters: ["@"],
});
```

以下是支持的可选允许字符列表：

- `;`
- `:`
- `@`
- `&`
- `=`
- `+`
- `$`
- `,`
