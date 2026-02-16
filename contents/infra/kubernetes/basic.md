---
short_title: Basic
---

# Kubernetes Basic

## Master

### 基本概念

#### 组件

etcd
: etcd 是 Kubernetes 的分布式键值数据库，用来存储整个集群的所有状态数据，比如 Pod、Node、Deployment、Service、ConfigMap 等信息。它保存的是集群的“真实状态”，所有组件都通过 API Server 读写 etcd。如果 etcd 出现问题，整个集群会无法正常工作。

kube-apiserver
: kube-apiserver 是 Kubernetes 的唯一对外入口，所有请求（kubectl、Controller、Scheduler、kubelet 等）都必须通过它访问集群。它负责认证、鉴权、参数校验，并把合法的数据写入 etcd，同时把变更通知其他组件。可以理解为整个集群的“总控中心”。

kube-controller-manager
: kube-controller-manager 是一组控制器的集合，负责不断比较“期望状态”和“当前状态”，并自动修正差异（称为调谐循环）。例如当某个 Pod 挂掉时，它会自动重新创建；当副本数不足时，它会补齐副本。它相当于 Kubernetes 的自动运维系统。

cloud-controller-manager
: cloud-controller-manager 负责 Kubernetes 与云厂商平台的对接，比如创建云负载均衡器、管理云主机、分配公网 IP 等。当你创建 type=LoadBalancer 的 Service 时，它会调用云厂商 API 创建真实的云资源。在裸机环境中通常不会使用这个组件。

kube-scheduler
: kube-scheduler 负责决定每个新创建的 Pod 应该运行在哪个 Node 上。它会根据资源情况（CPU、内存）、节点标签、污点容忍等条件进行过滤和打分，选出最合适的节点，并将调度结果写回 API Server。它是 Kubernetes 的调度决策核心。

### 疑难问题

Master节点的数据流转与监听机制是什么？
: 在 Kubernetes 中，etcd 负责存储数据，但所有组件只与 kube-apiserver 交互，而不会直接访问 etcd。当某个组件（例如 Controller）发现期望状态与当前状态不一致时，它会通过 kube-apiserver 创建或更新资源对象；kube-apiserver 在完成认证、鉴权、校验及准入控制后，将数据写入 etcd。etcd 仅负责持久化存储和一致性保证，并不会主动通知其他组件。
: 当 etcd 中的数据发生变化后，kube-apiserver 会通过 Watch 机制 将变更推送给所有正在监听该资源的组件（如 Scheduler、kubelet 等）。例如，Scheduler 监听到一个尚未绑定节点的 Pod 后，会计算并选择合适的 Node，然后再次通过 kube-apiserver 更新 Pod 的 nodeName 字段；随后，目标节点上的 kubelet 监听到该变更，开始创建容器并将运行状态写回 kube-apiserver。整个过程始终遵循“组件监听 API Server → 修改资源状态 → API Server 写入 etcd → 再由 API Server 广播变更”的模式。

有哪些常见的控制器？
: 最常用的 Controller 是 Deployment、ReplicaSet、EndpointSlice、DaemonSet 和 HPA，因为它们分别负责应用发布与滚动更新、副本数维护、Service 后端流量管理、节点级守护进程部署以及自动扩缩容，是日常部署和运行业务几乎一定会涉及的核心控制器。

| Controller                   | 作用说明                                                                                               |
| ---------------------------- | ------------------------------------------------------------------------------------------------------ |
| ReplicaSet Controller        | 保证实际运行的 Pod 数量与 spec.replicas 一致，通过创建或删除 Pod 进行调谐                              |
| Deployment Controller        | 通过创建和管理多个 ReplicaSet 实现滚动更新、回滚和版本控制                                             |
| StatefulSet Controller       | 管理有状态 Pod，保证稳定的网络标识、持久化存储绑定，以及按顺序创建和删除                               |
| DaemonSet Controller         | 确保符合条件（NodeSelector/Taint 等）的每个 Node 上运行一个 Pod 副本                                   |
| Node Controller              | 监控 Node 心跳和状态，标记 NotReady/Unreachable，并触发 Pod 驱逐（非直接迁移）                         |
| EndpointSlice Controller     | 根据符合 Service Selector 且 Ready 的 Pod 动态生成和更新 EndpointSlice 资源                            |
| Service Controller           | 在云环境中根据 Service 类型（如 LoadBalancer）创建、更新或删除云厂商负载均衡资源                       |
| HPA Controller               | 根据指标（CPU、内存或自定义指标）计算期望副本数，并更新目标对象（如 Deployment/ReplicaSet）的 replicas |
| Job Controller               | 确保指定数量的 Pod 成功完成任务，失败时根据策略重试                                                    |
| CronJob Controller           | 根据定时规则周期性创建 Job 对象                                                                        |
| Namespace Controller         | 在删除 Namespace 时清理其中所有命名空间级资源，完成后移除 Finalizer                                    |
| ServiceAccount Controller    | 自动为 Namespace 创建默认 ServiceAccount                                                               |
| Token Controller             | 为 ServiceAccount 创建和管理用于 API 访问的 Token（现代版本主要基于 TokenRequest API）                 |
| Garbage Collector Controller | 根据 OwnerReference 关系自动级联删除或清理孤儿资源                                                     |
| ResourceQuota Controller     | 统计并限制 Namespace 级别的资源使用总量                                                                |
| LimitRange Controller        | 为容器设置默认资源请求/限制，并校验其是否在允许范围内                                                  |

