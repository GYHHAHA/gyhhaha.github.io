---
short_title: 环境函数
---

# 环境函数 (Environment Functions)

## 什么是环境函数？

环境函数（Environment Functions）是用于根据运行时环境（无论是客户端还是服务器）定义和控制函数执行的工具。这些工具通过确保环境特定的逻辑被安全且有目的地执行，帮助防止运行时错误，并提高全栈或同构应用程序的可维护性。

TanStack Start 提供了三个核心环境函数：

- `createIsomorphicFn`：组合一个能够自适应客户端和服务器环境的单函数。
- `createServerOnlyFn`：创建一个只能在服务器端运行的函数。
- `createClientOnlyFn`：创建一个只能在客户端运行的函数。

---

## 同构函数 (Isomorphic Functions)

使用 `createIsomorphicFn()` 可以定义在客户端或服务器端调用时表现不同的函数。这对于在环境之间安全地共享逻辑，同时将特定环境的行为委托给相应的处理器非常有用。

### 完整实现

```tsx
import { createIsomorphicFn } from "@tanstack/react-start";

const getEnv = createIsomorphicFn()
  .server(() => "server")
  .client(() => "client");

const env = getEnv();
// ℹ️ 在 **服务器端**，它返回 `'server'`。
// ℹ️ 在 **客户端**，它返回 `'client'`。
```

### 仅服务器实现 (Partial Implementation)

以下是仅包含服务器端实现的 `createIsomorphicFn()` 示例：

```tsx
import { createIsomorphicFn } from "@tanstack/react-start";

const serverImplementationOnly = createIsomorphicFn().server(() => "server");

const server = serverImplementationOnly();
// ℹ️ 在 **服务器端**，它返回 `'server'`。
// ℹ️ 在 **客户端**，它是一个空操作（返回 `undefined`）。
```

### 仅客户端实现 (Partial Implementation)

以下是仅包含客户端实现的 `createIsomorphicFn()` 示例：

```tsx
import { createIsomorphicFn } from "@tanstack/react-start";

const clientImplementationOnly = createIsomorphicFn().client(() => "client");

const client = clientImplementationOnly();
// ℹ️ 在 **服务器端**，它是一个空操作（返回 `undefined`）。
// ℹ️ 在 **客户端**，它返回 `'client'`。
```

### 无实现

以下是没有定义任何环境特定实现的 `createIsomorphicFn()` 示例：

```tsx
import { createIsomorphicFn } from "@tanstack/react-start";

const noImplementation = createIsomorphicFn();

const noop = noImplementation();
// ℹ️ 在 **客户端** 和 **服务器端**，它都是空操作（返回 `undefined`）。
```

#### 什么是空操作 (no-op)？

空操作（no-op，全称 "no operation"）是指在执行时不执行任何操作的函数 —— 它只是简单地返回 `undefined` 而不执行任何逻辑。

```tsx
// 基础的空操作实现
function noop() {}
```

---

## 仅限环境 (Env-Only) 函数

`createServerOnlyFn` 和 `createClientOnlyFn` 工具函数强制执行严格的环境绑定。它们确保返回的函数只能在正确的运行时上下文中调用。如果被误用，它们会抛出描述性的运行时错误，以防止无意识的逻辑执行。

### `createServerOnlyFn`

```tsx
import { createServerOnlyFn } from "@tanstack/react-start";

const foo = createServerOnlyFn(() => "bar");

foo(); // ✅ 在服务器端：返回 "bar"
// ❌ 在客户端：抛出 "createServerOnlyFn() functions can only be called on the server!"
```

### `createClientOnlyFn`

```tsx
import { createClientOnlyFn } from "@tanstack/react-start";

const foo = createClientOnlyFn(() => "bar");

foo(); // ✅ 在客户端：返回 "bar"
// ❌ 在服务器端：抛出 "createClientOnlyFn() functions can only be called on the client!"
```

> [!NOTE]
> 这些函数对于 API 访问、文件系统读取、使用浏览器 API 或其他在目标环境之外无效或不安全的操作非常有用。

---

## Tree Shaking (摇树优化)

环境函数会根据每个生成的 Bundle 环境自动进行 Tree Shaking。

使用 `createIsomorphicFn()` 创建的函数支持 Tree Shaking。`.client()` 中的所有代码都不会包含在服务器 bundle 中，反之亦然。

在服务器上，使用 `createClientOnlyFn()` 创建的函数会被替换为一个在服务器端抛出 `Error` 的函数。对于客户端上的 `createServerOnlyFn` 函数，情况则正好相反。
