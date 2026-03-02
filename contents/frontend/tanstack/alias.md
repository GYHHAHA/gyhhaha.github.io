---
short_title: 路径别名
---

# 路径别名 (Path Aliases)

路径别名是 TypeScript 的一项非常实用的功能，它允许你为项目中目录结构较深的路径定义快捷方式。这可以帮助你避免在代码中使用冗长的相对导入，并使重构项目结构变得更加容易。这在解决多层级导入问题时尤为有效。

默认情况下，TanStack Start 并不包含路径别名配置。但是，你可以通过更新项目根目录下的 `tsconfig.json` 文件并添加以下配置，轻松地将其引入你的项目：

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "~/*": ["./src/*"]
    }
  }
}
```

在这个示例中，我们定义了路径别名 `~/*`，它映射到 `./src/*` 目录。这意味着你现在可以使用 `~` 前缀从 `src` 目录中导入文件。

更新 `tsconfig.json` 文件后，你需要安装 `vite-tsconfig-paths` 插件，以便在 TanStack Start 项目中启用路径别名。你可以通过运行以下命令来完成安装：

```sh
npm install -D vite-tsconfig-paths
```

接下来，你需要更新 `vite.config.ts` 文件以包含该插件：

```ts
// vite.config.ts
import { defineConfig } from "vite";
import viteTsConfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  vite: {
    plugins: [
      // 这个插件用于启用路径别名
      viteTsConfigPaths({
        projects: ["./tsconfig.json"],
      }),
    ],
  },
});
```

完成这些配置后，你就可以像下面这样使用路径别名导入文件了：

```ts
// app/routes/posts/$postId/edit.tsx
import { Input } from "~/components/ui/input";

// 从而替代以下冗长的写法：

import { Input } from "../../../components/ui/input";
```
