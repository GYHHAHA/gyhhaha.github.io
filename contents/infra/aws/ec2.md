---
short_title: EC2
---

# Elastic Compute Cloud

## 服务器

### 基本概念

chomod
: chmod xxx 是 Linux/Unix 系统里用来 修改文件权限 的命令。

x可以是0到7，含义为：

| 数字 | 符号 | 含义（对这一类用户）                           |
| ---: | :--: | ---------------------------------------------- |
|    0 | ---  | 不能读、不能写、不能执行                       |
|    1 | --x  | 只能执行（对文件是运行；对目录是“进入/穿越”）  |
|    2 | -w-  | 只能写（很少单独用；对文件可改内容但未必能读） |
|    3 | -wx  | 写 + 执行                                      |
|    4 | r--  | 只能读                                         |
|    5 | r-x  | 读 + 执行                                      |
|    6 | rw-  | 读 + 写                                        |
|    7 | rwx  | 读 + 写 + 执行                                 |

每个位置分别代表：

| 位置   | 代表谁 | 说明             |
| ------ | ------ | ---------------- |
| 第一位 | Owner  | 文件拥有者       |
| 第二位 | Group  | 文件所属组的用户 |
| 第三位 | Others | 系统里其他所有人 |

常见的使用组合包括：

| 权限 | 符号表示  | 常见用途                 |
| ---- | --------- | ------------------------ |
| 400  | r-------- | SSH 私钥（EC2 常用）     |
| 600  | rw------- | 私密配置文件             |
| 644  | rw-r--r-- | 普通文本文件             |
| 700  | rwx------ | 私有脚本/目录            |
| 755  | rwxr-xr-x | 程序、Web目录            |
| 777  | rwxrwxrwx | 所有人完全控制（不安全） |

Elastic IP
: Elastic IP是 AWS 提供的一个 固定的公网 IPv4 地址，可以绑定到 EC2 实例上，并且可以在实例之间快速切换。Elastic IP 适用于需要固定公网 IP 的单实例场景（如跳板机或需做 IP 白名单的服务），但不适合高可用或大规模 Web 架构（应使用 Load Balancer 代替）。

Placement Group
: Placement Group 是 AWS 用来控制 EC2 实例在物理硬件上“如何分布”的机制，从而优化性能或提高可用性。

| 类型      | 物理分布方式                         | AZ 范围 | 优点                 | 风险                  | 适用场景                   |
| --------- | ------------------------------------ | ------- | -------------------- | --------------------- | -------------------------- |
| Cluster   | 所有实例放在同一物理集群（同机架组） | 单 AZ   | 超低延迟、超高带宽   | 同一硬件故障影响大    | HPC、高性能计算、大数据    |
| Spread    | 每个实例分布在不同物理硬件上         | 可跨 AZ | 最大化物理隔离       | 每个 AZ 最多 7 台实例 | 小规模关键应用、主从数据库 |
| Partition | 实例分成多个分区，每个分区独立硬件   | 可跨 AZ | 分区隔离、适合大规模 | 单分区内仍可能受影响  | Hadoop、Kafka、分布式系统  |

ENI
: ENI 是 VPC 里的“虚拟网卡”，它可以独立创建，并在同一 AZ 内在 EC2 之间移动（常用于故障切换）。常见属性如下：

| 属性                   | 说明                           |
| ---------------------- | ------------------------------ |
| Primary Private IPv4   | 主私有 IP（必须有）            |
| Secondary Private IPv4 | 一个或多个辅助私有 IP          |
| Elastic IP (EIP)       | 每个私有 IPv4 可以绑定一个 EIP |
| Public IPv4            | 可分配一个公网 IPv4            |
| Security Groups        | 可绑定一个或多个安全组         |
| MAC Address            | 固定的 MAC 地址                |
| Availability Zone      | 绑定在特定 AZ，不能跨 AZ       |

Private IP与Secondary IP
: Primary Private IP 和 Secondary Private IP 都是 VPC 内部的私有 IP 地址。Primary Private IP 是网卡的“主 IP”，不能移除；Secondary Private IP 是附加 IP，可以增加、删除、或移动。

