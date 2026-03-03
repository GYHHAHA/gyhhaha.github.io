---
short_title: 托管部署
---

# 托管部署 (Hosting)

托管是将你的应用程序部署到互联网上以便用户访问的过程。这是任何 Web 开发项目中至关重要的一环，确保你的应用能够面向全世界提供服务。TanStack Start 构建在 Vite 之上 —— Vite 是一个强大的开发/构建平台，它让我们能够将你的应用程序部署到任何托管提供商。

## 我该选择哪一个？

TanStack Start **旨在与任何托管提供商协同工作**。如果你已经有了心仪的平台，可以使用 TanStack Start 提供的全栈 API 将应用部署到那里。

然而，鉴于托管对应用的性能、可靠性和可扩展性至关重要，我们建议使用我们的 **官方托管合作伙伴**：[Cloudflare](https://www.cloudflare.com?utm_source=tanstack)、[Netlify](https://www.netlify.com?utm_source=tanstack) 或 [Railway](https://railway.com?utm_source=tanstack)。

---

## 部署目标

一旦选定了部署目标，你可以按照以下指南将你的 TanStack Start 应用部署到指定的托管提供商：

- **Cloudflare Workers**：部署到 Cloudflare Workers
- **Netlify**：部署到 Netlify
- **Railway**：部署到 Railway
- **Nitro**：使用 Nitro 引擎部署
- **Vercel**：部署到 Vercel
- **Node.js / Docker**：部署到 Node.js 服务器
- **Bun**：部署到 Bun 服务器
- **Appwrite Sites**：部署到 Appwrite Sites
- ... 更多支持正在路上！

---

### Cloudflare Workers ⭐ _官方合作伙伴_

在部署到 Cloudflare Workers 时，你需要完成几个额外的步骤。

**1. 安装插件与工具**

```bash
pnpm add -D @cloudflare/vite-plugin wrangler
```

**2. 在 `vite.config.ts` 中配置插件**

```ts
// vite.config.ts
import { defineConfig } from "vite";
import { tanstackStart } from "@tanstack/react-start/plugin/vite";
import { cloudflare } from "@cloudflare/vite-plugin";
import viteReact from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [
    // 启用 Cloudflare Vite 插件
    cloudflare({ viteEnvironment: { name: "ssr" } }),
    tanstackStart(),
    viteReact(),
  ],
});
```

**3. 添加 `wrangler.jsonc` 配置文件**

```json
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "tanstack-start-app",
  "compatibility_date": "2025-09-02",
  "compatibility_flags": ["nodejs_compat"],
  "main": "@tanstack/react-start/server-entry"
}
```

**4. 修改 `package.json` 中的脚本**

```json
{
  "scripts": {
    "dev": "vite dev",
    "build": "vite build && tsc --noEmit",
    // ============ 👇 删除下面这行 ============
    "start": "node .output/server/index.mjs",
    // ============ 👇 添加下面这几行 ============
    "preview": "vite preview",
    "deploy": "npm run build && wrangler deploy",
    "cf-typegen": "wrangler types"
  }
}
```

**5. 登录身份验证**

使用 Wrangler 登录你的 Cloudflare 账户：

```bash
npx wrangler login
```

**6. 执行部署**

```bash
pnpm run deploy
```

---

### Netlify ⭐ _官方合作伙伴_

安装并添加 [`@netlify/vite-plugin-tanstack-start`](https://www.npmjs.com/package/@netlify/vite-plugin-tanstack-start) 插件。该插件会自动为 Netlify 部署配置构建，并在本地开发中提供完整的 Netlify 生产平台仿真：

```bash
pnpm add --save-dev @netlify/vite-plugin-tanstack-start
```

```ts
// vite.config.ts
import { defineConfig } from "vite";
import { tanstackStart } from "@tanstack/react-start/plugin/vite";
import netlify from "@netlify/vite-plugin-tanstack-start";
import viteReact from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [
    tanstackStart(),
    netlify(), // 添加此插件
    viteReact(),
  ],
});
```

最后，使用 [Netlify CLI](https://developers.netlify.com/cli/) 部署：

```bash
npx netlify deploy
```

#### 手动配置方案

如果你更倾向于手动配置，可以在项目根目录添加 `netlify.toml` 文件：

```toml
[build]
  command = "vite build"
  publish = "dist/client"
[dev]
  command = "vite dev"
  port = 3000
```

### Railway ⭐ _官方合作伙伴_

Railway 提供零配置的即时部署。请先按照下文的 Nitro 部署说明进行操作，然后部署到 Railway：

1. 将代码推送到 GitHub 仓库。
2. 在 [railway.com](https://railway.com?utm_source=tanstack) 连接你的仓库。
3. Railway 将自动检测构建设置并部署你的应用程序。

**Railway 自动提供以下功能：**

- **自动部署**：每次推送代码时自动触发。
- **内置数据库**：支持 Postgres, MySQL, Redis, MongoDB。
- **预览环境**：为每个 Pull Request 提供独立的测试环境。
- **自动 HTTPS**：支持自定义域名并自动配置证书。

更多详情请参阅 [Railway 官方文档](https://docs.railway.com)。

---

### Nitro

[Nitro](https://v3.nitro.build/) 是一个平台无关的中间层，它允许你将 TanStack Start 应用程序部署到 [广泛的托管平台](https://v3.nitro.build/deploy) 上。

**⚠️ 注意：[`nitro/vite`](https://v3.nitro.build/) 插件原生集成了 Vite Environments API，作为 TanStack Start 的底层构建工具。该插件目前仍处于活跃开发阶段并定期更新。如果你遇到任何问题，请提交带有复现步骤的报告以便调查。**

通过在 `package.json` 中指定以下内容来安装 Nitro 的每夜版（nightly）：

```json
"nitro": "npm:nitro-nightly@latest"
```

```tsx
// vite.config.ts
import { tanstackStart } from "@tanstack/react-start/plugin/vite";
import { defineConfig } from "vite";
import { nitro } from "nitro/vite";
import viteReact from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [tanstackStart(), nitro(), viteReact()],
});
```

#### 性能技巧：FastResponse

如果你使用 Nitro 部署到 Node.js 环境（底层使用 [srvx](https://srvx.h3.dev/)），通过将全局 `Response` 构造函数替换为 srvx 优化的 `FastResponse`，可以将吞吐量提高约 5%。

首先，安装 srvx：

```bash
npm install srvx
```

然后在服务器入口点（`src/server.ts`）中添加：

```ts
import { FastResponse } from "srvx";
globalThis.Response = FastResponse;
```

这种优化之所以奏效，是因为 `FastResponse` 包含了一条优化的 `_toNodeResponse()` 路径，避免了标准 Web `Response` 转换为 Node.js 响应时的额外开销。**此优化仅适用于使用 Nitro/h3/srvx 的 Node.js 部署。**

---

### Vercel

请遵循 Nitro 的部署说明。
使用 Vercel 的一键部署流程将你的应用程序推向生产环境，即可完成部署！

---

### Node.js / Docker

请遵循 Nitro 的部署说明。使用 `node` 命令从构建输出文件中启动服务器。

确保 `package.json` 中包含 `build` 和 `start` 脚本：

```json
{
  "scripts": {
    "build": "vite build",
    "start": "node .output/server/index.mjs"
  }
}
```

构建并启动应用：

```sh
npm run build
npm run start
```

---

### Bun

> [!IMPORTANT]
> 目前，针对 Bun 的特定部署指南仅适用于 **React 19**。如果你使用的是 React 18，请参考 Node.js 部署指南。

确保 `package.json` 中的 `react` 和 `react-dom` 版本不低于 19.0.0：

```sh
bun install react@19 react-dom@19
```

遵循 Nitro 的部署说明。根据你调用构建的方式，你可能需要在 Nitro 配置中设置 `'bun'` 预设：

```ts
// vite.config.ts
import { tanstackStart } from "@tanstack/react-start/plugin/vite";
import { defineConfig } from "vite";
import { nitro } from "nitro/vite";
import viteReact from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [
    tanstackStart(),
    nitro({ preset: "bun" }), // 设置为 bun 预设
    viteReact(),
  ],
});
```

#### 使用 Bun 构建生产级服务器 (Production Server with Bun)

此外，你还可以使用利用 Bun 原生 API 的自定义服务器实现。

我们提供了一个参考实现，展示了构建生产就绪型 Bun 服务器的方法。该示例使用 Bun 原生函数以获得最佳性能，并包含**智能资源预加载**和**内存管理**等特性。

**这只是一个起点——你可以根据自己的需求灵活调整或简化。**

**该示例展示了：**

- 使用 Bun 的原生文件处理功能提供静态资源。
- 混合加载策略（内存预加载小文件，按需读取大文件）。
- 可选功能，如 ETag 支持和 Gzip 压缩。
- 生产环境就绪的缓存响应头。

**快速设置：**

1. 从示例仓库复制 [`server.ts`](https://github.com/tanstack/router/blob/main/examples/react/start-bun/server.ts) 文件到项目根目录（或作为你实现方案的灵感来源）。

2. 构建应用程序：

   ```sh
   bun run build
   ```

3. 启动服务器：
   ```sh
   bun run server.ts
   ```

**配置（可选）：**

参考服务器实现通过环境变量提供了多个可选配置项。你可以直接使用，也可以根据需要修改或删除功能：

```sh
# 基本用法 - 开箱即用
bun run server.ts

# 常用配置
PORT=8080 bun run server.ts                           # 自定义端口
ASSET_PRELOAD_VERBOSE_LOGGING=true bun run server.ts  # 查看详细运行日志
```

**可用环境变量：**

| 变量名                           | 描述                               | 默认值                                                                        |
| :------------------------------- | :--------------------------------- | :---------------------------------------------------------------------------- |
| `PORT`                           | 服务器端口                         | `3000`                                                                        |
| `ASSET_PRELOAD_MAX_SIZE`         | 预加载到内存的最大文件限制（字节） | `5242880` (5MB)                                                               |
| `ASSET_PRELOAD_INCLUDE_PATTERNS` | 包含文件的 Glob 模式（逗号分隔）   | 所有文件                                                                      |
| `ASSET_PRELOAD_EXCLUDE_PATTERNS` | 排除文件的 Glob 模式（逗号分隔）   | 无                                                                            |
| `ASSET_PRELOAD_VERBOSE_LOGGING`  | 启用详细日志                       | `false`                                                                       |
| `ASSET_PRELOAD_ENABLE_ETAG`      | 启用 ETag 生成                     | `true`                                                                        |
| `ASSET_PRELOAD_ENABLE_GZIP`      | 启用 Gzip 压缩                     | `true`                                                                        |
| `ASSET_PRELOAD_GZIP_MIN_SIZE`    | 启用 Gzip 的最小文件限制（字节）   | `1024` (1KB)                                                                  |
| `ASSET_PRELOAD_GZIP_MIME_TYPES`  | 允许 Gzip 的 MIME 类型             | `text/,application/javascript,application/json,application/xml,image/svg+xml` |

<details>
<summary>高级配置示例</summary>

```sh
# 优化最小内存占用
ASSET_PRELOAD_MAX_SIZE=1048576 bun run server.ts

# 仅预加载核心资源
ASSET_PRELOAD_INCLUDE_PATTERNS="*.js,*.css" \
ASSET_PRELOAD_EXCLUDE_PATTERNS="*.map,vendor-*" \
bun run server.ts

# 禁用可选功能
ASSET_PRELOAD_ENABLE_ETAG=false \
ASSET_PRELOAD_ENABLE_GZIP=false \
bun run server.ts

# 自定义 Gzip 配置
ASSET_PRELOAD_GZIP_MIN_SIZE=2048 \
ASSET_PRELOAD_GZIP_MIME_TYPES="text/,application/javascript,application/json" \
bun run server.ts
```

</details>

**示例输出：**

```txt
📦 Loading static assets from ./dist/client...
   Max preload size: 5.00 MB

📁 Preloaded into memory:
   /assets/index-a1b2c3d4.js           45.23 kB │ gzip:  15.83 kB
   /assets/index-e5f6g7h8.css          12.45 kB │ gzip:   4.36 kB

💾 Served on-demand:
   /assets/vendor-i9j0k1l2.js          245.67 kB │ gzip:  86.98 kB

✅ Preloaded 2 files (57.68 KB) into memory
🚀 Server running at http://localhost:3000
```

完整的运行示例请查看本仓库中的 [TanStack Start + Bun 示例](https://github.com/TanStack/router/tree/main/examples/react/start-bun)。

---

### Appwrite Sites

要部署到 [Appwrite Sites](https://appwrite.io/products/sites)，你需要完成以下步骤：

1. **创建一个 TanStack Start 应用**（或使用现有应用）

   ```bash
   npm create @tanstack/start@latest
   ```

2. **将项目推送到 GitHub 仓库**
   创建一个新的 [GitHub 仓库](https://github.com/new) 并推送代码。

3. **创建 Appwrite 项目**
   前往 [Appwrite Cloud](https://cloud.appwrite.io) 注册并创建你的第一个项目。

4. **部署站点**
   在 Appwrite 项目中，从侧边栏导航到 **Sites** 页面。点击 **Create site**，选择 **Connect a repository**，连接你的 GitHub 账号并选择对应的仓库。
   1. 选择 **生产分支 (production branch)** 和 **根目录 (root directory)**。
   2. 确认框架已识别为 **TanStack Start**。
   3. 确认构建设置：
      - **Install command:** `npm install`
      - **Build command:** `npm run build`
      - **Output directory:** 如果你使用 Nitro v2 或 v3，这应该是 `./.output`（默认通常显示为 `./dist`）。
   4. 添加所需的 **环境变量**。
   5. 点击 **Deploy**。

部署成功后，点击 **Visit site** 按钮即可查看你已上线的应用程序。
