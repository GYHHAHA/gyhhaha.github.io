---
short_title: IAM
---

# Identity and Access Management

## 基本概念

IAM
: IAM 是一个全球服务，默认创建的 Root 账号不应被日常使用或共享；User 代表组织中的个人，可以被加入到一个或多个 Group 中，而 Group 只能包含用户、不能嵌套其他组；用户不一定必须属于某个组，但可以同时属于多个组。

IAM Policy
: IAM Policy 是用 JSON 格式编写的权限控制文件，用来定义“谁可以对哪些资源执行哪些操作，以及在什么条件下执行”。一个完整的 Policy 由 Version（策略语言版本）、可选的 Id，以及一个或多个 Statement 组成；而每个 Statement 则包含权限的核心要素，如 Effect（允许或拒绝）、Principal（适用于谁，资源策略中使用）、Action（允许或拒绝的操作）、Resource（作用的资源），以及可选的 Condition（生效条件）。通过这些字段组合，IAM 可以实现精细化的访问控制。

```json
{
  // 策略语言版本（当前IAM固定使用 2012-10-17）
  "Version": "2012-10-17",
  // 策略的唯一标识符（可选）
  "Id": "S3-Account-Permissions",
  // 权限声明数组（必需），可以包含多条Statement
  "Statement": [
    {
      // 单条声明的标识（可选）
      "Sid": "1",
      // 权限效果：Allow（允许）或 Deny（拒绝）
      "Effect": "Allow",
      // 指定策略作用的主体（谁可以访问资源）
      // ⚠️ 仅资源策略（如S3 Bucket Policy）才需要Principal
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      // 定义允许或拒绝的具体操作
      "Action": ["s3:GetObject", "s3:PutObject"],
      // 指定操作作用的资源（ARN格式）
      // 这里表示 mybucket 桶下的所有对象
      "Resource": "arn:aws:s3:::mybucket/*"
    }
  ]
}
```

AssumeRole
: AssumeRole 是 AWS STS 提供的一种临时权限获取机制，指的是某个身份（用户、角色或外部身份）通过调用 sts:AssumeRole API，临时“切换”为另一个 IAM Role，从而获得该角色的临时安全凭证（Access Key、Secret Key、Session Token）。它的特点是：权限是临时的（默认 1 小时）、不需要共享长期凭证、常用于跨账号访问或权限隔离。例如，开发账号中的用户可以 Assume 生产账号中的一个只读 Role，在限定时间内访问生产环境的 S3 资源，而无需拥有生产账号的长期访问密钥。

SSO
: Single Sign-On 是一种统一身份认证系统，用于集中管理“人如何登录多个 AWS 账号”。它通过企业身份源（如 Azure AD、Okta 等）进行认证，用户登录一次后即可访问多个 AWS 账号或应用。其特点是集中管理用户身份、基于角色分配权限、无需为每个账号创建独立 IAM User。比如，公司员工通过企业邮箱登录 AWS SSO 门户，选择“生产只读角色”进入生产账号，背后系统实际上会自动帮用户执行一次 AssumeRole，并发放临时凭证。

IAM Roles
: IAM Roles for Services 指的是当某些 AWS 服务需要代表你去访问其他 AWS 资源时，通过分配 IAM Role 来授予它们所需权限的机制。也就是说，服务本身没有权限，必须通过绑定的 IAM Role 才能执行操作，例如 EC2 实例通过 Instance Role 访问 S3、Lambda 函数通过 Execution Role 访问 DynamoDB，或 CloudFormation 通过专用 Role 创建资源。这样可以避免在服务中存储长期凭证，并实现基于最小权限原则的安全访问控制。

IAM Security Tools
: IAM Security Tools 主要用于帮助你审计和优化权限配置：IAM Credentials Report（账号级）可以生成一份报告，列出账户中所有 IAM 用户及其凭证状态（如是否启用密码、是否启用 MFA、Access Key 是否轮换等），用于整体安全检查；而 IAM Access Advisor（用户级）则显示某个用户或角色被授予了哪些服务权限，以及这些服务最近一次被访问的时间，帮助你识别未使用的权限，从而收缩策略、实现最小权限原则。

AWS Organizations
: AWS Organizations 是一个用于集中管理多个 AWS 账号的全局服务，它通过一个管理账号（Management Account）统一管理多个成员账号（Member Accounts），每个账号只能隶属于一个组织。Organizations 支持集中计费（Consolidated Billing），即所有账号共享一个支付方式，并可基于整体使用量获得更好的价格折扣（如 EC2、S3 的用量折扣），同时支持在账号间共享 Reserved Instances 和 Savings Plans 折扣。除了计费整合外，Organizations 还常用于企业级权限治理（如通过 SCP 限制账号最大权限范围），是多账号架构和企业云治理的核心基础。

