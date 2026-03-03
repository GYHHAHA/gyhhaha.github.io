---
short_title: 身份验证（概述）
---

# 身份验证 (Authentication)

## 身份验证 (Authentication) vs 授权 (Authorization)

- **身份验证 (Authentication)**：这个用户是谁？（登录/注销、身份核实）
- **授权 (Authorization)**：这个用户能做什么？（权限、角色、访问控制）

---

## 架构概览 (Architecture Overview)

### 全栈身份验证模型

**服务端 (安全层)**

- 会话存储与校验
- 用户凭证验证
- 数据库操作
- 令牌 (Token) 生成与验证
- 受保护的 API 端点

**客户端 (公开层)**

- 身份验证状态管理
- 路由保护逻辑
- 登录/注销用户界面
- 重定向处理

**同构层 (两者兼有)**

- 检查验证状态的路由加载器 (Loaders)
- 共享的校验逻辑
- 用户资料数据访问

---

### 会话管理模式 (Session Management Patterns)

**HTTP-Only Cookies (推荐)**

- 最安全的方法 —— JavaScript 无法访问。
- 浏览器自动处理。
- 通过 `sameSite` 属性内置 CSRF 防护。
- 传统 Web 应用程序的最佳选择。

**JWT 令牌 (JWT Tokens)**

- 无状态身份验证。
- 适用于 API 优先的应用。
- 需要小心处理以避免 XSS 漏洞。
- 应考虑使用刷新令牌 (Refresh Token) 轮换机制。

**服务端会话 (Server-Side Sessions)**

- 集中式会话控制。
- 易于撤销特定会话。
- 需要会话存储（数据库、Redis）。
- 适用于需要立即控制会话撤销的应用。

---

### 路由保护架构 (Route Protection Architecture)

**布局路由模式 (推荐)**

- 使用父级布局路由保护整个路由子树。
- 集中化的身份验证逻辑。
- 自动保护所有子路由。
- 清晰地分离已验证路由与公开路由。

**组件级保护**

- 组件内的条件渲染。
- 对 UI 状态更细粒度的控制。
- 适用于同一路由下混合显示公开/私有内容。
- 需要小心处理以防止布局抖动 (Layout Shifts)。

**服务器函数守卫 (Server Function Guards)**

- 在执行敏感操作前进行服务端验证。
- 与路由级保护协同工作。
- 无论客户端保护如何，这对于 API 安全都至关重要。

---

### 状态管理模式 (State Management Patterns)

**服务端驱动状态 (推荐)**

- 每次请求的验证状态均源自服务端。
- 始终与服务端状态保持同步。
- 与 SSR (服务器端渲染) 无缝配合。
- 安全性最高 —— 服务端是唯一的事实来源。

**基于上下文 (Context) 的状态**

- 客户端身份验证状态管理。
- 适用于第三方验证提供商（如 Auth0、Firebase）。
- 需要小心处理与服务端状态的同步。
- 适用于交互性极强的客户端优先应用。

**混合方案**

- 初始状态来自服务端，客户端进行后续更新。
- 在安全性和用户体验 (UX) 之间取得平衡。
- 定期进行服务端验证。

---

## 身份验证方案选择

### 🏢 合作伙伴解决方案

- **[WorkOS](https://workos.com)**
- **[Clerk](https://clerk.dev)**

### 🛠️ DIY 身份验证

使用 TanStack Start 的服务器函数和会话管理构建你自己的身份验证系统：

- **完全控制**：对身份验证流程进行深度自定义。
- **服务器函数**：在服务器上实现安全的验证逻辑。
- **会话管理**：内置支持 HTTP-only cookies 的会话处理。
- **类型安全**：为身份验证状态提供端到端的类型支持。

---

### 🌐 其他优秀选择

**开源及社区解决方案：**

- **[Better Auth](https://www.better-auth.com/)** —— 现代化的、TypeScript 优先的身份验证库。
- **[Auth.js](https://authjs.dev/)** (原 NextAuth.js) —— React 生态中广受欢迎的验证库。

**托管服务 (SaaS)：**

- **[Supabase Auth](https://supabase.com/auth)** —— 开源的 Firebase 替代方案，内置功能强大的验证模块。
- **[Auth0](https://auth0.com/)** —— 老牌身份验证平台，功能极其丰富。
- **[Firebase Auth](https://firebase.google.com/docs/auth)** —— Google 提供的身份验证服务。

---

## 合作伙伴解决方案 (Partner Solutions)

### WorkOS - 企业级身份验证

<a href="https://workos.com/" alt="WorkOS Logo">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/workos-white.svg" width="280">
    <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/workos-black.svg" width="280">
    <img alt="WorkOS logo" src="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/workos-black.svg" width="280">
  </picture>
</a>

- **单点登录 (SSO)** —— 集成 SAML、OIDC 和 OAuth。
- **目录同步 (Directory Sync)** —— 通过 SCIM 与 Active Directory 和 Google Workspace 同步。
- **多因素身份验证 (MFA)** —— 提供企业级的安全选项。
- **合规就绪** —— 符合 SOC 2、GDPR 和 CCPA 标准。

[访问 WorkOS →](https://workos.com/) | [查看示例 →](https://github.com/TanStack/router/tree/main/examples/react/start-workos)

---

### Clerk - 全能身份验证平台

<a href="https://go.clerk.com/wOwHtuJ" alt="Clerk Logo">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/clerk-logo-dark.svg" width="280">
    <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/clerk-logo-light.svg" width="280">
    <img alt="Clerk logo" src="https://raw.githubusercontent.com/tanstack/tanstack.com/main/src/images/clerk-logo-light.svg" width="280">
  </picture>
</a>

- **开箱即用的 UI 组件** —— 登录、注册、用户资料和组织管理。
- **社交登录** —— 支持 Google、GitHub、Discord 等 20 多个平台。
- **多因素身份验证** —— 支持 SMS、TOTP 和备份代码。
- **组织与团队** —— 为团队协作应用提供内置支持。

[访问 Clerk →](https://go.clerk.com/wOwHtuJ) | [免费注册 →](https://go.clerk.com/PrSDXti) | [查看示例 →](https://github.com/TanStack/router/tree/main/examples/react/start-clerk-basic)

---

## 架构决策指南

### 选择身份验证方案

**合作伙伴解决方案 (Partner Solutions)：**

- 优势：专注于核心业务逻辑、企业级特性（SSO、合规）、托管安全更新、预制 UI 组件。

**开源方案 (OSS Solutions)：**

- 优势：社区驱动、支持深度定制、可自行托管、避免供应商锁定。

**DIY 自建方案：**

- 优势：对验证流程的完全控制、满足特殊的安全要求、特定的业务逻辑需求、完全拥有验证数据。

---

### 安全注意事项

- **生产环境强制使用 HTTPS**。
- **尽可能使用 HTTP-only cookies**。
- **在服务端验证所有输入**。
- **将敏感密钥留在仅限服务端的函数中**。
- **为验证端点实现速率限制 (Rate Limiting)**。
- **为表单提交开启 CSRF 保护**。

---

## 后续步骤

- **寻求合作伙伴方案** → [Clerk](https://go.clerk.com/wOwHtuJ) 或 [WorkOS](https://workos.com/)
- **尝试 DIY 实现** → [身份验证指南](./authentication.md)
- **查看示例代码** → [运行中的实现案例](https://github.com/TanStack/router/tree/main/examples/react)
