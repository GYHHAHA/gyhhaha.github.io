---
short_title: 可观测性
---

# 可观测性 (Observability)

可观测性是现代 Web 开发的一个关键方面，它使你能够监控、追踪和调试应用程序的性能与错误。TanStack Start 提供了内置的可观测性模式，并能与外部工具无缝集成，为你提供关于应用程序的全方位洞察。

## 合作伙伴解决方案：Sentry

为了获得全面的可观测性，我们推荐使用 [Sentry](https://sentry.io?utm_source=tanstack) —— 我们在错误追踪和性能监控领域值得信赖的合作伙伴。Sentry 提供：

- **实时错误追踪**：捕获并调试整个技术栈中的错误。
- **性能监控**：追踪缓慢的事务并优化性能瓶颈。
- **版本健康度**：监控部署情况并追踪随时间变化的错误率。
- **用户影响分析**：了解错误是如何影响你的用户的。
- **TanStack Start 集成**：与服务器函数（Server Functions）和客户端代码无缝协作。

**快速设置：**

```tsx
// 客户端 (app.tsx)
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.NODE_ENV,
});

// 服务器函数
import * as Sentry from "@sentry/node";

const serverFn = createServerFn().handler(async () => {
  try {
    return await riskyOperation();
  } catch (error) {
    Sentry.captureException(error);
    throw error;
  }
});
```

[开始使用 Sentry →](https://sentry.io/signup?utm_source=tanstack) | [查看集成示例 →](https://github.com/TanStack/router/tree/main/e2e/react-router/sentry-integration)

## 内置可观测性模式

TanStack Start 的架构提供了多种无需外部依赖即可实现的内置可观测性机会：

### 服务器函数日志记录 (Server Function Logging)

在你的服务器函数中添加日志，以追踪执行情况、性能和错误：

```tsx
import { createServerFn } from "@tanstack/react-start";

const getUser = createServerFn({ method: "GET" })
  .inputValidator((id: string) => id)
  .handler(async ({ data: id }) => {
    const startTime = Date.now();

    try {
      console.log(`[SERVER] 正在获取用户 ${id}`);

      const user = await db.users.findUnique({ where: { id } });

      if (!user) {
        console.log(`[SERVER] 未找到用户 ${id}`);
        throw new Error("用户未找到");
      }

      const duration = Date.now() - startTime;
      console.log(`[SERVER] 用户 ${id} 获取成功，耗时 ${duration}ms`);

      return user;
    } catch (error) {
      const duration = Date.now() - startTime;
      console.error(`[SERVER] 获取用户 ${id} 出错，耗时 ${duration}ms:`, error);
      throw error;
    }
  });
```

### 请求/响应中间件 (Request/Response Middleware)

创建中间件来记录所有的请求和响应：

```tsx
import { createMiddleware } from "@tanstack/react-start";

const requestLogger = createMiddleware().server(async ({ request, next }) => {
  const startTime = Date.now();
  const timestamp = new Date().toISOString();

  console.log(`[${timestamp}] ${request.method} ${request.url} - 开始执行`);

  try {
    const result = await next();
    const duration = Date.now() - startTime;

    console.log(
      `[${timestamp}] ${request.method} ${request.url} - ${result.response.status} (${duration}ms)`,
    );

    return result;
  } catch (error) {
    const duration = Date.now() - startTime;
    console.error(
      `[${timestamp}] ${request.method} ${request.url} - 错误 (${duration}ms):`,
      error,
    );
    throw error;
  }
});

// 应用于所有服务器路由
export const Route = createFileRoute("/api/users")({
  server: {
    middleware: [requestLogger],
    handlers: {
      GET: async () => {
        return Response.json({ users: await getUsers() });
      },
    },
  },
});
```

### 路由性能监控 (Route Performance Monitoring)

在客户端和服务器端追踪路由加载性能：

```tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/dashboard")({
  loader: async ({ context }) => {
    const startTime = Date.now();

    try {
      const data = await loadDashboardData();
      const duration = Date.now() - startTime;

      // 记录服务器端 (SSR) 性能
      if (typeof window === "undefined") {
        console.log(`[SSR] Dashboard 加载耗时 ${duration}ms`);
      }

      return data;
    } catch (error) {
      const duration = Date.now() - startTime;
      console.error(`[LOADER] Dashboard 错误，耗时 ${duration}ms:`, error);
      throw error;
    }
  },
  component: Dashboard,
});

function Dashboard() {
  const data = Route.useLoaderData();

  // 追踪客户端渲染时间
  React.useEffect(() => {
    const renderTime = performance.now();
    console.log(`[CLIENT] Dashboard 渲染耗时 ${renderTime}ms`);
  }, []);

  return <div>Dashboard 内容</div>;
}
```

### 健康检查端点 (Health Check Endpoints)

创建用于健康监测的服务器路由：

```tsx
// routes/health.ts
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/health")({
  server: {
    handlers: {
      GET: async () => {
        const checks = {
          status: "healthy",
          timestamp: new Date().toISOString(),
          uptime: process.uptime(), // 运行时间
          memory: process.memoryUsage(), // 内存占用
          database: await checkDatabase(), // 数据库检查
          version: process.env.npm_package_version, // 版本号
        };

        return Response.json(checks);
      },
    },
  },
});

async function checkDatabase() {
  try {
    await db.raw("SELECT 1");
    return { status: "connected", latency: 0 };
  } catch (error) {
    return { status: "error", error: error.message };
  }
}
```

### 错误边界 (Error Boundaries)

实现全方位的错误处理：

```tsx
// 客户端错误边界
import { ErrorBoundary } from "react-error-boundary";

function ErrorFallback({ error, resetErrorBoundary }: any) {
  // 记录客户端错误
  console.error("[客户端错误]:", error);

  // 也可以发送到外部服务
  // sendErrorToService(error)

  return (
    <div role="alert">
      <h2>抱歉，出错了</h2>
      <button onClick={resetErrorBoundary}>重试</button>
    </div>
  );
}

export function App() {
  return (
    <ErrorBoundary FallbackComponent={ErrorFallback}>
      <Router />
    </ErrorBoundary>
  );
}

// 服务器函数错误处理
const riskyOperation = createServerFn().handler(async () => {
  try {
    return await performOperation();
  } catch (error) {
    // 记录带有上下文的服务器错误
    console.error("[服务器错误]:", {
      error: error.message,
      stack: error.stack,
      timestamp: new Date().toISOString(),
      // 如果可用，添加请求上下文
    });

    // 返回用户友好的错误信息
    throw new Error("操作失败，请重试。");
  }
});
```

### 性能指标收集 (Performance Metrics Collection)

收集并公开基础性能指标：

```tsx
// utils/metrics.ts
class MetricsCollector {
  private metrics = new Map<string, number[]>();

  // 记录耗时
  recordTiming(name: string, duration: number) {
    if (!this.metrics.has(name)) {
      this.metrics.set(name, []);
    }
    this.metrics.get(name)!.push(duration);
  }

  // 获取统计信息
  getStats(name: string) {
    const timings = this.metrics.get(name) || [];
    if (timings.length === 0) return null;

    const sorted = timings.sort((a, b) => a - b);
    return {
      count: timings.length,
      avg: timings.reduce((a, b) => a + b, 0) / timings.length, // 平均值
      p50: sorted[Math.floor(sorted.length * 0.5)], // 中位数
      p95: sorted[Math.floor(sorted.length * 0.95)], // 95分位数
      min: sorted[0],
      max: sorted[sorted.length - 1],
    };
  }

  getAllStats() {
    const stats: Record<string, any> = {};
    for (const [name] of this.metrics) {
      stats[name] = this.getStats(name);
    }
    return stats;
  }
}

export const metrics = new MetricsCollector();

// 指标端点
// routes/metrics.ts
export const Route = createFileRoute("/metrics")({
  server: {
    handlers: {
      GET: async () => {
        return Response.json({
          system: {
            uptime: process.uptime(),
            memory: process.memoryUsage(),
            timestamp: new Date().toISOString(),
          },
          application: metrics.getAllStats(),
        });
      },
    },
  },
});
```

### 开发环境调试响应头 (Debug Headers)

在响应中添加有助于开发的调试信息：

```tsx
import { createMiddleware } from "@tanstack/react-start";

const debugMiddleware = createMiddleware().server(async ({ next }) => {
  const result = await next();

  if (process.env.NODE_ENV === "development") {
    result.response.headers.set("X-Debug-Timestamp", new Date().toISOString());
    result.response.headers.set("X-Debug-Node-Version", process.version);
    result.response.headers.set("X-Debug-Uptime", process.uptime().toString());
  }

  return result;
});
```

### 环境特定日志记录 (Environment-Specific Logging)

为开发环境和生产环境配置不同的日志策略：

```tsx
// utils/logger.ts
import { createIsomorphicFn } from "@tanstack/react-start";

type LogLevel = "debug" | "info" | "warn" | "error";

const logger = createIsomorphicFn()
  .server((level: LogLevel, message: string, data?: any) => {
    const timestamp = new Date().toISOString();

    if (process.env.NODE_ENV === "development") {
      // 开发环境：详细的控制台日志
      console[level](`[${timestamp}] [${level.toUpperCase()}]`, message, data);
    } else {
      // 生产环境：结构化的 JSON 日志
      console.log(
        JSON.stringify({
          timestamp,
          level,
          message,
          data,
          service: "tanstack-start",
          environment: process.env.NODE_ENV,
        }),
      );
    }
  })
  .client((level: LogLevel, message: string, data?: any) => {
    if (process.env.NODE_ENV === "development") {
      console[level](`[CLIENT] [${level.toUpperCase()}]`, message, data);
    } else {
      // 生产环境：发送到分析服务
      // analytics.track('client_log', { level, message, data })
    }
  });

// 在应用中任何地方使用
export { logger };

// 使用示例
const fetchUserData = createServerFn().handler(async ({ data: userId }) => {
  logger("info", "开始获取用户数据", { userId });

  try {
    const user = await db.users.findUnique({ where: { id: userId } });
    logger("info", "用户数据获取成功", { userId });
    return user;
  } catch (error) {
    logger("error", "获取用户数据失败", {
      userId,
      error: error.message,
    });
    throw error;
  }
});
```

### 简单错误报告 (Simple Error Reporting)

无需外部依赖的基础错误报告机制：

```tsx
// utils/error-reporter.ts
const errorStore = new Map<
  string,
  { count: number; lastSeen: Date; error: any }
>();

export function reportError(error: Error, context?: any) {
  const key = `${error.name}:${error.message}`;
  const existing = errorStore.get(key);

  if (existing) {
    existing.count++;
    existing.lastSeen = new Date();
  } else {
    errorStore.set(key, {
      count: 1,
      lastSeen: new Date(),
      error: {
        name: error.name,
        message: error.message,
        stack: error.stack,
        context,
      },
    });
  }

  // 立即记录到控制台
  console.error("[已报告错误]:", {
    error: error.message,
    count: existing ? existing.count : 1,
    context,
  });
}

// 错误报告端点
// routes/errors.ts
export const Route = createFileRoute("/admin/errors")({
  server: {
    handlers: {
      GET: async () => {
        const errors = Array.from(errorStore.entries()).map(([key, data]) => ({
          id: key,
          ...data,
        }));

        return Response.json({ errors });
      },
    },
  },
});
```

## 外部可观测性工具 (External Observability Tools)

虽然 TanStack Start 提供了内置的可观测性模式，但外部工具可以提供更全面的监控：

### 其他流行工具

**应用性能监控 (APM):**

- **[DataDog](https://www.datadoghq.com/)** - 具备 APM 的全栈监控
- **[New Relic](https://newrelic.com/)** - 性能监控与告警
- **[Honeycomb](https://honeycomb.io/)** - 针对复杂系统的可观测性方案

**错误追踪:**

- **[Bugsnag](https://bugsnag.com/)** - 带有部署追踪的错误监控
- **[Rollbar](https://rollbar.com/)** - 实时错误告警

**分析与用户行为:**

- **[PostHog](https://posthog.com/)** - 带有错误追踪的产品分析
- **[Mixpanel](https://mixpanel.com/)** - 事件追踪与用户分析

### New Relic 集成

[New Relic](https://newrelic.com/) 是一款流行的应用性能监控工具。以下是如何将其与 TanStack Start 集成的方法。

#### 服务端渲染 (SSR)

要为服务端渲染启用 New Relic，你需要执行以下操作：

在 New Relic 上创建一个类型为 `Node` 的新集成。你将获得一个许可密钥 (license key)，我们将在下方使用它。

```js
// newrelic.js - New Relic 代理配置
exports.config = {
  app_name: ["YourTanStackApp"], // 你在 New Relic 中的应用名称
  license_key: "YOUR_NEW_RELIC_LICENSE_KEY", // 你的 New Relic 许可密钥
  agent_enabled: true,
  distributed_tracing: { enabled: true },
  span_events: { enabled: true },
  transaction_events: { enabled: true },
  // 其他默认设置
};
```

```tsx
// server.tsx
import newrelic from "newrelic"; // 确保这是第一个导入项
import {
  createStartHandler,
  defaultStreamHandler,
  defineHandlerCallback,
} from "@tanstack/react-start/server";
import type { ServerEntry } from "@tanstack/react-start/server-entry";

const customHandler = defineHandlerCallback(async (ctx) => {
  // 我们这样做是为了将事务按路由 ID 分组，而不是按唯一的 URL 分组
  const matches = ctx.router?.state?.matches ?? [];
  const leaf = matches[matches.length - 1];
  const routeId = leaf?.routeId ?? new URL(ctx.request.url).pathname;

  newrelic.setControllerName(routeId, ctx.request.method ?? "GET");
  newrelic.addCustomAttributes({
    "route.id": routeId,
    "http.method": ctx.request.method,
    "http.path": new URL(ctx.request.url).pathname,
    // 你想添加的任何其他自定义属性
  });

  return defaultStreamHandler(ctx);
});

export default {
  fetch(request) {
    const handler = createStartHandler(customHandler);
    return handler(request);
  },
} satisfies ServerEntry;
```

```bash
# 启动时预加载 New Relic 代理
node -r newrelic .output/server/index.mjs
```

#### 服务器函数与服务器路由

如果你想为服务器函数和服务器路由添加监控，需要按照上述步骤操作，然后添加以下内容：

```ts
// newrelic-middleware.ts
import newrelic from "newrelic";
import { createMiddleware } from "@tanstack/react-start";

export const nrTransactionMiddleware = createMiddleware().server(
  async ({ request, next }) => {
    const reqPath = new URL(request.url).pathname;
    newrelic.setControllerName(reqPath, request.method ?? "GET");
    return await next();
  },
);
```

```ts
// start.ts
import { createStart } from "@tanstack/react-start";
import { nrTransactionMiddleware } from "./newrelic-middleware";

export const startInstance = createStart(() => {
  return {
    requestMiddleware: [nrTransactionMiddleware],
  };
});
```

#### SPA 与浏览器 (SPA & Browser)

在 New Relic 上创建一个类型为 `React` 的新集成。

设置完成后，你需要将 New Relic 提供的集成脚本添加到你的根路由（root route）中。

```tsx
// __root.tsx
export const Route = createRootRoute({
  head: () => ({
    scripts: [
      {
        id: "new-relic",

        // 方式一：直接在此处复制并粘贴你的 New Relic 集成脚本内容
        children: `...`,

        // 方式二：也可以将其创建在 public 文件夹中（例如 /newrelic.js），然后在此引用
        src: "/newrelic.js",
      },
    ],
  }),
});
```

### OpenTelemetry 集成 (实验性)

[OpenTelemetry](https://opentelemetry.io/) 是可观测性的行业标准。以下是将其与 TanStack Start 集成的实验性方法：

```tsx
// instrumentation.ts - 在应用启动前初始化
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { Resource } from "@opentelemetry/resources";
import { SemanticResourceAttributes } from "@opentelemetry/semantic-conventions";

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: "tanstack-start-app",
    [SemanticResourceAttributes.SERVICE_VERSION]: "1.0.0",
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

// 务必在导入应用其他部分之前初始化
sdk.start();
```

```tsx
// 服务器函数追踪 (Tracing)
import { trace, SpanStatusCode } from "@opentelemetry/api";

const tracer = trace.getTracer("tanstack-start");

const getUserWithTracing = createServerFn({ method: "GET" })
  .inputValidator((id: string) => id)
  .handler(async ({ data: id }) => {
    return tracer.startActiveSpan("get-user", async (span) => {
      span.setAttributes({
        "user.id": id,
        operation: "database.query",
      });

      try {
        const user = await db.users.findUnique({ where: { id } });
        span.setStatus({ code: SpanStatusCode.OK });
        return user;
      } catch (error) {
        span.recordException(error);
        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: error.message,
        });
        throw error;
      } finally {
        span.end();
      }
    });
  });
```

```tsx
// 用于自动追踪的中间件
import { createMiddleware } from "@tanstack/react-start";
import { trace, SpanStatusCode } from "@opentelemetry/api";

const tracer = trace.getTracer("tanstack-start");

const tracingMiddleware = createMiddleware().server(
  async ({ next, request }) => {
    const url = new URL(request.url);

    return tracer.startActiveSpan(
      `${request.method} ${url.pathname}`,
      async (span) => {
        span.setAttributes({
          "http.method": request.method,
          "http.url": request.url,
          "http.route": url.pathname,
        });

        try {
          const result = await next();
          span.setAttribute("http.status_code", result.response.status);
          span.setStatus({ code: SpanStatusCode.OK });
          return result;
        } catch (error) {
          span.recordException(error);
          span.setStatus({
            code: SpanStatusCode.ERROR,
            message: error.message,
          });
          throw error;
        } finally {
          span.end();
        }
      },
    );
  },
);
```

> **注意**：上述 OpenTelemetry 集成仍处于实验阶段，需要手动设置。我们正在探索原生支持（first-class support），未来将为服务器函数、中间件和路由加载器（Loaders）提供自动插桩功能。

### 快速集成模式

大多数可观测性工具与 TanStack Start 的集成模式都非常相似：

```tsx
// 在应用入口点进行初始化
import { initObservabilityTool } from "your-tool";

initObservabilityTool({
  dsn: import.meta.env.VITE_TOOL_DSN,
  environment: import.meta.env.NODE_ENV,
});

// 服务器函数中间件
const observabilityMiddleware = createMiddleware().handler(async ({ next }) => {
  return yourTool.withTracing("server-function", async () => {
    try {
      return await next();
    } catch (error) {
      yourTool.captureException(error);
      throw error;
    }
  });
});
```

## 最佳实践

### 开发环境 vs 生产环境

```tsx
// 不同环境下的不同策略
const observabilityConfig = {
  development: {
    logLevel: "debug",
    enableTracing: true,
    enableMetrics: false, // 开发环境下指标数据可能干扰视线
  },
  production: {
    logLevel: "warn",
    enableTracing: true,
    enableMetrics: true,
    enableAlerting: true, // 启用告警
  },
};
```

### 性能监控检查清单 (Checklist)

- [ ] **服务器函数性能**：追踪执行时间
- [ ] **路由加载时间**：监控 Loader 的性能
- [ ] **数据库查询性能**：记录慢查询
- [ ] **外部 API 延迟**：监控第三方服务调用
- [ ] **内存占用**：追踪内存消耗模式
- [ ] **错误率**：监控错误频率及类型

### 安全注意事项

- 永远不要记录敏感数据（密码、令牌、个人隐私信息 PII）。
- 使用结构化日志（Structured Logging）以便更好地解析。
- 在生产环境中实施日志轮转（Log Rotation）。
- 考虑合规性要求（如 GDPR、CCPA）。

## 未来的 OpenTelemetry 支持

TanStack Start 即将迎来直接的 OpenTelemetry 支持，届时无需上述手动设置，即可为服务器函数、中间件和路由加载器提供自动插桩。

## 相关资源

- **[Sentry 官方文档](https://docs.sentry.io/)**
- **[OpenTelemetry 官方文档](https://opentelemetry.io/docs/)** —— 行业标准的可观测性方案
- **[运行示例](https://github.com/TanStack/router/tree/main/examples/react/start-basic)** —— 了解实际运行中的可观测性模式