IAM Condition
: Condition 是 IAM Policy 里的“附加限制条件”，用于在 Allow / Deny 生效之前增加判断逻辑。它的作用是：让权限不仅取决于“谁对什么资源做什么操作”，还取决于“在什么情况下”。也就是说，Condition 让权限控制从“三维”（Principal / Action / Resource）变成“四维”（再加上 Context）。

| 运算符          | 用途                      | 示例                                                                              |
| --------------- | ------------------------- | --------------------------------------------------------------------------------- |
| StringEquals    | 字符串精确匹配            | `"StringEquals": { "aws:RequestedRegion": "us-east-1" }`                          |
| StringLike      | 字符串模糊匹配（支持 \*） | `"StringLike": { "s3:prefix": "home/*" }`                                         |
| IpAddress       | IP 地址匹配               | `"IpAddress": { "aws:SourceIp": "203.0.113.0/24" }`                               |
| NotIpAddress    | 排除指定 IP               | `"NotIpAddress": { "aws:SourceIp": "10.0.0.0/8" }`                                |
| Bool            | 布尔值判断                | `"Bool": { "aws:SecureTransport": true }`                                         |
| BoolIfExists    | 条件存在时才判断          | `"BoolIfExists": { "aws:MultiFactorAuthPresent": true }`                          |
| DateGreaterThan | 当前时间晚于某时间        | `"DateGreaterThan": { "aws:CurrentTime": "2024-01-01T00:00:00Z" }`                |
| DateLessThan    | 当前时间早于某时间        | `"DateLessThan": { "aws:CurrentTime": "2024-12-31T23:59:59Z" }`                   |
| NumericEquals   | 数值匹配                  | `"NumericEquals": { "s3:max-keys": 10 }`                                          |
| ArnEquals       | ARN 精确匹配              | `"ArnEquals": { "aws:PrincipalArn": "arn:aws:iam::123456789012:role/AdminRole" }` |

Permission Boundary
: Permission Boundary（权限边界）是一种高级 IAM 功能，用一个 Managed Policy 来定义某个 IAM User 或 Role 能拥有的“最大权限上限”。它本身不授予权限，而是限制身份最终能生效的权限范围。权限边界只支持 User 和 Role（不支持 Group）。

| 机制                | 控制层级   | 控制对象          |
| ------------------- | ---------- | ----------------- |
| Identity Policy     | 赋权       | User / Role       |
| Permission Boundary | 身份级上限 | 单个 User / Role  |
| SCP                 | 账号级上限 | 整个 Account / OU |

IAM Identity Center
: AWS IAM Identity Center（原 AWS Single Sign-On）是一个统一身份与访问管理服务，用于实现企业级的单点登录（SSO）。它允许用户通过一次登录访问多个 AWS 账号（基于 AWS Organizations）、各类业务云应用（如 Salesforce、Microsoft 365 等）、支持 SAML 2.0 的应用，以及 EC2 Windows 实例。Identity Center 可以使用内置身份存储，也可以集成第三方身份提供商（如 Active Directory、Okta、OneLogin 等），实现集中身份管理与权限分配。本质上，它解决的是“人如何统一登录并访问多个账号和应用”的问题，并在背后通过分配 IAM Role 来控制访问权限。

Microsoft Active Directory
: Microsoft Active Directory（AD）是运行在 Windows Server 上的企业级目录服务，本质上是一个集中管理身份和资源的数据库，用于存储和管理用户账户、计算机、组、打印机、文件共享等对象，并通过域控制器（Domain Controller）提供统一的身份认证与授权机制。它支持集中创建账号、分配权限、组织结构分层管理（树和域结构），是传统企业内部实现单点登录和统一安全管理的核心系统。

AWS Directory Services
: AWS Directory Services 是 AWS 提供的一组托管目录解决方案，用于在云上运行、连接或兼容 Microsoft Active Directory。

| 服务类型                 | 是否创建新 AD   | 是否可与本地 AD 建立信任         | 适用场景                 |
| ------------------------ | --------------- | -------------------------------- | ------------------------ |
| AWS Managed Microsoft AD | ✅ 是           | ✅ 支持双向信任                  | 企业级 AD 上云或混合架构 |
| AD Connector             | ❌ 否（仅代理） | ❌ 不建立信任（直接连接本地 AD） | 使用现有本地 AD 做认证   |
| Simple AD                | ✅ 是（轻量级） | ❌ 不支持信任                    | 小规模或测试环境         |