| 对比点          | Primary Private IP | Secondary Private IP |
| --------------- | ------------------ | -------------------- |
| 是否必须存在    | ✅ 必须有          | ❌ 可选              |
| 能否删除        | ❌ 不能删除        | ✅ 可以删除          |
| 能否更换        | ❌ 不能直接替换    | ✅ 可以替换          |
| 数量            | 每个 ENI 只有 1 个 | 可以有多个           |
| 是否可绑定 EIP  | ✅ 可以            | ✅ 也可以            |
| 是否随 ENI 移动 | ✅ 会一起移动      | ✅ 会一起移动        |

### 疑难问题

EC2有哪些常见配置项目？
: EC2 常见配置包括计算规格、网络设置、存储配置、安全访问、扩展管理以及购买方式六大类。

| 类别              | 主要配置项（含说明）                                                                                                                                                                                    |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 计算 (Compute)    | AMI（操作系统镜像）、Instance Type（CPU/内存规格）、vCPU/Memory（算力资源）、CPU Credits（T系列突发机制）                                                                                               |
| 网络 (Networking) | VPC（所属虚拟网络）、Subnet（所属子网/AZ）、Private IP（私有地址）、Public IP（临时公网IP）、Elastic IP（固定公网IP）、Security Group（实例级防火墙）、ENI（虚拟网卡）、Placement Group（物理部署策略） |
| 存储 (Storage)    | Root Volume（系统盘）、Additional EBS（数据盘）、Instance Store（本地临时盘）、EBS Type（磁盘性能类型）、EBS Encryption（磁盘加密）                                                                     |
| 安全 (Security)   | Key Pair（SSH登录密钥）、IAM Role（访问AWS权限）、User Data（启动脚本）、IMDS Version（元数据访问控制）                                                                                                 |
| 管理 (Management) | Auto Scaling（自动扩缩容）、Monitoring（CloudWatch监控）、Termination Protection（防误删）、Shutdown Behavior（关机行为）                                                                               |
| 计费 (Pricing)    | On-Demand（按需计费）、Reserved Instances（预留折扣）、Savings Plan（承诺消费折扣）、Spot（抢占式实例）                                                                                                 |

EC2有哪些实例类型？
: 选型核心看5个指标：vCPU、Memory、Network bandwidth、EBS bandwidth、是否支持Nitro。实例类型如下：

| 分类     | 系列前缀              | 代表实例                | 资源特点                 | 典型场景                | 关键说明                     |
| -------- | --------------------- | ----------------------- | ------------------------ | ----------------------- | ---------------------------- |
| 通用型   | T / M                 | t3、t4g、m6i、m7g       | CPU/内存均衡，T可突发    | Web服务器、应用层       | 默认选择最多，T有CPU Credits |
| 计算优化 | C / Hpc               | c6i、c7g、hpc6a         | 高vCPU性能，高网络带宽   | 高并发、批处理、HPC     | 计算密集型                   |
| 内存优化 | R / X / U             | r6i、r7g、x2idn、u-6tb1 | 高内存比例，支持TB级内存 | Redis、缓存、企业数据库 | 内存密集型                   |
| 存储优化 | I / D / H             | i4i、d3、h1             | NVMe SSD或高吞吐HDD      | 高IO数据库、大数据      | 高IOPS或大容量               |
| 加速计算 | G / P / Inf / Trn / F | g5、p5、inf2、trn1、f1  | GPU或专用芯片加速        | AI训练、推理、图形渲染  | 专用硬件加速                 |

安全组的显式并集规则是什么？
: 只有 Allow，没有 Deny，没有显式拒绝规则，默认拒绝所有未允许的流量，所以不会出现“一个 SG 允许，一个 SG 拒绝”的冲突问题。一个实例可以绑定多个安全组，多个 Security Group 是 并集（Union）关系。

安全组可以互相引用吗？
: 安全组的入站规则里，可以把“来源”设置成另一个 Security Group，而不是写 IP。即使 IP 变了，只要实例还绑定在那个 SG 里，就仍然允许访问，这是 AWS 里微服务内部通信的标准做法，实现基于“身份（SG）”而不是“IP”的访问控制。