## Node

### 基本概念

#### 组件

kubelet
: kubelet 是运行在每个 Node 上的核心代理进程，负责与 API Server 通信，监听分配给本节点的 Pod 资源对象，并根据 PodSpec 调用容器运行时创建和管理容器。同时它会定期上报节点状态和 Pod 运行状态，是节点侧的生命周期管理核心组件。

kube-proxy
: kube-proxy 负责实现 Kubernetes Service 的网络转发功能，它会监听 Service 和 EndpointSlice 的变化，并在本节点上维护 iptables 或 IPVS 规则，将访问 Service 的流量正确转发到后端 Pod，实现集群内部的服务负载均衡。

容器运行时（Container Runtime）
: 容器运行时是实际创建和运行容器的软件组件，例如 containerd。kubelet 通过 CRI（Container Runtime Interface）与容器运行时交互，由运行时负责拉取镜像、创建容器、启动进程以及管理容器的生命周期。

#### 其他

Service
: Service 是 Kubernetes 中对一组 Pod 提供统一访问入口的抽象资源，它通过 selector 选择后端 Pod，并为它们分配一个稳定的虚拟 IP（ClusterIP）和端口。由于 Pod 是动态创建和销毁的，IP 会变化，而 Service 提供一个固定不变的访问地址，从而实现服务发现和负载均衡（L4）。客户端只需要访问 Service，而不需要关心具体的 Pod。

Ingress
: Ingress 是 Kubernetes 中用于管理“集群外部 HTTP/HTTPS 访问规则”的资源对象，它通过域名和路径规则，将外部流量转发到集群内部的不同 Service，实现统一入口和七层（L7）路由控制。

EndpointSlice
: EndpointSlice 是用来存储某个 Service 对应后端 Pod 列表的资源对象，它记录了每个 Pod 的 IP、端口以及 Ready 状态等信息。当 Pod 状态发生变化时，EndpointSlice 会被自动更新，kube-proxy 根据它生成转发规则，从而实现流量的动态摘除和加入。相比旧版的 Endpoints 资源，EndpointSlice 支持更大规模集群，性能更好。

ClusterIP
: ClusterIP 是 Kubernetes Service 的默认类型，它为一组符合 selector 条件的 Pod 分配一个仅在集群内部可访问的虚拟 IP 地址。该 IP 由 kube-proxy 通过 iptables 或 IPVS 规则实现流量转发，将访问 Service 的请求负载均衡到后端 Pod 上。ClusterIP 主要用于集群内部服务之间的通信，不对外部网络暴露。

NodePort
: NodePort 是在 ClusterIP 的基础上扩展的一种 Service 类型，它会在每个 Node 上分配并监听一个固定端口（通常范围为 30000–32767），使外部客户端可以通过“任意节点 IP + NodePort”访问该服务。外部请求进入节点后，由 kube-proxy 转发到对应的 ClusterIP，再负载均衡到后端 Pod，从而实现对外暴露服务。

LoadBalancer
: LoadBalancer 是 Kubernetes Service 的一种类型，用于将服务直接暴露到集群外部，并通过云厂商提供的外部负载均衡器对流量进行分发。当创建 type=LoadBalancer 的 Service 时，Kubernetes 会调用云平台接口（如 AWS、阿里云、GCP 等）自动创建一个外部负载均衡实例，并分配一个公网 IP 或外部访问地址，所有外部流量首先进入该负载均衡器，再被转发到集群内部的后端 Pod。

### 疑难问题

