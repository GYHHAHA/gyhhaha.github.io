---
short_title: 自定义搜索参数序列化
---

# 自定义搜索参数序列化 (Custom Search Param Serialization)

默认情况下，TanStack Router 会自动使用 `JSON.stringify` 和 `JSON.parse` 来解析和序列化 URL 搜索参数（Search Params）。除了对搜索对象进行序列化和反序列化外，此过程还涉及对查询字符串进行转义和取消转义，这是 URL 参数处理的常见做法。

例如，在默认配置下，如果你有以下搜索对象：

```tsx
const search = {
  page: 1,
  sort: "asc",
  filters: { author: "tanner", min_words: 800 },
};
```

它会被序列化并转义为以下查询字符串：

```txt
?page=1&sort=asc&filters=%7B%22author%22%3A%22tanner%22%2C%22min_words%22%3A800%7D
```

我们可以通过以下代码手动实现这种默认行为：

```tsx
import {
  createRouter,
  parseSearchWith,
  stringifySearchWith,
} from "@tanstack/react-router";

const router = createRouter({
  // ...
  parseSearch: parseSearchWith(JSON.parse),
  stringifySearch: stringifySearchWith(JSON.stringify),
});
```

然而，默认行为可能并不适用于所有场景。例如，你可能希望使用不同的序列化格式（如 Base64 编码），或者使用专门的序列化/反序列化库，如 [query-string](https://github.com/sindresorhus/query-string)、[JSURL2](https://github.com/wmertens/jsurl2) 或 [Zipson](https://jgranstrom.github.io/zipson/)。

你可以通过在 [`Router`](../api/router/RouterOptionsType.md#stringifysearch-method) 配置中为 `parseSearch` 和 `stringifySearch` 选项提供自定义函数来实现这一点。在操作时，可以利用 TanStack Router 内置的辅助函数 `parseSearchWith` 和 `stringifySearchWith` 来简化流程。

> [!TIP]
> 序列化和反序列化的一个重要方面是**幂等性**，即反序列化后能够获得完全相同的对象。如果处理不当，可能会丢失信息。例如，如果使用的库不支持嵌套对象，那么在反序列化查询字符串时，嵌套的对象结构可能会丢失。

以下是几种在 TanStack Router 中自定义搜索参数序列化的示例：

## 使用 Base64

为了在浏览器和 URL 预览工具（Unfurlers）之间获得最大的兼容性，对搜索参数进行 Base64 编码是很常见的做法。可以使用以下代码实现：

```tsx
import {
  Router,
  parseSearchWith,
  stringifySearchWith,
} from "@tanstack/react-router";

const router = createRouter({
  parseSearch: parseSearchWith((value) => JSON.parse(decodeFromBinary(value))),
  stringifySearch: stringifySearchWith((value) =>
    encodeToBinary(JSON.stringify(value)),
  ),
});

function decodeFromBinary(str: string): string {
  return decodeURIComponent(
    Array.prototype.map
      .call(atob(str), function (c) {
        return "%" + ("00" + c.charCodeAt(0).toString(16)).slice(-2);
      })
      .join(""),
  );
}

function encodeToBinary(str: string): string {
  return btoa(
    encodeURIComponent(str).replace(/%([0-9A-F]{2})/g, function (match, p1) {
      return String.fromCharCode(parseInt(p1, 16));
    }),
  );
}
```

> [⚠️ 为什么不直接使用原始的 atob/btoa？](安全的二进制编码解码)

在这种配置下，之前的对象转换后的查询字符串如下所示：

```txt
?page=1&sort=asc&filters=eyJhdXRob3IiOiJ0YW5uZXIiLCJtaW5fd29yZHMiOjgwMH0%3D
```

> [!WARNING]
> 如果你将用户输入直接序列化为 Base64，可能会与 URL 反序列化发生冲突，导致 URL 解析错误或被解析为错误的值。为避免此问题，应使用安全的二进制编码/解码方法（见下文）。

## 使用 query-string 库

[query-string](https://github.com/sindresorhus/query-string) 是一个非常流行的库，能够可靠地解析和格式化查询字符串。你可以用它自定义序列化格式：

```tsx
import { createRouter } from "@tanstack/react-router";
import qs from "query-string";

const router = createRouter({
  // ...
  stringifySearch: stringifySearchWith((value) =>
    qs.stringify(value, {
      // ...配置选项
    }),
  ),
  parseSearch: parseSearchWith((value) =>
    qs.parse(value, {
      // ...配置选项
    }),
  ),
});
```

在这种配置下，转换后的查询字符串如下所示：

```txt
?page=1&sort=asc&filters=author%3Dtanner%26min_words%3D800
```

## 使用 JSURL2 库

[JSURL2](https://github.com/wmertens/jsurl2) 是一个非标准库，它可以在保持可读性的同时压缩 URL。

```tsx
import {
  Router,
  parseSearchWith,
  stringifySearchWith,
} from "@tanstack/react-router";
import { parse, stringify } from "jsurl2";

const router = createRouter({
  // ...
  parseSearch: parseSearchWith(parse),
  stringifySearch: stringifySearchWith(stringify),
});
```

在这种配置下，转换后的查询字符串如下所示：

```txt
?page=1&sort=asc&filters=(author~tanner~min*_words~800)~
```

## 使用 Zipson 库

[Zipson](https://jgranstrom.github.io/zipson/) 是一个非常易用且高性能的 JSON 压缩库。要结合它来压缩搜索参数（这同样需要转义和 Base64 编码），可以使用以下代码：

```tsx
import {
  Router,
  parseSearchWith,
  stringifySearchWith,
} from "@tanstack/react-router";
import { stringify, parse } from "zipson";

const router = createRouter({
  parseSearch: parseSearchWith((value) => parse(decodeFromBinary(value))),
  stringifySearch: stringifySearchWith((value) =>
    encodeToBinary(stringify(value)),
  ),
});

// 使用下方的安全编码/解码函数...
```

在这种配置下，转换后的查询字符串如下所示：

```txt
?page=1&sort=asc&filters=JTdCJUMyJUE4YXV0aG9yJUMyJUE4JUMyJUE4dGFubmVyJUMyJUE4JUMyJUE4bWluX3dvcmRzJUMyJUE4JUMyJUEyQ3UlN0Q%3D
```

<hr>

## 安全的二进制编码/解码

在浏览器中，原生的 `atob` 和 `btoa` 函数无法保证能正确处理非 UTF-8 字符。我们建议使用以下工具函数进行编码/解码：

**将字符串编码为二进制字符串：**

```ts
export function encodeToBinary(str: string): string {
  return btoa(
    encodeURIComponent(str).replace(/%([0-9A-F]{2})/g, function (match, p1) {
      return String.fromCharCode(parseInt(p1, 16));
    }),
  );
}
```

**将二进制字符串解码为字符串：**

```ts
export function decodeFromBinary(str: string): string {
  return decodeURIComponent(
    Array.prototype.map
      .call(atob(str), function (c) {
        return "%" + ("00" + c.charCodeAt(0).toString(16)).slice(-2);
      })
      .join(""),
  );
}
```