AWS Control Tower
: AWS Control Tower 是一个用于快速搭建和治理多账号 AWS 环境的管理服务，它基于 AWS Organizations，按照最佳实践自动创建和组织多个账号，并通过预设的“护栏（guardrails）”实现持续的安全与合规管控。它可以自动化环境初始化、策略管理和合规监控，帮助企业在多账号架构下统一管理访问控制、检测策略违规并提供可视化合规仪表盘，是企业级 AWS 环境落地和治理的自动化解决方案。

Guardrails
: AWS Control Tower 的 Guardrails（护栏）用于对多账号环境进行持续治理，分为两类：预防型（Preventive）和检测型（Detective）。预防型护栏基于 AWS Organizations 的 SCP，在账号层面直接限制不允许的操作（例如限制所有账号只能使用指定 Region）；检测型护栏基于 AWS Config，用于持续监控资源状态（例如检测未打标签的资源）。当检测到不合规（NON_COMPLIANT）时，可以通过 SNS 通知管理员，或触发 Lambda 自动修复（如自动补标签）。

## 疑难问题

Permission和Inline Policy区别是什么？
: Permission（权限）指的是某个 IAM 用户、组或角色最终拥有的访问能力，也就是它“能对哪些 AWS 资源执行哪些操作”。权限不是一个具体的对象，而是多个策略综合评估后的结果，可能来源于 Managed Policy、Inline Policy、Resource-based Policy、SCP 等。当我们说“这个角色有 S3 读权限”，说的就是它最终生效的 Permission。
: Inline Policy（内联策略）则是一种附加策略的方式，它是直接嵌入到某一个特定的用户、组或角色中的策略，不能被复用，也不能独立存在。删除该 IAM 实体时，Inline Policy 也会随之删除。它本身只是权限的来源之一，而不是最终的权限结果。

| 对比项               | Inline Policy | Managed Policy |
| -------------------- | ------------- | -------------- |
| 是否可复用           | ❌ 不可复用   | ✅ 可复用      |
| 是否可单独存在       | ❌ 不可以     | ✅ 可以        |
| 是否推荐在企业中使用 | ❌ 不推荐     | ✅ 推荐        |
| 是否容易集中管理     | ❌ 不容易     | ✅ 容易        |
| 适用场景             | 特殊定制权限  | 标准权限模板   |

IAM 各类策略对比表
: Identity Policy 和 Inline Policy 是授予权限的方式，Resource Policy 是从资源角度控制访问，而 SCP 和 Permission Boundary 不授予权限，只用于限制最大权限范围。最终权限是多种策略共同计算的结果，并遵循显式拒绝优先原则。

| 类型                | 定义位置                       | 作用范围层级       | 是否授予权限 | 是否限制权限上限 |
| ------------------- | ------------------------------ | ------------------ | ------------ | ---------------- |
| Identity Policy     | IAM 用户 / 组 / 角色上         | 身份级             | ✅ 是        | ❌ 否            |
| Inline Policy       | IAM 用户 / 组 / 角色内部       | 身份级             | ✅ 是        | ❌ 否            |
| Resource Policy     | 具体 AWS 资源上（如 S3、KMS）  | 资源级             | ✅ 是        | ❌ 否            |
| SCP                 | AWS Organizations（OU / 账号） | 账号级             | ❌ 否        | ✅ 是            |
| Permission Boundary | IAM 用户 / 角色上              | 身份级（上限控制） | ❌ 否        | ✅ 是            |

一张层级理解图

```text
Organizations
   └── SCP  (限制账号)

Account
   └── IAM
         ├── User / Role
         │       ├── Identity Policy
         │       ├── Inline Policy
         │       └── Permission Boundary
         │
         └── Resource
                 └── Resource Policy
```

AWS 用户访问方式有哪些？
: 列表如下：

| 访问方式               | 使用场景           | 认证方式                          | 是否需要 Access Key | 典型使用人群  |
| ---------------------- | ------------------ | --------------------------------- | ------------------- | ------------- |
| AWS Management Console | 浏览器登录操作 AWS | 用户名 + 密码 + MFA               | ❌ 不需要           | 运维 / 管理员 |
| AWS CLI                | 命令行操作 AWS     | Access Key ID + Secret Access Key | ✅ 需要             | 运维 / DevOps |
| AWS SDK                | 代码中调用 AWS API | Access Key 或 IAM Role            | ✅（或使用Role）    | 开发人员      |