安全组的Outbound规则对返回流量和主动发出流量的影响有什么区别？
: Security Group 是 stateful 的。Outbound 规则只会影响实例“主动发起”的新连接，例如服务器访问外部 API 或下载更新时，必须匹配出站规则才可以发送请求。但对于已经被允许的入站连接，其“返回流量”会被自动放行，即使没有对应的 Outbound 规则也不会被阻止。因此，Outbound 规则不会影响已建立连接的响应流量，只影响实例主动发起的流量。

EC2实例能绑定几个IAM Role？
: EC2 实例通过一个叫 Instance Profile 的机制来关联 IAM Role。一个 Instance 只能附加 一个 Instance Profile，一个 Instance Profile 里面只能有 一个 IAM Role。如果我需要多个权限，把多个权限策略（Policy）附加到同一个 IAM Role 上。一个 Role 可以绑定：多个 AWS Managed Policy、多个 Customer Managed Policy、inline policy。

假设EC2安装了Apache服务，用户访问公网IP的时候为什么能够看到主页？
: 当用户访问 EC2 的 Public IP 时，浏览器会向该公网地址的 80 端口发送 HTTP 请求，请求通过互联网进入 AWS 的 Internet Gateway，再根据子网路由表转发到对应的 EC2 实例；如果安全组允许 80 端口入站流量，流量就会进入实例，而正在监听 80 端口的 Apache（httpd）服务会读取 /var/www/html/index.html 并将网页内容返回给用户，因此浏览器就能显示 Apache 页面。

## 存储

### 基本概念

EBS
: EBS（Elastic Block Store）是 AWS 为 EC2 提供的持久化块存储服务，相当于云中的硬盘。
它可以作为 EC2 的系统盘或数据盘使用，支持独立扩容、快照备份和加密，即使实例停止或重启，数据也不会丢失（只要不删除卷）。EBS 适用于数据库、应用数据存储以及需要高可靠性的工作负载。

Instance Store
: Instance Store 是 EC2 实例附带的本地物理磁盘存储（通常是 NVMe SSD），直接连接在宿主机上。特点：性能非常高（低延迟、高 IOPS），因为数据直接写在宿主机的本地磁盘上；但它是非持久化存储，实例终止、停止或宿主机故障时数据都会丢失；同时它不能单独创建或独立使用，只能随特定实例类型一起提供，适合缓存、临时计算和高速数据处理场景。

| 对比项          | Instance Store     | EBS        |
| --------------- | ------------------ | ---------- |
| 存储位置        | 本地物理机         | 网络块存储 |
| 是否持久        | ❌ 否              | ✅ 是      |
| 性能            | 极高               | 高         |
| 可否 Stop       | ❌（旧类型不支持） | ✅         |
| 可否做 Snapshot | ❌                 | ✅         |

AMI
: AMI（Amazon Machine Image）是用于启动 EC2 实例的镜像模板，包含操作系统、已安装的软件以及启动所需的配置（如 root 卷映射等）；当创建一台新的 EC2 实例时，本质上是基于某个 AMI 复制出一台服务器，因此 AMI 常用于快速部署、批量扩容、环境标准化以及系统备份和灾难恢复。

Multi-Attach
: EBS Multi-Attach 是 io1/io2 系列卷提供的功能，允许在同一个 Availability Zone 内把同一个 EBS 卷同时挂载到最多 16 台 EC2 实例，并且每台实例都拥有读写权限，从而实现高可用的集群架构。它常用于需要共享高性能块存储的集群应用（如数据库或企业级应用集群），但应用必须自己处理并发写入问题，并且需要使用支持集群的文件系统（不能使用普通的 XFS 或 EXT4）。简单说：Multi-Attach 让多个实例共享同一块高性能 EBS，用于提升应用可用性和一致性控制。

EFS
: Amazon EFS（Elastic File System）是一个托管的 NFSv4.1 网络文件系统，基于 POSIX 标准，主要用于 Linux 实例之间共享文件数据。它可以被多台 EC2 同时挂载，并支持跨多个可用区（Multi-AZ）部署，从而实现高可用和高耐久性；存储容量会根据使用情况自动扩展到 PB 级，无需手动规划，并按实际使用量付费。同时，EFS 使用 Security Group 控制访问，支持 KMS 加密，适合需要多实例共享数据的应用场景（如 Web 服务器集群、CMS、容器共享存储等）。

