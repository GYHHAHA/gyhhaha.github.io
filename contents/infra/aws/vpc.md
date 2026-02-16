---
short_title: VPC
---

# Virtual Private Cloud

## 基本概念

VPC
: VPC（Virtual Private Cloud）是一种在公有云中创建的 逻辑隔离的私有网络环境。你在云上拥有一块“专属网络”，可以自定义 IP 地址段（CIDR），可以划分子网（公有/私有），可配置路由表、网关、安全组、ACL等，与其他用户网络完全隔离。

默认VPC
: 每个 Region 会自动创建一个默认 VPC，它已经配置好子网、Internet Gateway、路由表和安全组，实例默认可以获取公网 IP，可以直接访问互联网，适合测试或简单业务场景使用。

CIDR
: CIDR（Classless Inter-Domain Routing）是一种表示 IP 地址范围的方法，用“IP地址/前缀长度”表示网络大小。例如 192.168.1.0/24 中，“/24”表示前 24 位是网络位，剩余 8 位是主机位，因此该网段共有 256 个 IP 地址。前缀长度越小，可用地址范围越大；前缀长度越大，网段越小。

预留IP
: 每个子网自动预留 5 个 IP 地址，不能分配给实例。以 10.0.0.0/24 为例，这 5 个地址分别是：第 1 个是网络地址（10.0.0.0），第 2 个是 VPC 路由器地址（10.0.0.1），第 3 个是 AWS 预留的 DNS 服务器地址（10.0.0.2），第 4 个是 AWS 未来保留地址（10.0.0.3），最后 1 个是广播地址（10.0.0.255）。

NAT Gateway
: NAT Gateway（网络地址转换网关） 用于让 Private Subnet 中只有私有 IP 的实例访问互联网，但不允许互联网主动访问这些实例，同时它自己需要通过 IGW 才能看到互联网。它部署在 Public Subnet 中，必须绑定弹性公网 IP，并通过路由表让私有子网的 0.0.0.0/0 指向该 NAT Gateway。

ENI
: Elastic Network Interface是云服务器上的虚拟网卡，用于在 VPC 中为实例提供网络连接。它属于某个子网并获得该子网内的私有 IP；可以绑定一个或多个私有 IP，也可以绑定弹性公网 IP；默认每个实例有一个主 ENI（不可分离），也可以额外挂载多个 ENI（取决于实例类型）；ENI 可以在同一可用区内在实例之间移动，从而实现故障切换；安全组是绑定在 ENI 上而不是直接绑定在实例上的。

Bastion Host
: Bastion Host（堡垒机） 是部署在 Public Subnet 中、用于安全访问 Private Subnet 实例的跳板服务器。它通常有公网 IP，允许管理员通过 SSH（或 RDP）连接，然后再从该主机跳转访问私有子网中的实例；私有实例本身不暴露公网，从而降低攻击面。

VPC Peering
: VPC Peering 是一种把两个 VPC 直接私网互通的连接方式。建立 Peering 后，两个 VPC 可以通过私有 IP 通信，就像在同一个网络中一样，但它们仍然是独立的 VPC。特点是：需要双方接受连接请求；必须在双方路由表中添加指向对方 CIDR 的路由；不支持传递路由（不支持 transitive，A 连 B，B 连 C，不代表 A 能连 C）；可以同 Region 或跨 Region。

VPC Endpoint
: VPC Endpoint 是一种让 VPC 内资源通过私网访问 AWS 服务的方式，不需要经过 Internet Gateway、NAT Gateway 或公网。常见类型有两种：Gateway Endpoint（用于 S3、DynamoDB，通过路由表生效，并且免费）和 Interface Endpoint（基于 ENI，在子网内创建私有 IP，通过 PrivateLink 访问大多数 AWS 服务）。推荐仅在涉及跨网络访问时，用 Interface Endpoint。

VPC Flow Logs
: 用于记录 VPC 网络流量元数据的功能。它可以记录经过 ENI、子网或整个 VPC 的流量信息，包括源 IP、目标 IP、端口、协议、允许/拒绝状态等，但不会记录具体数据内容；日志可发送到 CloudWatch Logs 或 S3，用于排查网络问题和安全审计。

Virtual Private Gateway
: 挂载在 VPC 上的虚拟网关，用于连接本地数据中心到 AWS。它通常与 Site-to-Site VPN 或 Direct Connect 一起使用，作为 VPC 侧的终端；本地侧通过 Customer Gateway 连接到 VGW，从而实现私网互通。

Route Propagation
: 路由传播指的是当 VPC 连接了 VGW 或 Direct Connect 后，来自本地网络的路由会自动注入到 VPC 的路由表中，无需手动添加静态路由。前提是你在对应的路由表上启用了 route propagation；启用后，本地通过 BGP（边界网关协议）宣告的网段会自动出现在路由表里。