生产环境（EC2）SDK 如何拥有正确权限？
: 给 EC2 实例绑定一个 IAM Role（Instance Profile），这个 Role 上挂最小权限的 Identity Policy（例如只允许访问指定 S3 bucket / DynamoDB 表）。应用代码里不要写任何 Access Key，AWS SDK 会通过 Instance Metadata Service（IMDS） 自动拿到该 Role 的 STS 临时凭证 并签名请求；最终能做什么由该 Role 的有效权限（再叠加资源策略、SCP、Boundary 等限制）决定。

本地开发如何与生产权限一致？
: 代码保持和生产一样（不硬编码凭证），本地用 AssumeRole 到同一个生产/等效的 Role 来获取临时凭证。做法是配置 AWS CLI 的 profile（~/.aws/config 里写 role_arn + source_profile 或用 SSO），运行时设置 AWS_PROFILE=appX；这样本地 SDK 使用的就是与 EC2 相同的 Role 权限，避免“本地能跑、上生产 AccessDenied”。

1. 使用SSO

当你执行 aws sso login --profile appA 时，实际上只是完成了身份认证：通过浏览器登录公司账号后，AWS 在本地缓存一个 SSO token，但此时还没有真正获得访问 AWS 资源的权限。随后当你运行 AWS_PROFILE=appA node app.js 时，SDK 会读取该 profile 的 SSO 配置，使用缓存的 SSO token 向 IAM Identity Center 请求对应的账号和角色权限，并自动执行一次 AssumeRole，获取 STS 临时凭证；之后所有 AWS API 调用都使用这组临时凭证完成签名和访问。也就是说，SSO 负责确认“你是谁”，而真正赋予访问能力的是后续自动完成的 AssumeRole。

```bash
# ~/.aws/config
[profile appA]
# ===== 这是 SSO 登录入口地址 =====
sso_start_url = https://company.awsapps.com/start
# ===== SSO 所在区域 =====
sso_region = us-east-1
# ===== 目标 AWS 账号 ID =====
sso_account_id = 123456789012
# ===== 要 Assume 的角色名称 =====
sso_role_name = AppA-Role
# ===== 默认区域 =====
region = us-east-1
# ===== 输出格式 =====
output = json
```

2. 使用临时凭证

当你运行 AWS_PROFILE=appA node app.js 时，SDK 会读取 appA 这个 profile 的配置，发现它指定了 role_arn 和 source_profile=default。于是 SDK 先使用 default profile 中的长期 Access Key 作为基础身份，调用 sts:AssumeRole(AppA-Role) 请求切换到目标角色；STS 返回一组临时凭证（Access Key、Secret Key、Session Token），随后应用使用这组临时凭证对所有 AWS API 请求进行签名和访问。因此，真正生效的权限来自被 Assume 的 Role，而不是 default 用户本身。

```bash
# ~/.aws/config
[profile appA]
# ===== 要切换的角色 ARN =====
role_arn = arn:aws:iam::123456789012:role/AppA-Role

# ===== 基础身份来源 =====
# 这里必须是一个已经有凭证的 profile
source_profile = default

region = us-east-1
output = json
```

```bash
# ~/.aws/credentials
[default]
aws_access_key_id = AKIAxxxxxxxx
aws_secret_access_key = xxxxxxxxxxxxx
```

多个不同应用如何切换权限？
: 为每个应用对应一个独立 Role（AppA-Role、AppB-Role…），本地为每个 Role 配一个 profile（如 [profile appA] role_arn=...、[profile appB] ...），切换时只需要切换 AWS_PROFILE（或启动命令前加 AWS_PROFILE=appA node app.js）。代码不变，权限随 profile/Role 切换，既安全又可复用。

IAM Role和Resource-based Policy有什么区别？
: 在跨账号访问场景中，可以通过两种方式授权：一种是使用 IAM Role 作为中介（代理），即账户 A 的用户先 Assume 账户 B 的 Role，再以该 Role 的权限访问目标资源；此时用户会“放弃”原有权限，完全使用 Role 的权限。另一种是使用 Resource-based Policy（资源策略），直接在资源（如 S3 Bucket）上写策略，允许账户 A 的用户访问；这种方式下用户不会切换身份，而是保留原有权限，再叠加资源允许。两者都能实现跨账号访问，但 Role 更适合复杂授权和最小权限控制，而资源策略更适合直接授权给特定主体。
: 当 EventBridge 规则触发时，如果目标服务支持资源策略（如 Lambda、SNS、SQS、S3、API Gateway），则需要在目标资源上添加 resource-based policy，允许 EventBridge 调用它；而如果目标服务不支持资源策略（如 Kinesis、EC2 Auto Scaling、ECS 等），则需要为 EventBridge 配置一个 IAM Role，让 EventBridge 通过该 Role 获得访问目标服务的权限。简单来说：支持资源策略的服务用 resource-based policy，不支持的就让 EventBridge Assume 一个 IAM Role 来执行操作。