### 疑难问题

EC2实例和EBS是一对多吗？
: EC2 实例和 EBS 的关系通常是一对多（1:N），一台 EC2 实例可以挂载多个 EBS 卷。EBS 只能挂载到 同一个 AZ 的 EC2，不能跨 AZ 挂载，跨 AZ 需要 Snapshot 再恢复。

EC2 启动是否必须至少有一个 root EBS？
: EC2 必须有一个 root volume，但这个 root volume 可以是 EBS，也可以是 Instance Store。现在绝大多数实例默认使用 EBS 作为 root volume。

创建了EBS并挂到EC2实例后就能直接使用吗？
: 创建 EBS 并挂载到 EC2 之后，还需要在操作系统里做初始化和挂载操作，才能真正使用。

```bash
# 1️⃣ 查看系统识别到的磁盘设备
lsblk

# 假设新挂载的 EBS 设备是 /dev/xvdf
# ⚠️ 请根据 lsblk 实际输出确认设备名

# 2️⃣ 对新磁盘进行格式化（第一次使用才需要）
# 这里使用 xfs 文件系统（也可以改成 ext4）
sudo mkfs -t xfs /dev/xvdf

# 3️⃣ 创建挂载目录
sudo mkdir /data

# 4️⃣ 挂载磁盘到目录
sudo mount /dev/xvdf /data

# 5️⃣ 验证是否挂载成功
df -h

# 6️⃣ 获取磁盘的 UUID（用于开机自动挂载）
sudo blkid /dev/xvdf

# 7️⃣ 编辑 /etc/fstab 添加自动挂载（使用 UUID 更安全）
# 将下面这一行添加到 /etc/fstab 文件末尾
# UUID=你的UUID值  /data  xfs  defaults,nofail  0  2

sudo vi /etc/fstab
```

EBS Snapshot有哪些增强功能？
: EBS Snapshot 只有一种类型，但可以开启三种增强功能：Archive 用于将快照转入更便宜的归档存储层（成本约低 75%），适合长期备份和合规存档，但恢复需要 24–72 小时；Recycle Bin 提供误删保护，可设置 1 天到 1 年的保留规则，避免因人为操作导致数据永久丢失；Fast Snapshot Restore（FSR） 通过提前完成快照初始化来消除首次访问延迟，使从快照创建的 EBS 卷可以立即获得完整性能，但需要额外费用。简单理解就是：Archive 省钱、Recycle Bin 防误删、FSR 提性能。

AMI和Docker Image有什么区别？
: AMI 是虚拟机级别的镜像，用于创建完整的 EC2 实例，包含操作系统、内核及相关配置，相当于一整台服务器的模板；而 Docker Image 是容器级别的镜像，只包含应用程序及其依赖，不包含完整操作系统，它运行在已有操作系统之上的 Docker 引擎中，属于进程级隔离。

EBS有哪些类型？
: 总结如下表：

| 类型              | 存储介质                    | 容量范围         | IOPS / 吞吐特性                                                    | 是否可做 Boot Volume | 典型使用场景                             |
| ----------------- | --------------------------- | ---------------- | ------------------------------------------------------------------ | -------------------- | ---------------------------------------- |
| gp3               | SSD                         | 1 GiB – 16 TiB   | 默认 3,000 IOPS / 125 MiB/s；可独立扩展至 16,000 IOPS / 1000 MiB/s | ✅                   | 通用应用、系统盘、开发测试、虚拟桌面     |
| gp2               | SSD                         | 1 GiB – 16 TiB   | 3 IOPS/GB；可 burst 至 3,000 IOPS；最大 16,000 IOPS                | ✅                   | 旧通用场景（现在通常推荐 gp3）           |
| io1               | SSD (Provisioned IOPS)      | 4 GiB – 16 TiB   | 最多 64,000 IOPS（Nitro）/ 32,000（其他）；IOPS 可独立配置         | ✅                   | 关键业务系统、数据库、高一致性应用       |
| io2 Block Express | SSD (高端)                  | 4 GiB – 64 TiB   | 亚毫秒延迟；最高 256,000 IOPS；IOPS:GiB = 1000:1                   | ✅                   | 大型数据库、企业级核心系统、极低延迟需求 |
| st1               | HDD（Throughput Optimized） | 125 GiB – 16 TiB | 吞吐型；最高 500 MiB/s；最高 500 IOPS                              | ❌                   | 大数据、数据仓库、日志处理               |
| sc1               | HDD（Cold）                 | 125 GiB – 16 TiB | 吞吐型；最高 250 MiB/s；最高 250 IOPS                              | ❌                   | 冷数据、归档数据、成本优先场景           |

