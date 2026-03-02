---
short_title: 数据库
---

# 数据库 (Databases)

数据库是任何动态应用程序的核心，为存储、检索和管理数据提供必要的基础设施。TanStack Start 使集成各种数据库变得非常简单，为你管理应用的数据层提供了一种灵活的方法。

## 我该使用什么？

TanStack Start **旨在与任何数据库提供商协同工作**。如果你已经有了偏好的数据库系统，你可以利用 Start 提供的全栈 API 将其集成。无论是 SQL、NoSQL 还是其他类型的数据库，TanStack Start 都能满足你的需求。

---

## 在 TanStack Start 中使用数据库有多简单？

在 TanStack Start 中使用数据库，就像从服务器函数或服务器路由中调用数据库的适配器、客户端、驱动或服务一样简单。

以下是一个关于如何连接数据库并进行读写操作的抽象示例：

```tsx
import { createServerFn } from "@tanstack/react-start";

// 假设这是你的数据库客户端
const db = createMyDatabaseClient();

export const getUser = createServerFn().handler(async ({ context }) => {
  // 在服务器函数中直接调用数据库
  const user = await db.getUser(context.userId);
  return user;
});

export const createUser = createServerFn({ method: "POST" }).handler(
  async ({ data }) => {
    const user = await db.createUser(data);
    return user;
  },
);
```

虽然这个例子很简略，但它证明了：只要你能从服务器端调用它，你就可以在 TanStack Start 中使用**任何**数据库提供商。

---

## 推荐的数据库提供商

虽然 TanStack Start 支持所有数据库，但我们强烈建议考虑我们的深度合作伙伴：[Neon](https://neon.tech?utm_source=tanstack) 或 [Convex](https://convex.dev?utm_source=tanstack)。它们都经过了 TanStack 的审核，在质量、开放性和性能标准上非常契合，是满足你数据库需求的绝佳选择。

### 什么是 Neon？

<a href="https://neon.tech?utm_source=tanstack" alt="Neon Logo">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/neon-dark.svg" width="280">
    <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/neon-light.svg" width="280">
    <img alt="Neon logo" src="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/neon-light.svg" width="280">
  </picture>
</a>

Neon 是一个全托管的无服务器 PostgreSQL，提供丰厚的免费额度。它实现了存储与计算的分离，支持自动扩缩容、分支功能（Branching）和无限存储。

**Neon 的核心特性：**

- **自动扩缩容**：无服务器架构，根据负载自动调整。
- **数据库分支**：像 Git 一样为开发和测试创建数据库分支。
- **内置连接池**：无需担心连接数限制。
- **无限存储**：不再为磁盘空间发愁。

- 了解更多请访问 [Neon 官网](https://neon.tech?utm_source=tanstack)
- 立即注册：[Neon 控制台](https://console.neon.tech/signup?utm_source=tanstack)

---

### 什么是 Convex？

<a href="https://convex.dev?utm_source=tanstack" alt="Convex Logo">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/convex-white.svg" width="280">
    <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/convex-color.svg" width="280">
    <img alt="Convex logo" src="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/convex-color.svg" width="280">
  </picture>
</a>

Convex 是一个强大的无服务器数据库平台，简化了数据管理流程。它提供实时、可扩展且支持事务的数据后端，与 TanStack Start 配合极佳。

**Convex 的优势：**

- **实时性**：内置数据推送，轻松实现实时应用。
- **声明式数据模型**：让后端开发变得更简单。
- **无需管理服务器**：专注于业务逻辑，而非运维。

- 了解更多请访问 [Convex 官网](https://convex.dev?utm_source=tanstack)
- 立即注册：[Convex 控制台](https://dashboard.convex.dev/signup?utm_source=tanstack)

---

### 什么是 Prisma Postgres？

<a href="https://www.prisma.io?utm_source=tanstack&via=tanstack" alt="Prisma Logo">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/prisma-dark.svg" width="280">
    <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/prisma-light.svg" width="280">
    <img alt="Prisma logo" src="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/prisma-light.svg" width="280">
  </picture>
</a>

即时可用的 Postgres，零配置：几秒钟内获得生产级别的 Postgres 数据库。它负责连接、缩放和性能调优。

**核心亮点：**

- **边缘优化**：本地路由意味着更低的延迟。
- **自动扩缩容**：从零用户平滑增长到数百万用户。
- **Unikernel 隔离**：每个数据库作为独立的 unikernel 运行，安全且高效。

- 了解更多请访问 [Prisma 官网](https://www.prisma.io?utm_source=tanstack&via=tanstack)
- 立即注册：[Prisma 控制面板](https://console.prisma.io/sign-up?utm_source=tanstack&via=tanstack)

---

## 文档与 API

关于集成不同数据库的详细指南即将推出！在此期间，请关注我们的示例代码，了解如何充分利用你的数据层。