IAM Policy的评估流程是什么？
: 见下：

```text
START
│
├─ 默认状态：Deny（默认拒绝）
│
├─ 1️⃣ 是否存在 Explicit Deny？
│     ├─ Yes → ❌ Final: DENY
│     └─ No  → 继续
│
├─ 2️⃣ 是否在 Organization 中且有 SCP？
│     ├─ Yes →
│     │     ├─ SCP 是否 Allow？
│     │     │     ├─ No → ❌ Final: DENY (implicit)
│     │     │     └─ Yes → 继续
│     └─ No → 继续
│
├─ 3️⃣ 资源是否有 Resource-based Policy？
│     ├─ Yes →
│     │     ├─ 是否 Allow？
│     │     │     ├─ No → ❌ Final: DENY (implicit)
│     │     │     └─ Yes → 继续
│     └─ No → 继续
│
├─ 4️⃣ 是否有 Identity-based Policy？
│     ├─ No → ❌ Final: DENY (implicit)
│     └─ Yes →
│           ├─ 是否 Allow？
│           │     ├─ No → ❌ Final: DENY (implicit)
│           │     └─ Yes → 继续
│
├─ 5️⃣ 是否有 Permission Boundary？
│     ├─ Yes →
│     │     ├─ Boundary 是否 Allow？
│     │     │     ├─ No → ❌ Final: DENY (implicit)
│     │     │     └─ Yes → 继续
│     └─ No → 继续
│
├─ 6️⃣ 是否为 Session Principal？
│     ├─ Yes →
│     │     ├─ 是否有 Session Policy？
│     │     │     ├─ Yes →
│     │     │     │     ├─ 是否 Allow？
│     │     │     │     │     ├─ No → ❌ Final: DENY (implicit)
│     │     │     │     │     └─ Yes → 继续
│     │     │     └─ No → 继续
│     └─ No → 继续
│
└─ ✅ Final: ALLOW
```

IAM Identity Center 如何实现跨账号分配和细粒度权限管理？
: 在 AWS Organizations 环境下，IAM Identity Center 通过 Permission Set 来实现跨账号权限管理。Permission Set 本质上是一组 IAM Policy 的逻辑集合，用来定义某类用户在某个账号中应具备的权限。当你把一个 Permission Set 分配给用户或用户组，并指定目标 AWS 账号后，Identity Center 会在该账号中自动创建并管理一个对应的 IAM Role。用户登录后无需手动切换配置，系统会自动让其 Assume 这个 Role，从而获得在目标账号中的访问权限。简单来说，Permission Set 是权限模板，而真正生效的是登录时在目标账号中生成并被 Assume 的 IAM Role。
: ABAC（Attribute-Based Access Control，基于属性的访问控制）是一种通过“属性”来动态控制访问权限的机制，而不是逐个用户手动分配权限。权限判断基于用户的属性（如部门、成本中心、职位、地区等）与资源的标签进行匹配，例如当用户的属性为 Department=Data 时，系统会自动允许其访问带有 Department=Data 标签的资源。这样，当用户属性发生变化（比如调岗到其他部门）时，无需修改策略或重新授权，只需更新属性，权限便会自动随之调整。核心思想是用“属性驱动权限”，实现更灵活、可扩展的细粒度访问控制。

IAM Identity Center 如何与 AD 集成？
: IAM Identity Center 可以将 Active Directory 作为身份来源使用，实现企业统一登录和跨账号访问控制。它可以直接连接 AWS Managed Microsoft AD（开箱即用），也可以通过建立信任关系或使用 AD Connector 接入自建本地 AD。一旦集成完成，Identity Center 就可以使用 AD 中的用户和组来分配 AWS 账号权限或 SaaS 应用访问权限，用户登录后会通过对应的 IAM Role 获得授权，从而实现基于企业现有身份体系的统一认证与访问管理。

```text
AD（存员工账号）
     ↓
IAM Identity Center（用 AD 做身份源）
     ↓
SSO 登录
     ↓
自动 AssumeRole
     ↓
CLI 获得临时凭证
```