EBS Encryption是什么？
: EBS Encryption 是对 EBS 卷进行端到端加密的功能，使用 AWS KMS 管理密钥并采用 AES-256 算法，对数据在静态存储（at rest）和实例与卷之间传输过程（in transit）都进行加密。所有基于加密卷创建的 Snapshot 以及从这些 Snapshot 创建的新卷都会自动保持加密状态，整个加解密过程对用户透明，几乎不会带来明显的性能影响。
: 如果需要加密一个未加密的 EBS 卷，不能直接开启加密，而是需要先创建该卷的 Snapshot，然后在复制 Snapshot 时启用加密，最后从加密后的 Snapshot 创建新的加密卷，并将其挂载回实例。简单来说，EBS 加密是默认继承、自动扩展的，但对已有未加密卷需要通过 Snapshot 复制方式实现。

EFS性能相关模式有哪些？
: Performance Mode 控制延迟和并发特性，而 Throughput Mode 控制带宽大小，两者互不替代。

| 类别             | 模式                    | 特点                     | 适用场景           |
| ---------------- | ----------------------- | ------------------------ | ------------------ |
| Performance Mode | General Purpose（默认） | 低延迟                   | Web、CMS、常规应用 |
|                  | Max I/O                 | 更高并发与吞吐，延迟较高 | 大数据、媒体处理   |
| Throughput Mode  | Bursting                | 吞吐随存储容量增长       | 普通应用           |
|                  | Provisioned             | 自定义固定吞吐           | 需要稳定吞吐的应用 |
|                  | Elastic                 | 自动根据负载扩缩吞吐     | 不可预测工作负载   |

EFS的存储模式有哪些？
: 模式如下，可通过Lifecycle Policy转移。

| 类型                    | 说明     | 成本             |
| ----------------------- | -------- | ---------------- |
| Standard                | 频繁访问 | 高               |
| IA（Infrequent Access） | 低频访问 | 更便宜，读取收费 |
| Archive                 | 极少访问 | 最便宜           |

EFS和EBS的区别有哪些？
: 总结如下：

| 对比             | EFS      | EBS                            |
| ---------------- | -------- | ------------------------------ |
| 是否可多实例挂载 | ✅       | ❌（除 Multi-Attach 特殊情况） |
| 是否跨 AZ        | ✅       | ❌                             |
| 是否自动扩容     | ✅       | ❌                             |
| 成本             | 高       | 低                             |
| 延迟             | 高于 EBS | 低                             |

## 负载与扩容

### 基本概念

Health Check
: Health Check 是负载均衡器中的关键机制，用来定期检测后端实例是否能够正常处理请求。负载均衡器会按照指定的协议（如 HTTP）、端口和路径（例如 /health）向实例发送探测请求，如果返回状态码为 200（OK）则认为实例健康，否则会将其标记为不健康并停止向其转发流量，从而确保只有可正常响应的实例对外提供服务，提高系统的可用性和稳定性。

ALB
: Application Load Balancer（ALB）工作在第 7 层（HTTP/HTTPS），能够基于请求内容进行智能路由，而不仅仅是基于 IP 和端口。它支持 HTTP/2 和 WebSocket，支持 HTTP 自动重定向到 HTTPS，并提供固定的 AWS 域名（xxx.region.elb.amazonaws.com）。ALB 可以根据路径、域名、Header、Query String 等条件转发流量，非常适合需要精细化流量控制的应用架构。
: ALB 非常适合微服务架构和容器化环境（如 Docker 和 Amazon ECS）。多个应用可以共享同一个 ALB，通过不同规则转发到不同 Target Group。它还支持动态端口映射，特别适用于 ECS 场景：当容器被分配随机端口时，ALB 可以自动将请求转发到对应端口，使容器调度更加灵活高效。