Transit Gateway
: 一个中心化网络枢纽，用来连接多个 VPC、本地数据中心（VPN / Direct Connect），实现大规模网络互通。 它的特点是采用“Hub-and-Spoke”结构，所有 VPC 连接到 TGW，而不是两两做 VPC Peering；支持跨 Region（通过 Peering）、支持路由隔离（多路由表）、支持路由传播；并且具备传递性（transitive），A 连 TGW，B 连 TGW，A 可以访问 B。

Egress-only IGW
: 专门用于 IPv6 的“只出不进”互联网网关。它允许 VPC 中的 IPv6 实例主动访问互联网（出站流量），但阻止互联网主动发起连接进入 VPC；作用类似于 IPv4 中的 NAT Gateway，但因为 IPv6 不做地址转换，所以使用 Egress-only IGW 来实现出站控制。

## 疑难问题

Public Subnet和Private Subnet的区别是什么？
: 区别在于是否通过路由表直接连接 Internet Gateway。Public Subnet 的路由表包含指向 Internet Gateway 的默认路由（如 0.0.0.0/0），子网内实例可以拥有公网 IP 并直接访问互联网；Private Subnet 没有到 Internet Gateway 的直接路由，实例通常只有私有 IP，如需访问公网需通过 NAT Gateway 或 NAT Instance 转发。

子网和路由表是一一对应的吗？
: 一个子网在任意时刻只能关联一个路由表，但一个路由表可以被多个子网关联；如果子网没有显式关联路由表，它会使用 VPC 的主路由表。

路由表、NACL、安全组的区别是什么？
: 路由表决定流量往哪里走（控制路径转发，关联子网）；NACL（Network ACL）是作用在子网级别的无状态访问控制列表，需要分别配置入站和出站规则；安全组是作用在实例级别的有状态防火墙，只需定义入站规则，返回流量自动放行。

| 特性             | 安全组 (Security Group)        | 网络 ACL (Network ACL)     |
| ---------------- | ------------------------------ | -------------------------- |
| 作用范围         | 实例 (Instance) 级别（如 EC2） | 子网 (Subnet) 级别         |
| 有状态 vs 无状态 | 有状态 (Stateful)              | 无状态 (Stateless)         |
| 规则类型         | 只能设置“允许”规则             | 可以设置“允许”和“拒绝”规则 |
| 处理顺序         | 所有规则都会被检查             | 按规则编号顺序执行         |
| 默认状态         | 默认拒绝所有入站流量           | 默认允许所有进出流量       |

有状态防火墙和无状态防火墙的区别是什么？
: 有状态（Stateful） 指系统会记录连接状态。如果入站流量被允许，返回流量会自动被允许，不需要单独配置出站规则，例如安全组就是有状态的。无状态（Stateless） 指系统不记录连接状态，入站和出站规则必须分别显式允许，返回流量不会自动放行，例如NACL。

VPC与Subnet能够覆盖多个Region或AZ吗？
: VPC 可以跨多个 Availability Zone，但不能跨 Region；如果需要跨 Region 通信，需要使用 VPC Peering（跨 Region 对等连接）、Transit Gateway 或 VPN 等方式实现。Subnet（子网）属于某一个 VPC，但每个子网只能位于一个 Availability Zone 内；因此通常会在不同 AZ 中分别创建子网，用来部署实例以实现高可用。

让Private Subnet流量指向Nat的路由表是在私有子网还是在公有子网上创建的？
: NAT Gateway 本身部署在 Public Subnet，并通过该子网的路由表连接 Internet Gateway；但指向 NAT Gateway 的那条默认路由（0.0.0.0/0）是在 Private Subnet 关联的路由表里配置的，这样私有子网的流量才会先到 NAT Gateway，再转发到公网。

Internet Gateway和NAT Gateway的区别是什么？
: 区别如下表格所示

| 对比项          | IGW (Internet Gateway)        | NAT Gateway                                |
| --------------- | ----------------------------- | ------------------------------------------ |
| 作用            | 让 VPC 与互联网双向通信       | 让私有子网实例访问互联网（仅出站）         |
| 部署位置        | 直接挂载在 VPC 上             | 部署在 Public Subnet 中                    |
| 公网访问        | 允许公网访问实例（需公网 IP） | 不允许公网主动访问私有实例                 |
| 适用子网        | Public Subnet                 | Private Subnet                             |
| 是否需要公网 IP | 实例需要公网 IP               | NAT GW 需要绑定弹性公网 IP，私有实例不需要 |
| 流量方向        | 入站 + 出站                   | 仅出站                                     |
| 典型用途        | Web 服务器对外提供服务        | 私有服务器访问外网（如更新、下载依赖）     |