Service 与 EndpointSlice 工作原理是什么？
: 假设你部署了一个 Web 应用，副本数是 3 个，现在集群中运行着 3 个 Pod，它们的 IP 分别是 10.244.1.2、10.244.2.5 和 10.244.3.7。由于 Pod 是会重建和变化的，IP 也可能改变，所以你创建了一个 Service，系统为它分配了一个固定的虚拟 IP，例如 10.96.0.10。以后客户端只需要访问 10.96.0.10:80，而不需要关心后端具体有哪些 Pod。无论 Pod 重启还是扩容，客户端访问地址始终不变，这就是 Service 的作用——提供一个稳定的统一入口。
: 而 EndpointSlice 则负责记录这个 Service 当前对应的真实后端 Pod 列表。例如它会记录：10.244.1.2:80、10.244.2.5:80、10.244.3.7:80，并标记它们是否 Ready。kube-proxy 会根据这个列表生成转发规则，把访问 10.96.0.10 的流量负载均衡到这些 Pod 上。如果其中一个 Pod（比如 10.244.2.5）宕机变为 NotReady，EndpointSlice 会自动更新并移除它，kube-proxy 也会同步更新规则，后续流量就不会再打到故障 Pod，实现自动摘除与动态负载均衡。

ClusterIP、NodePort 和 LoadBalancer 的区别是什么？
: ClusterIP 仅在集群内部提供稳定访问入口，不对外暴露； NodePort 在每个节点上开放一个对外端口，使外部流量能够通过“节点 IP + 端口”进入集群并访问服务，本质上 NodePort = ClusterIP + 节点对外端口映射；LoadBalancer 则在 NodePort 的基础上，由云平台创建一个外部负载均衡器并分配公网 IP，使外部流量通过公网 IP 进入集群，而无需直接暴露节点 IP，本质上 LoadBalancer = NodePort + 云负载均衡入口。

## 资源对象

### 基本概念

pause 容器
: pause 容器是 Kubernetes 在每个 Pod 中自动创建的基础容器，它本身不运行任何业务，只负责创建并持有 Pod 的网络命名空间等共享资源，使同一 Pod 内的多个业务容器能够共享同一个 IP 地址、端口空间和 localhost 通信环境。

StatefulSet
: StatefulSet 是 Kubernetes 中用于管理“有状态应用”的工作负载控制器，它为每个 Pod 提供稳定的网络标识、固定的名称顺序以及独立且持久化的存储，并按照有序方式进行创建、扩缩容和删除。和 Deployment 不同，Deployment 适合无状态应用（比如 Web 服务），Pod 可以随便替换、名字随机、存储可丢；而 StatefulSet 适合数据库、Kafka、Redis 集群这类有状态服务，因为它保证：Pod 名字固定（如 mysql-0、mysql-1）、Pod 重建后仍然使用原来的存储卷、创建和删除按顺序执行（先 0 再 1 再 2）。

Headless Service
: Headless Service 是一种 不分配 ClusterIP 的 Service（clusterIP: None），它不会提供一个统一的虚拟 IP 进行负载均衡，而是通过 DNS 直接返回后端 Pod 的真实 IP 地址列表，通常与 StatefulSet 配合使用，用于有状态应用之间的精确访问和节点发现。

volumeClaimTemplates
: volumeClaimTemplates 是 StatefulSet 中用于“自动为每个 Pod 创建独立 PVC”的模板配置。当 StatefulSet 创建 Pod 时，会根据这个模板为每个 Pod 动态生成一个专属的 PersistentVolumeClaim（PVC），从而保证每个 Pod 都拥有独立且持久化的存储空间。

DaemonSet
: DaemonSet 是一种保证“每个符合条件的 Node 上都运行一个 Pod 副本”的控制器，常用于部署日志收集、监控代理或网络插件等节点级服务。只要有新的 Node 加入集群，DaemonSet 就会自动在该节点上创建对应的 Pod；如果某个 Node 被删除，对应的 Pod 也会自动被清理。

### 疑难问题

Kubernetes 资源是如何按作用范围划分的？
: 在 Kubernetes 中，资源按照“作用范围（Scope）”可以分为两类：命名空间级资源和集群级资源。命名空间级资源必须属于某个 Namespace，只在该命名空间内生效，例如 HPA、LimitRange 等；而集群级资源不属于任何 Namespace，作用于整个集群，例如 Node、ClusterRole 和 ClusterRoleBinding。这种划分方式是基于资源的管理范围和隔离需求，而不是功能分类，本质上就是区分“是否受 Namespace 隔离控制”。

Deployment和ReplicaSet有什么区别？
: ReplicaSet 只负责保证指定数量的 Pod 副本始终运行，而 Deployment 是对 ReplicaSet 的更高层封装，用来管理应用版本的发布、滚动更新和回滚。也就是说，ReplicaSet 关注的是“副本数量是否正确”，当少了就创建、多了就删除；而 Deployment 关注的是“应用如何升级和演进”，它通过创建和切换不同版本的 ReplicaSet 来实现平滑发布和历史版本回滚。因此，在实际使用中通常直接创建 Deployment，而不是手动创建 ReplicaSet。