NLB
: Network Load Balancer 工作在第 4 层（Layer 4），基于 TCP/UDP 协议进行流量转发，而不解析应用层内容。它能够处理每秒百万级请求，具备超低延迟和极高吞吐能力，适用于对性能要求极高的场景。NLB 在每个 Availability Zone 都会分配一个静态 IP，并支持绑定 Elastic IP。这对于需要 IP 白名单（whitelisting）或必须使用固定公网 IP 的系统非常重要。NLB 主要用于极端性能场景，特别是 TCP 或 UDP 流量，例如金融系统、实时通信、游戏服务器等。

GWLB
:Gateway Load Balancer（GWLB）用于在 AWS 中部署和扩展第三方网络虚拟设备，如防火墙、IDS/IPS、深度包检测（DPI）和流量分析系统等。它工作在第 3 层（网络层），处理的是 IP 数据包，并将“透明网关”和“负载均衡”能力结合起来：既作为流量的统一入口/出口，又将流量分发到多个安全设备，实现可扩展的流量检查架构。GWLB 通过 GENEVE 协议（端口 6081）对流量进行封装，再交由后端设备处理。
: GWLB 的 Target Group 支持 EC2 实例和 VPC 内的私有 IP 地址，这些目标通常运行虚拟防火墙等安全设备。流量会被封装后发送到这些实例进行检测或处理，再按原路径返回，实现对网络流量的透明检查，而无需改变应用架构。

Sticky Session
: Sticky Sessions（会话保持）是指通过启用粘性机制，使同一个客户端在访问负载均衡器时始终被转发到同一台后端实例，从而避免会话数据丢失。该功能适用于 Classic Load Balancer、Application Load Balancer 和 Network Load Balancer，通常通过设置带有可控过期时间的 Cookie 来实现。它的典型场景是保证用户登录状态或购物车数据不丢失，但需要注意，启用粘性会话可能导致后端实例之间的负载分布不均衡。

Cross-Zone LB
: Cross-Zone Load Balancing（跨可用区负载均衡）是指负载均衡器可以将来自某一个 AZ 的请求，分发到所有 AZ 中注册的后端实例，而不仅仅是当前 AZ 内的实例。开启后，每个负载均衡节点都会将流量平均分配到所有可用区中的目标实例，从而避免某个 AZ 实例较少却承担过多流量的情况，提高整体资源利用率和负载均衡效果；关闭时，请求只会在对应 AZ 内部进行分发，可能导致不同 AZ 之间负载不均。

| 负载均衡类型                    | 默认是否开启 Cross-Zone                | 是否产生跨 AZ 流量费用     |
| ------------------------------- | -------------------------------------- | -------------------------- |
| Application Load Balancer (ALB) | 默认开启（可在 Target Group 级别关闭） | 不收取跨 AZ 数据费用       |
| Network Load Balancer (NLB)     | 默认关闭                               | 开启后会收取跨 AZ 数据费用 |
| Gateway Load Balancer (GWLB)    | 默认关闭                               | 开启后会收取跨 AZ 数据费用 |
| Classic Load Balancer (CLB)     | 默认关闭                               | 开启后不收取跨 AZ 数据费用 |

SSL/TLS
: SSL/TLS 用于在客户端与负载均衡器之间建立加密连接，实现传输过程中的数据加密（in-flight encryption）。SSL（Secure Sockets Layer）是早期的加密协议，TLS（Transport Layer Security）是其更新版本，如今实际使用的主要是 TLS，但人们仍习惯称为 SSL 证书。公网证书通常由受信任的证书颁发机构（CA）签发。SSL/TLS 证书具有有效期，需要定期续期以保证通信安全。