为什么一个VPC可以有多个CIDR？
: 创建 VPC 时必须指定一个主 CIDR，之后可以添加辅助 CIDR（Secondary CIDR）。这样做的原因包括：当原有 IP 地址不够用时可以扩容；可以为不同业务或环境划分独立地址段；便于与本地数据中心或其他 VPC 进行地址规划避免冲突。

CIDR和Subnet的区别是什么？
: CIDR 定义了你有多少 IP，而 Subnet 是你把这些 IP 实际用在哪个“房间”里。

| 特性     | CIDR（无类别域间路由）                      | Subnet（子网）                                   |
| -------- | ------------------------------------------- | ------------------------------------------------ |
| 它是什么 | 一种记法/标准，用来描述一个 IP 地址块的大小 | 一个网络划分，VPC 内的一个实际逻辑分区           |
| 它的作用 | 告诉系统：从哪个 IP 开始，到哪个 IP 结束    | 承载资源（如 EC2），为不同性质的服务提供隔离空间 |
| 例子     | 10.0.1.0/24（代表 256 个地址）              | “Web 子网”或“数据库子网”                         |
| 灵活性   | 只是一个数字范围                            | 需要关联 Availability Zone（AZ）和 Route Table   |

私有子网实例如何控制只能通过跳板机登录？
: 核心做法是控制安全组规则：把私有实例的安全组入站规则（如 SSH 22 端口）来源设置为“Bastion Host 的安全组”而不是 0.0.0.0/0 或公网 IP，这样只有绑定该安全组的跳板机可以访问。安全组的本质是绑定在 ENI（网卡）上的虚拟有状态防火墙规则集合，它控制的是“允许哪些流量进入或离开这张网卡”。同时确保私有子网没有指向 Internet Gateway 的路由，从网络路径上也无法被公网直接访问。

外部系统访问VPC有VPN和DX两种方式，有什么区别？
: Site-to-Site VPN vs. Direct Connect (DX)

| 特性     | Site-to-Site VPN       | Direct Connect（专线）     |
| -------- | ---------------------- | -------------------------- |
| 底层路径 | 公共互联网（加密隧道） | 独占物理光纤               |
| 稳定性   | 容易受公网波动影响     | 极稳，延迟极低             |
| 开通速度 | 几分钟到几小时         | 几周到几个月（涉及拉光纤） |
| 成本     | 较低（按小时计费）     | 昂贵（有初装费和端口费）   |

VPC Flow Logs和Traffic Mirroring有什么区别？
: Traffic Mirroring流量镜像允许你复制指定网卡（ENI）上的真实流量，并将其实时发送到另一个地方（比如一台安全扫描设备）进行检查，而不会对原始流量造成任何干扰。

| 特性     | VPC Flow Logs                                | Traffic Mirroring                     |
| -------- | -------------------------------------------- | ------------------------------------- |
| 内容     | 元数据（源 IP、目的 IP、端口、协议、包大小） | 完整的数据包（包含 Payload/负载内容） |
| 类比     | 像通话记录（谁打给谁，打了几分钟）           | 像通话录音（连说话内容都录下来）      |
| 性能消耗 | 极低（由控制面处理）                         | 会占用网卡的额外带宽                  |
| 主要用途 | 计费、排查连通性、查看谁在扫描               | 深度包检测、安全分析、抓包            |

如何理解数据在不同边界传输时的收费差异？
: 数据从互联网传进 EC2 是免费的，在同一个可用区内，两台机器通过私有 IP 通信是免费的。即使在同一个 Region，只要跨了可用区（AZ），每 GB 通常收 0.01。数据从一个地区传到另一个地区（Inter-region），费用翻倍，每 GB 约 0.02。即使两台机器在同一个 AZ，如果你用 Public IP 或 Elastic IP 通信，AWS 会收 0.02/GB，因为流量绕到了公网。因此策略为：优先使用私有 IP，这不仅能省钱，还能获得更好的网络性能（延迟更低）；尽量留在同一个 AZ。

访问 S3 时，用 NAT Gateway 和用 Gateway VPC Endpoint 的成本如何对比？
: 如果走 NAT Gateway → Internet Gateway → S3，不仅要支付 NAT 的小时费用（约 0.045/小时），还要支付 NAT 的数据处理费（约 0.045/GB），即使同区域访问 S3 数据传输本身免费，也仍然会产生 NAT 处理成本；而如果改用 Gateway VPC Endpoint（S3 专用），流量直接在 AWS 内网转发，不经过 NAT，也没有 Endpoint 使用费，从而省掉 NAT 的小时费和每 GB 处理费，因此在大量访问 S3 的场景下成本明显更低。