Deployment的滚动升级和回滚是什么，是怎么做的？
: 滚动升级（Rolling Update）是指在不中断服务的情况下，逐步用新版本 Pod 替换旧版本 Pod；回滚（Rollback）是指当新版本出现问题时，快速恢复到之前的稳定版本。整个过程中，Deployment 会确保始终有一定数量的 Pod 对外提供服务，避免全部下线导致服务不可用。
: Deployment 本身并不直接管理 Pod，它是通过“控制多个 ReplicaSet”来实现升级和回滚的。当你修改镜像版本时，Deployment 会创建一个新的 ReplicaSet（对应新版本），同时逐步减少旧 ReplicaSet 的副本数、增加新 ReplicaSet 的副本数，这个比例由 maxSurge 和 maxUnavailable 控制。当升级完成后，旧 ReplicaSet 会被保留（副本数为 0），用于将来回滚。如果需要回滚，Deployment 只需把旧 ReplicaSet 的副本数重新调高、新 ReplicaSet 的副本数调低，即可恢复到之前的版本。

为什么 MySQL 等数据库通常需要使用 StatefulSet？
: MySQL 等有状态应用存在节点角色依赖和数据依赖关系，例如主从复制中从节点需要连接主节点进行数据同步，因此要求节点具备稳定的网络身份和有序的启动顺序。StatefulSet 能保证 Pod 具有固定名称（如 mysql-0、mysql-1）、稳定的网络标识以及独立持久化存储，并按照顺序创建和删除（创建按 0→1→2，删除按 2→1→0），从而避免主节点未就绪或数据丢失等问题。而 Deployment 无法保证固定身份和有序调度，更适合无状态应用，因此数据库等有状态服务通常使用 StatefulSet。

为什么需要 Headless Service?
: Headless Service 是将 clusterIP 设置为 None 的 Service，它不会分配虚拟 IP，也不会通过 kube-proxy 做负载均衡，而是通过 DNS 直接返回后端 Pod 的真实 IP 地址。比如你用 StatefulSet 部署一个 3 节点 MySQL 集群，会生成 mysql-0、mysql-1、mysql-2 三个 Pod，并创建一个名为 mysql-service 的 Headless Service，那么集群内可以通过 mysql-0.mysql-service、mysql-1.mysql-service 等域名分别访问对应的 Pod，而不是被随机转发。这样从节点就可以明确连接主节点（例如 mysql-0.mysql-service），实现稳定的节点发现和角色通信。

DaemonSet 和 Deployment 有什么区别？
: Deployment 是按照“副本数”来管理 Pod 的控制器，你指定 replicas=3，它会在集群中调度 3 个 Pod，至于运行在哪些节点并不固定，适用于无状态业务应用；而 DaemonSet 是按照“节点数量”来运行 Pod，它会在每个符合条件的 Node 上自动创建一个 Pod 副本，新增节点会自动补齐，删除节点会自动回收，通常用于部署日志采集、监控代理或网络插件等节点级服务。因此，Deployment 关注“运行多少个实例”，而 DaemonSet 关注“每个节点都要运行一个实例”。

有哪些常见资源及其缩写？
: 内容见下表

| 资源全名               | 简写   | 常见命令示例       |
| ---------------------- | ------ | ------------------ |
| pods                   | po     | kubectl get po     |
| deployments            | deploy | kubectl get deploy |
| services               | svc    | kubectl get svc    |
| namespaces             | ns     | kubectl get ns     |
| nodes                  | no     | kubectl get no     |
| replicasets            | rs     | kubectl get rs     |
| statefulsets           | sts    | kubectl get sts    |
| daemonsets             | ds     | kubectl get ds     |
| configmaps             | cm     | kubectl get cm     |
| secrets                | secret | kubectl get secret |
| ingresses              | ing    | kubectl get ing    |
| persistentvolumeclaims | pvc    | kubectl get pvc    |
| persistentvolumes      | pv     | kubectl get pv     |

kubectl get 的输出格式参数有哪些？
: 内容见下表

| 输出参数          | 作用            | 典型使用场景           |
| ----------------- | --------------- | ---------------------- |
| -o wide           | 表格 + 更多字段 | 人工排障、查看 Node/IP |
| -o name           | 只输出资源名    | 批量操作、脚本         |
| -o yaml           | YAML 格式       | 查看/复制资源配置      |
| -o json           | JSON 格式       | 程序处理、CI/CD        |
| -o jsonpath       | 精确取字段      | 自动化脚本             |
| -o custom-columns | 自定义列        | 运维报表               |