SNI
: 当客户端访问不同域名时，请求都会先到达同一个 ALB。但在 TLS 握手的最开始阶段，客户端会在握手信息中主动声明它要访问的主机名（Hostname）。ALB 收到这个信息后，就能根据域名选择对应的 SSL 证书，然后完成加密连接。之后，再根据七层路由规则把请求转发到对应的 Target Group。SNI 解决了“一个负载均衡器同时服务多个 HTTPS 网站”的问题，使多个域名可以共享同一个 IP 和同一个 ALB，而无需为每个域名单独配置负载均衡器。需要注意的是，SNI 只支持新一代负载均衡器（ALB、NLB）以及 CloudFront，不支持旧版 Classic Load Balancer（CLB）。

Deregistration Delay
: Deregistration Delay（也叫 Connection Draining）是指当某个 EC2 实例从负载均衡器中被移除（例如正在下线或变为不健康）时，负载均衡器会停止向该实例发送新的请求，但会在一段时间内继续等待该实例上“正在处理中的请求（in-flight requests）”完成，如图中所示该实例进入 DRAINING 状态，而新的连接则被转发到其他健康实例。这个等待时间可配置为 1～3600 秒（默认 300 秒），也可以设置为 0 直接关闭。它的作用是在实例下线或滚动更新时避免请求被中断，从而保证服务平滑过渡。

ASG
: Auto Scaling Group（ASG）是 AWS 中用于自动管理 EC2 实例数量的服务，它可以根据设定的最小、期望和最大实例数，以及监控指标（如 CPU 使用率、请求数量等）自动扩容或缩容实例，从而应对流量变化并保持应用高可用性。ASG 还能在实例变为不健康或宕机时自动替换实例，通常与负载均衡器（如 ALB/NLB）结合使用，实现弹性伸缩和故障自愈能力。

| 分类         | 配置项                                                                                                                              | 说明                                                     |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| 实例启动配置 | Launch Template（包含 AMI、Instance Type、User Data、EBS、Security Groups、SSH Key Pair、IAM Role、网络与子网信息、负载均衡器信息） | 定义 ASG 启动 EC2 实例所需的完整配置                     |
| 容量控制     | Min Size / Max Size / Desired（Initial）Capacity                                                                                    | 定义最小、最大以及期望运行的实例数量                     |
| 扩展策略     | Scaling Policies                                                                                                                    | 定义根据监控指标（如 CPU、请求数等）自动扩容或缩容的规则 |

### 疑难问题

为什么需要Load Balancer（负载均衡）？
: 将流量分摊到多台后端实例
: 提供统一的访问入口（DNS）
: 自动处理后端实例故障
: 定期对实例进行健康检查
: 提供 SSL 终止（HTTPS）功能
: 支持基于 Cookie 的会话保持（Stickiness）
: 支持跨多个可用区的高可用部署
: 实现公网流量与私网后端实例的隔离

ALB具备哪些路由能力？
: 列表如下：

| 路由类型                           | 示例                                   | 说明                                       |
| ---------------------------------- | -------------------------------------- | ------------------------------------------ |
| 基于路径（Path-based Routing）     | example.com/users<br>example.com/posts | 根据 URL 路径将流量转发到不同 Target Group |
| 基于域名（Host-based Routing）     | one.example.com<br>other.example.com   | 根据请求中的 Host 头进行路由               |
| 基于查询参数（Query String）       | example.com/users?id=123               | 根据 URL 查询参数进行匹配                  |
| 基于请求头（Header-based Routing） | 根据特定 Header 值                     | 可用于灰度发布或特定客户端流量控制         |

ALB可以转发到哪些Target Groups？
: Target Group 是 ALB 实际转发流量的目标集合。ALB 可以同时关联多个 Target Group，并根据规则将流量分发到不同组。Target 类型可以是 EC2 实例（通常由 Auto Scaling Group 管理）、ECS Tasks、Lambda 函数（HTTP 请求会转换为 JSON 事件）或私有 IP 地址。健康检查是在 Target Group 层面配置的，每个 Target Group 可以拥有独立的健康检查规则。

什么是ALB的客户端IP机制？
: 当客户端请求经过 ALB 转发到后端实例时，后端服务器看到的来源 IP 是 Load Balancer 的私有 IP，而不是客户端真实 IP。ALB 会将客户端真实信息写入 HTTP Header 中，包括 X-Forwarded-For（真实客户端 IP）、X-Forwarded-Port（客户端端口）以及 X-Forwarded-Proto（原始协议 HTTP/HTTPS）。因此，如果后端应用需要获取真实用户 IP，必须从这些 Header 中读取。

NLB支持哪些目标和协议？
: NLB 的 Target Group 支持多种目标类型，包括 EC2 实例、VPC 内的私有 IP 地址，以及 Application Load Balancer（可以将 NLB 作为前端入口，ALB 作为后端进行七层处理）。这种设计使 NLB 在不同架构场景下具有较高的灵活性，例如结合 ALB 实现“高性能入口 + 七层路由”的架构模式。
: 在健康检查方面，NLB 支持 TCP、HTTP 和 HTTPS 协议。与 ALB 不同，NLB 不进行七层内容解析或基于路径的智能路由，而是基于四层连接进行转发，因此更加轻量，能够提供更高性能和更低延迟，适用于对吞吐量和实时性要求极高的场景。

粘性会话以来Cookie实现，有哪些方式？
: 整理如下：

| 类型                      | 子类型             | Cookie 生成方      | Cookie 名称                    | 说明                       | 注意事项                                                                           |
| ------------------------- | ------------------ | ------------------ | ------------------------------ | -------------------------- | ---------------------------------------------------------------------------------- |
| Application-based Cookies | Custom Cookie      | 由后端目标实例生成 | 自定义名称                     | 可包含应用所需的自定义属性 | 每个 Target Group 需单独指定名称；不要使用 AWSALB、AWSALBAPP、AWSALBTG（AWS 保留） |
| Application-based Cookies | Application Cookie | 由负载均衡器生成   | AWSALBAPP                      | ALB 自动生成并管理         | 仅用于 ALB                                                                         |
| Duration-based Cookies    | —                  | 由负载均衡器生成   | AWSALB（ALB）<br>AWSELB（CLB） | 基于过期时间控制会话保持   | 适用于 ALB 和 CLB                                                                  |

SSL和LB如何协作？
: SSL/TLS 与 Load Balancer（负载均衡器）的协作核心是由负载均衡器负责处理 HTTPS 加密连接，这通常称为 SSL Termination（SSL 终止）。客户端通过 HTTPS 访问应用时，负载均衡器上配置的 X.509 证书（通常通过 ACM 管理）会与客户端完成 TLS 握手并建立加密连接。负载均衡器随后解密请求流量，再将请求以 HTTP 或 HTTPS 的形式转发到后端 EC2 实例处理，响应返回时再加密发送给客户端。
: 这种架构的优势在于统一管理证书、减轻后端实例的加密计算负担，并支持 SNI 多证书、多域名以及自定义 TLS 安全策略（如限制协议版本和加密套件）。此外，在某些场景下（例如使用 NLB），也可以采用 SSL Pass-through，让负载均衡器不解密流量，而由后端实例自行处理 TLS，从而满足特定安全或性能需求。

ASG的Scaling Policies有哪些？
: 整理如下：

| 类型               | 子类型                  | 触发方式                 | 配置方式                       | 示例                                         | 适用场景                               |
| ------------------ | ----------------------- | ------------------------ | ------------------------------ | -------------------------------------------- | -------------------------------------- |
| Dynamic Scaling    | Target Tracking Scaling | 自动根据目标指标动态调整 | 设置一个目标值，ASG 自动维持   | 保持 ASG 平均 CPU 在 40%                     | 最常用、简单配置、自动平滑扩缩容       |
| Dynamic Scaling    | Simple / Step Scaling   | 基于 CloudWatch 告警触发 | 定义阈值 + 扩缩容步长          | CPU > 70% 增加 2 台；CPU < 30% 减少 1 台     | 需要精细控制扩缩容节奏                 |
| Scheduled Scaling  | —                       | 按时间计划触发           | 预先设定时间与容量             | 每周五 17:00 将最小容量提高到 10             | 已知固定流量高峰                       |
| Predictive Scaling | —                       | 基于历史数据预测未来负载 | 系统分析历史负载趋势并提前扩容 | 根据过去 2 周的每日高峰，提前在 18:00 前扩容 | 流量有明显周期性（电商、工作日高峰等） |
