---
short_title: Advanced
---

# Kubernetes Advanced

## Pod

### 配置文件

下面是一个简单的 Pod 部署示例，用 `kubectl create -f pod.yaml` 启动。

```yaml
apiVersion: v1 # API 版本，Pod 固定为 v1
kind: Pod # 资源类型：Pod（也可以是 Deployment、StatefulSet 等）

metadata:
  name: nginx-demo # Pod 名称（同一 namespace 内必须唯一）
  namespace: default # 命名空间（默认 default）
  labels: # 标签（用于被 Service / Deployment 选择）
    type: app
    test: "1.0.0"

spec:
  containers: # Pod 中的容器列表（一个 Pod 可以有多个容器）
    - name: nginx # 容器名称
      image: nginx:1.7.9 # 镜像名:版本
      imagePullPolicy: IfNotPresent
      # 可选值：
      # Always          每次都拉取镜像
      # IfNotPresent    本地有就不拉（默认）
      # Never           从不拉取（必须本地已有）

      command: # 覆盖镜像的 ENTRYPOINT
        - nginx
        - -g
        - "daemon off;"
      # 如果只想改参数，用 args 字段

      workingDir: /usr/share/nginx/html # 容器启动后的工作目录（可选）

      ports:
        - name: http # 端口名称（可选，但建议写，方便 Service 引用）
          containerPort: 80 # 容器内部端口（必填）
          protocol: TCP # TCP / UDP / SCTP（默认 TCP）

      env: # 环境变量（可选）
        - name: JVM_OPTS
          value: "-Xms128m -Xmx128m"
      # 也可以使用 valueFrom：
      # valueFrom:
      #   configMapKeyRef:
      #   secretKeyRef:

      resources: # 资源限制（强烈建议生产环境必须设置）
        requests: # 最少需要多少资源（调度依据）
          cpu: "100m" # 100m = 0.1 核
          memory: "128Mi"
        limits: # 最多可以使用多少资源（硬限制）
          cpu: "200m"
          memory: "256Mi"

  restartPolicy: OnFailure
  # 可选值：
  # Always      默认值（Deployment 必须是 Always）
  # OnFailure   失败才重启（常用于 Job）
  # Never       从不重启（常用于一次性任务）
```

### 探针

StartupProbe
: StartupProbe 用来判断容器是否启动完成，主要解决“启动慢”的问题；在它成功之前，Liveness 和 Readiness 都不会生效，如果启动失败超过阈值，容器会被重启，适用于需要较长初始化时间的应用（如加载大模型、数据预热等）。

下面的例子最多检测 30 × 10 = 300 秒，在 startupProbe 成功之前，liveness 和 readiness 不会执行

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  failureThreshold: 30 # 失败 30 次才算失败
  periodSeconds: 10 # 每 10 秒检测一次
```

LivenessProbe
: LivenessProbe 用来判断容器是否还活着；如果探测失败达到阈值，Kubernetes 会重启容器，用于处理应用“假死”或“卡死”但进程还在的情况。

下面的例子，连续 3 次失败，容器会被重启

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5 # 启动 5 秒后开始检测
  periodSeconds: 10 # 每 10 秒检测一次
  failureThreshold: 3 # 连续 3 次失败才重启
```

ReadinessProbe
: ReadinessProbe 用来判断容器是否准备好对外提供服务；如果探测失败，Pod 不会被重启，但会从 Service 的负载均衡列表中移除，不再接收流量。

下面的例子探测失败时，Pod 不会重启，但会从 Service 负载均衡中移除

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

ExecAction
: ExecAction 是在容器内部执行一条命令来判断应用是否正常，如果命令返回码为 0 表示成功，非 0 表示失败；它适用于需要通过脚本或特定逻辑判断健康状态的场景，比如检查进程是否存在、文件是否生成、数据库是否连通等。

TCPSocketAction
: TCPSocketAction 是通过尝试连接容器的某个 TCP 端口来判断是否健康，如果端口能成功建立 TCP 连接则认为成功，否则认为失败；它适用于只要端口能连上就代表服务存活的简单场景，比如 Web 服务、数据库服务等。

HTTPGetAction
: HTTPGetAction 是通过向容器的指定 URL 发送 HTTP 请求来判断是否健康，如果返回状态码在 200~399 之间则认为成功；它适用于 Web 应用，可以通过访问 /health、/ready 等接口来精确判断应用是否可用。

### 生命周期

1. 创建与调度：当执行 kubectl apply 后，Pod 对象被提交到 apiserver 并存入 etcd，随后 scheduler 选择一个合适的 Node 进行调度；节点上的 kubelet 接收到指令后开始真正创建 Pod。
2. Pause 容器创建：在 Node 上，kubelet 首先创建 Pause 容器，用来持有 Pod 的网络命名空间（Pod IP 就绑定在这里），Pod 内所有其他容器共享这个网络与端口空间。
3. 初始化阶段（Init Containers）：如果定义了 initContainers，会按顺序串行执行，每个容器必须成功退出（exit code = 0）才会继续下一个；全部完成后才会启动主容器，常用于做初始化准备工作。
4. 主容器启动：Init 容器完成后，开始启动 Pod 内的主容器（containers），执行 command/args，同时如果定义了 postStart 钩子也会触发，此时应用真正运行。
5. 三类探针：在 Pod 启动后，kubelet 会先执行 startupProbe，如果成功则执行 livenessProbe，最后执行 readinessProbe。
6. 运行阶段（Running）：当主容器成功启动且 ReadinessProbe 通过后，Pod 进入 Running 状态，开始正常对外提供服务，健康检查会持续周期性执行。
7. 终止阶段（Termination）：当 Pod 被删除或驱逐时，kubelet 会先执行 preStop 钩子，然后发送 SIGTERM 信号，等待优雅退出时间（terminationGracePeriodSeconds），若超时未退出则发送 SIGKILL 强制结束。
8. 最终状态：当所有容器退出后，Pod 会进入 Succeeded 或 Failed 状态，生命周期结束；如果由控制器（如 Deployment）管理，控制器可能会创建新的 Pod 进行替换。

Pod 被删除时的退出流程
: 当执行删除操作后，Pod 会先进入 Terminating 状态，同时会从 Service 的 Endpoint 中移除该 Pod 的 IP（不再接收新流量）；随后 kubelet 执行容器的 preStop 钩子，然后向容器发送 SIGTERM 信号进行优雅关闭，在等待 terminationGracePeriodSeconds 时间后如果仍未退出则强制终止，整个过程实现的是“先摘流量，再优雅下线”的安全退出机制。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prestop-demo
spec:
  terminationGracePeriodSeconds: 30 # 优雅退出最大等待时间（默认30秒）
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80

      lifecycle:
        preStop:
          exec:
            command:
              - sh
              - -c
              - "echo 'Pod is terminating...' && sleep 10"
```

## 资源调度

### 标签与选择器

label的创建方式：

- 通过文件：spec.metadata.labels
- 通过命令创建：kubectl label pod <pod_name> env=prod
- 通过命令修改：kubectl label pod <pod_name> env=prod --overwrite
- 查看所有标签：kubectl get pod --show-labels

通过选择器找到label：

- kubectl get po -l env=prod
- kubectl get po -l 'version in (1.0.0, 2.0.0)'
- kubectl get po -l version!=1.0.0,env=prod

### Deployment

- kubectl create deploy <deployment_name> --image=<image_name> --replicas=\<replicas> --port=\<port> --labels="env=prod,version=1.0.0"
- kubectl get deploy <deployment_name> -o yaml
- kubectl get deploy <deployment_name> -o jsonpath='{.spec.progressDeadlineSeconds}'
- kubectl set image deploy <deployment_name> <container_name>=<image_name>
- kubectl describe deploy <deployment_name>
- kubectl rollout status deploy <deployment_name>
- kubectl rollout history deploy <deployment_name>
- kubectl rollout undo deploy <deployment_name>

```yaml
apiVersion: apps/v1 # 指定使用的 Kubernetes API 版本，apps/v1 用于部署对象
kind: Deployment # 资源类型为“部署（Deployment）”，用于管理无状态应用
metadata: # 资源的元数据，包含名称、标签等
  labels: # 为 Deployment 自身设置的标签
    app: nginx-deploy # 标签键值对：应用名为 nginx-deploy
  name: nginx-deploy # Deployment 的唯一名称
  namespace: default # 部署所在的命名空间，默认为 default
spec: # 部署的具体规格定义
  progressDeadlineSeconds: 600 # 超时新 Pod 没 Ready，Deployment 被标记为 failed
  replicas: 1 # 期望运行的 Pod 副本数量
  revisionHistoryLimit: 10 # 保留的历史版本 RS 数量，用于回滚（默认为 10）
  selector: # 选择器，定义 Deployment 如何查找并管理它的 Pod
    matchLabels: # 匹配具有以下标签的 Pod
      app: nginx-deploy # 必须匹配 app: nginx-deploy 标签
  strategy: # 更新策略
    rollingUpdate: # 滚动更新的具体配置
      maxSurge: 25% # 更新时允许超过期望副本数的最大比例
      maxUnavailable: 25% # 更新时允许处于不可用状态的最大副本比例
    type: RollingUpdate # 更新类型为“滚动更新（RollingUpdate）”
  template: # 定义 Pod 的模板
    metadata: # Pod 的元数据
      labels: # Pod 的标签，必须与上面的 selector.matchLabels 对应
        app: nginx-deploy # Pod 标签：app: nginx-deploy
    spec: # Pod 内部容器的详细定义
      containers: # 容器列表
        - image: nginx:1.7.9 # 使用的 Docker 镜像版本
          imagePullPolicy: IfNotPresent # 镜像拉取策略：如果本地已有则不拉取
          name: nginx # 容器的名称
      restartPolicy: Always # 容器重启策略：总是重启
      terminationGracePeriodSeconds: 30 # 容器接收到关闭信号后的优雅停机等待时间
```

一些发版相关的操作：

- kubectl apply -f deploy.yaml
- kubectl annotate deploy backend kubernetes.io/change-cause="release v1.1.0"
- kubectl rollout history deploy backend
- kubectl rollout undo deploy backend --to-revision=1 # 回到 v1.0.0

扩缩容操作：

- 改 yaml 文件里的 replicas 然后 kubectl apply，配置有记录
- kubectl scale deploy nginx-deploy --replicas=5
- 配置 HPA（水平自动扩缩），根据 CPU/内存自动调整

Deployment 中 selector 和 template.labels 为什么要分开写，直接自动推导不行吗？
: 因为两者职责不同，selector 是 Deployment 认领 Pod 的过滤条件，template.labels 是 Pod 创建时被打上的身份标签，分开写是为了允许 selector 作为 template.labels 的子集。如果自动推导，selector 就只能等于 template.labels 的全集，你就无法做到"用少数几个标签匹配 Pod，同时让 Pod 携带更多标签给 Service 或监控组件使用"这种灵活组合。此外，selector 在 Deployment 创建后是不可变的（强制修改会报 field is immutable），这是为了防止改 selector 后旧 Pod 因匹配不到新条件而变成无人管理的僵尸 Pod。想换 label 体系只能删除重建 Deployment。分开写看似冗余，实际上是 Kubernetes 在简洁性和灵活性之间有意做出的取舍。

kubectl rollout undo 的 --to-revision 参数是什么意思？
: --to-revision 指定的是 Kubernetes 内部的 REVISION 序号，即 kubectl rollout history 输出里左边那列的数字，和应用自身的版本号（如 x.y.z）没有关系。回滚前应先执行 rollout history 确认目标序号，再用 --to-revision 指定回滚到哪个版本。

kubectl apply 和 kubectl create 有什么区别？
: create 只能用于创建资源，如果资源已存在会直接报错；apply 则是声明式的，资源不存在时创建，已存在时更新，不需要关心当前状态。实际开发中持续迭代场景下基本只用 apply，create 适合一次性创建且不需要后续更新的场景。

annotation 的 key 为什么用域名格式作为前缀？
: annotation 的 key 格式为 <前缀>/<名称>，前缀采用域名格式是为了避免不同组织或工具之间的命名冲突。比如 kubernetes.io/change-cause 是官方定义的，prometheus.io/scrape 是 Prometheus 的，域名格式天然全局唯一，和 Java 包名用反向域名是同一个道理，和实际网络请求没有任何关系。

### StatefulSet

Headless Service（clusterIP: None）— 给 StatefulSet 提供稳定的网络标识，不做负载均衡。访问方式是直接指定 Pod DNS，比如 web-0.nginx。

```yaml
# ============================================================
# 1. Headless Service（StatefulSet 必须搭配 Headless Service）
# ============================================================
apiVersion: v1
kind: Service
metadata:
  name: nginx # Service 名称，StatefulSet 通过 serviceName 引用它
  labels:
    app: nginx
spec:
  ports:
    - port: 80 # Service 暴露的端口
      name: web # 端口命名，方便引用
  clusterIP:
    None # 设为 None 表示 Headless Service，不分配 ClusterIP
    # 这样每个 Pod 会获得独立的 DNS 记录：
    # <pod-name>.<service-name>.<namespace>.svc.cluster.local
  selector:
    app: nginx # 匹配带有 app=nginx 标签的 Pod

---
# ============================================================
# 2. StatefulSet（有状态应用的核心控制器）
# ============================================================
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web # StatefulSet 的名称
spec:
  serviceName: nginx # 关联上面的 Headless Service，用于生成 Pod 的网络标识
  replicas: 3 # 副本数，会创建 web-0、web-1、web-2 三个 Pod
  selector:
    matchLabels:
      app: nginx # 必须与 template.metadata.labels 一致

  # ---------- Pod 模板 ----------
  template:
    metadata:
      labels:
        app: nginx # Pod 的标签，供 Service 和 selector 匹配
    spec:
      terminationGracePeriodSeconds: 10 # Pod 终止前的优雅等待时间（秒）
      containers:
        - name: nginx # 容器名称
          image: nginx:1.25 # 使用的镜像及版本
          ports:
            - containerPort: 80 # 容器监听的端口
              name: web # 端口名，与 Service 的 port.name 对应
          volumeMounts:
            - name: www-storage # 引用下方 volumeClaimTemplates 中定义的名称
              mountPath: /usr/share/nginx/html # 将 PVC 挂载到 nginx 默认的静态文件目录

  # ---------- Pod 管理策略（可选） ----------
  podManagementPolicy:
    OrderedReady # 默认值：按序创建/删除 Pod（0→1→2）
    # 另一个选项是 Parallel（并行创建）

  # ---------- 更新策略 ----------
  updateStrategy:
    type: RollingUpdate # 滚动更新（默认）
    rollingUpdate:
      partition:
        0 # 分区更新：只有序号 >= partition 的 Pod 会被更新
        # 设为 0 表示全部更新；设为 2 则只更新 web-2

  # ---------- PVC 模板（核心：为每个 Pod 自动创建独立的 PVC） ----------
  volumeClaimTemplates:
    - metadata:
        name:
          www-storage # PVC 名称前缀，实际创建的 PVC 名为：
          # www-storage-web-0、www-storage-web-1、www-storage-web-2
      spec:
        accessModes:
          - ReadWriteOnce # 访问模式：单节点读写
            # 其他选项：ReadWriteMany（多节点读写）、ReadOnlyMany（多节点只读）
        storageClassName:
          standard # 指定 StorageClass，按集群实际情况填写
          # 如果不指定，会使用集群默认的 StorageClass
        resources:
          requests:
            storage: 1Gi # 每个 Pod 申请 1Gi 的持久化存储空间
```

普通 Service（ClusterIP / NodePort / LoadBalancer）— 由 kube-proxy 通过 iptables 或 IPVS 把流量轮询分发到后端 Pod，这才是负载均衡。两个 Service 可以同时存在，用同一个 selector，各司其职。StatefulSet 场景下这种"一个 Headless + 一个普通 Service"的组合很常见。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      name: web
  # 不写 clusterIP: None，默认就是 ClusterIP 类型
  # K8s 会分配一个虚拟 IP，由 kube-proxy 做负载均衡
```

很常用，BusyBox 基本是 K8s 集群调试的瑞士军刀。它镜像极小（只有几 MB），但内置了大量常用的 Linux 工具。快速起一个临时 Pod 做网络测试：

```bash
kubectl run debug --rm -it --image=busybox -- sh
```

进去之后就可以用 `nslookup`、`wget`、`ping`、`telnet` 等工具来验证 DNS 解析、Service 连通性等：

```sh
# 返回 web-0 这个 Pod 的实际 IP 地址
nslookup web-0.nginx.default.svc.cluster.local
# 向 web-1 这个 Pod 的 nginx 发起 HTTP 请求，把返回的页面内容输出到终端
wget -qO- http://web-1.nginx
```

```{tip}
集群内部的 DNS 和网络是隔离的，只在集群内部生效。web-0.nginx.default.svc.cluster.local 这个域名是 K8s 内部的 CoreDNS 管理的，Master 节点的宿主机并不走这套 DNS 解析。所以你在 Master 终端上直接 nslookup 或 wget，大概率解析不了或者访问不到。启动一个 BusyBox Pod，相当于把自己放进集群网络里面，这样才能用集群内部的 DNS 和网络去验证 Service、Pod 之间的连通性。
```

BusyBox 的工具都是精简版，有些参数不支持。如果需要更完整的调试能力，还有几个常用的替代选择：

- nicolaka/netshoot — 网络调试专用，自带 `curl`、`tcpdump`、`dig`、`iftop` 等完整工具
- curlimages/curl — 只需要测 HTTP 的话够用了
- alpine — 比 BusyBox 稍大，但有 `apk` 包管理器，缺什么可以随时装

怎么更新生产环境的statefulset的镜像版本？
: 见下

| 方式      | 命令                                                              | 适用场景                            | 可追溯性          |
| --------- | ----------------------------------------------------------------- | ----------------------------------- | ----------------- |
| set image | `kubectl set image sts/web nginx=nginx:1.26`                      | 快速更新单个镜像，临时调试          | 低 - 无记录       |
| patch     | `kubectl patch sts web --type='json' -p='[{"op":"replace",...}]'` | 脚本化批量修改，CI 中精准改某个字段 | 低 - 需自行记录   |
| apply -f  | `kubectl apply -f statefulset.yaml`                               | 生产环境标准流程，配合 Git + CI/CD  | 高 - Git 全程追溯 |

StatefulSet 默认是 RollingUpdate 策略，会从最大序号的 Pod 开始逐个更新（web-2 → web-1 → web-0），和 Deployment 的顺序相反。如果想先灰度验证，可以利用之前 YAML 里的 partition 参数，比如设成 2，就只更新 web-2，确认没问题再逐步放开。

什么是 OnDelete 更新？
: `OnDelete` 是 StatefulSet 的一种更新策略，意思是你手动删除 Pod 时，它才会用新配置重建。K8s 不会自动帮你滚动更新。RollingUpdate（默认）改了镜像版本后，K8s 自动从最大序号开始逐个重建 Pod，而 OnDelete 你改了镜像版本后，什么都不会发生。只有当你手动 `kubectl delete pod web-2` 时，新建出来的 `web-2` 才会用新镜像。这种策略适合对更新顺序有严格要求的场景，比如数据库集群。你可能需要先更新从节点、验证数据同步没问题、再更新主节点，整个过程不想让 K8s 自动操作，而是自己完全掌控节点的更新顺序和时机。

什么是联级删除？
: 删除 StatefulSet 时默认是 --cascade=background，会连 StatefulSet 带 Pod 一起全删掉。--cascade=orphan 表示只删 StatefulSet 本身，但保留它管理的 Pod。删除之后，web-0、web-1、web-2 这些 Pod 还在正常运行，只是变成了"孤儿 Pod"，没有任何控制器管理它们了。比如你想替换 StatefulSet 的不可变字段。有些字段（如 selector）创建后不能改，你只能删了重建。用 --cascade=orphan 删除后，Pod 不受影响继续跑，然后你 apply 一个新的同名 StatefulSet，它会重新接管那些 Pod，整个过程服务不中断。

```bash
kubectl delete statefulset web --cascade=orphan
kubectl apply -f statefulset.yaml
```

### DaemonSet

Fluentd DaemonSet + Elasticsearch 日志采集例子。每个节点上的容器写日志到 /var/log/containers/ → DaemonSet 保证每个节点都有一个 Fluentd Pod → Fluentd 通过 hostPath 挂载读取宿主机日志 → 附加 K8s 元数据（Pod 名、Namespace 等）→ 批量写入 Elasticsearch → 最后用 Kibana 查询和可视化。

- 为什么用 DaemonSet 而不是 Deployment：DaemonSet 保证每个节点恰好跑一个 Pod，新节点加入集群时也会自动部署，不需要手动干预。日志采集天然是"每个节点都需要"的场景。
- tolerations 的作用：默认 Master 节点有污点，普通 Pod 不会调度上去。加了 tolerations 后 Fluentd 也能采集 Master 节点的日志。
- hostPath 挂载：这是 DaemonSet 日志采集的核心思路——把宿主机的日志目录直接挂进 Pod，Fluentd 就能读到同一台机器上所有容器的日志。

```yaml
# ============================================================
# 1. 专用 Namespace（日志组件统一放这里，便于管理）
# ============================================================
apiVersion: v1
kind: Namespace
metadata:
  name: logging

---
# ============================================================
# 2. ServiceAccount + RBAC（Fluentd 需要读取集群中 Pod 的元数据）
# ============================================================
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: logging

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces"] # Fluentd 需要读取 Pod 和 Namespace 信息
    verbs: ["get", "list", "watch"] # 用于给日志附加 K8s 元数据（Pod名、Namespace等）

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluentd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluentd
subjects:
  - kind: ServiceAccount
    name: fluentd
    namespace: logging

---
# ============================================================
# 3. Fluentd 配置（通过 ConfigMap 注入）
# ============================================================
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    # ---- 输入：采集容器日志 ----
    <source>
      @type tail                                          # 以 tail 方式持续读取日志文件
      path /var/log/containers/*.log                      # K8s 容器日志的标准路径
      pos_file /var/log/fluentd-containers.log.pos        # 记录读取位置，重启后不会重复采集
      tag kubernetes.*                                    # 给日志打标签，后续用于路由和过滤
      read_from_head true                                 # 首次启动时从文件头开始读
      <parse>
        @type json                                        # 容器日志格式通常是 JSON
        time_key time                                     # 从 JSON 中提取时间字段
        time_format %Y-%m-%dT%H:%M:%S.%NZ                # 时间格式
      </parse>
    </source>

    # ---- 过滤：附加 K8s 元数据 ----
    <filter kubernetes.>
      @type kubernetes_metadata                           # 自动给每条日志附加 Pod名、Namespace
    </filter>                                             # 容器名、标签等信息，方便在 ES 中检索

    # ---- 输出：发送到 Elasticsearch ----
    <match >
      @type elasticsearch                                 # 输出到 ES
      host elasticsearch.logging.svc.cluster.local        # ES 的 Service 地址
      port 9200                                           # ES 默认端口
      logstash_format true                                # 使用 logstash 格式的索引名
      logstash_prefix k8s-logs                            # 最终索引名如 k8s-logs-2026.03.12
      include_tag_key true                                # 在日志中包含 tag 字段
      flush_interval 5s                                   # 每 5 秒批量写入一次 ES
      <buffer>
        @type file                                        # 缓冲区写到文件，防止数据丢失
        path /var/log/fluentd-buffers/kubernetes.buffer   # 缓冲文件路径
        flush_mode interval                               # 按时间间隔刷新
        retry_type exponential_backoff                    # 写入失败时指数退避重试
        flush_thread_count 2                              # 2 个线程并发写入
        chunk_limit_size 2M                               # 每个缓冲块最大 2MB
        queue_limit_length 8                              # 缓冲队列最多 8 个块
        retry_max_interval 30                             # 最大重试间隔 30 秒
      </buffer>
    </match>

---
# ============================================================
# 4. DaemonSet（核心：确保每个节点都跑一个 Fluentd Pod）
# ============================================================
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd # 使用上面创建的 ServiceAccount
      tolerations:
        - key: node-role.kubernetes.io/control-plane # 容忍 Master 节点的污点
          effect: NoSchedule # 这样 Master 节点也会部署 Fluentd
        - key: node-role.kubernetes.io/master # 兼容旧版本 K8s 的污点 key
          effect: NoSchedule
      containers:
        - name: fluentd
          image:
            fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch7-1
            # 官方镜像，已内置 ES 和 K8s 元数据插件
          env:
            - name: FLUENT_ELASTICSEARCH_HOST # 也可以通过环境变量覆盖 ES 地址
              value: "elasticsearch.logging.svc.cluster.local"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
          resources:
            limits:
              memory: 512Mi # 内存上限，防止 OOM
            requests:
              cpu: 100m # 最低 CPU 请求
              memory: 200Mi # 最低内存请求
          volumeMounts:
            - name: varlog
              mountPath: /var/log # 挂载宿主机的 /var/log
            - name: dockercontainers
              mountPath: /var/lib/docker/containers # 挂载 Docker 容器日志目录
              readOnly: true # 只读，Fluentd 只需要读日志
            - name: fluentd-config
              mountPath: /fluentd/etc/fluent.conf # 挂载自定义配置文件
              subPath: fluent.conf # 只挂载单个文件，不覆盖整个目录
      volumes:
        - name: varlog
          hostPath:
            path: /var/log # 宿主机路径，所有容器日志都在这里
        - name: dockercontainers
          hostPath:
            path: /var/lib/docker/containers # Docker 容器原始日志路径
        - name: fluentd-config
          configMap:
            name: fluentd-config # 引用上面的 ConfigMap
```

DaemonSet 的调度策略中，nodeSelector 最简单直接。给节点打标签：`kubectl label node node-1 disk=ssd`。简单的键值匹配，够用就用这个：

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd # 只部署到带有 disk=ssd 标签的节点
```

nodeSelector 的增强版为 nodeAffinity，比 `nodeSelector` 强在支持更丰富的表达式，比如"部署到 zone 在 us-east-1a 或 1b 的节点"，还有 `preferredDuringScheduling` 软约束（尽量满足但不强制）。

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution: # 硬性要求，不满足就不调度
            nodeSelectorTerms:
              - matchExpressions:
                  - key: zone
                    operator: In # 支持 In、NotIn、Exists、Gt、Lt 等操作符
                    values:
                      - us-east-1a
                      - us-east-1b
```

tolerations 解决的是"我能不能去"。节点如果有 taint，DaemonSet Pod 默认会被拒绝，必须加上对应的 toleration 才能调度上去。就像之前 Fluentd 例子里容忍 Master 节点的污点一样。

```yaml
tolerations:
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule # 容忍 gpu=true 的污点节点
```

常见的污点主要分几类：

- K8s 自动添加的，这些是集群自己打上去的，你不需要手动操作：
  - `node-role.kubernetes.io/control-plane:NoSchedule` — Master 节点，防止业务 Pod 跑上去
  - `node.kubernetes.io/not-ready:NoSchedule` — 节点状态异常，还没就绪
  - `node.kubernetes.io/unreachable:NoSchedule` — 节点失联了
  - `node.kubernetes.io/disk-pressure:NoSchedule` — 磁盘快满了
  - `node.kubernetes.io/memory-pressure:NoSchedule` — 内存不够了
  - `node.kubernetes.io/pid-pressure:NoSchedule` — 进程数快耗尽
  - `node.kubernetes.io/network-unavailable:NoSchedule` — 网络没配好
  - `node.kubernetes.io/unschedulable:NoSchedule` — 节点被 `kubectl cordon` 手动标记为不可调度
- 运维手动添加的常见场景

```bash
# GPU 节点，只允许 AI 任务跑
kubectl taint nodes gpu-node-1 gpu=true:NoSchedule

# 专属节点，只给某个团队用
kubectl taint nodes team-node dedicated=team-a:NoSchedule

# 节点维护中，现有 Pod 驱逐，新 Pod 不调度
kubectl taint nodes node-3 maintenance=true:NoExecute
```

effect 的三种级别：

- NoSchedule — 新 Pod 不会调度上来，已有的 Pod 不受影响
- PreferNoSchedule — 尽量不调度，但资源紧张时还是会放上来，是个软约束
- NoExecute — 最严格，新 Pod 不调度，已经在跑的 Pod 也会被驱逐。节点维护时常用这个

### HPA

- CPU + 内存百分比）是最常见的，大部分 Web 应用用这个就够了。前提是 Pod 必须设置了 resources.requests，否则算不出利用率百分比。

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web # 要自动伸缩的目标 Deployment
  minReplicas: 2 # 最少保留 2 个 Pod
  maxReplicas: 10 # 最多扩到 10 个 Pod
  metrics:
    # ---- CPU 利用率 ----
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization:
            70 # 所有 Pod 平均 CPU 超过 70% 就扩容
            # 注意：Pod 必须设置了 resources.requests.cpu 才能算利用率

    # ---- 内存利用率 ----
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization:
            80 # 平均内存超过 80% 就扩容
            # 内存指标要注意：缩容不一定及时，因为内存不像 CPU 会自然下降
```

- 绝对值适合你明确知道单个 Pod 的承载上限，比如"一个 Pod 最多用 500m CPU"，不想依赖 requests 的配置比例。

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker
  minReplicas: 1
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: AverageValue
          averageValue:
            500m # 每个 Pod 平均 CPU 超过 500m（0.5核）就扩容
            # 和 Utilization 的区别：这个不依赖 requests 的设置

    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 256Mi # 每个 Pod 平均内存超过 256Mi 就扩容
```

- 自定义 + 外部指标，是生产环境最实用的，因为 CPU 和内存其实是滞后指标——等 CPU 飙高了说明请求已经在排队了。用 QPS 或队列深度这种业务指标来驱动扩容，响应更及时。不过需要额外部署 Prometheus + Prometheus Adapter 来提供指标。

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 50
  metrics:
    # ---- 每个 Pod 的 QPS（自定义指标） ----
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second # 指标名，需要 Prometheus Adapter 注册
        target:
          type: AverageValue
          averageValue: "1000" # 每个 Pod 平均 QPS 超过 1000 就扩容

    # ---- 消息队列深度（外部指标，如 RabbitMQ / Kafka 积压量） ----
    - type: External
      external:
        metric:
          # https://www.rabbitmq.com/docs/prometheus
          name: queue_messages_ready # 来自外部监控系统的指标
          selector:
            matchLabels:
              queue: order-processing # 指定具体哪个队列
        target:
          type: Value
          value: "5000" # 队列积压超过 5000 条就扩容

  # ---- 扩缩容行为控制（防止抖动） ----
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30 # 扩容冷却期 30 秒，避免瞬间暴增
      policies:
        - type: Percent
          value: 100 # 单次最多扩一倍
          periodSeconds: 60
        - type: Pods
          value: 5 # 或者单次最多加 5 个 Pod
          periodSeconds: 60
      selectPolicy: Max # 取两个策略中扩得更多的那个

    scaleDown:
      stabilizationWindowSeconds: 300 # 缩容冷却期 5 分钟，避免流量波动导致反复缩扩
      policies:
        - type: Percent
          value: 10 # 单次最多缩 10%，缩容要保守
          periodSeconds: 60
```

Metrics Server 可以监控 Pod 和 Node 实时情况

```bash
# 安装
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

如果是测试环境（比如 minikube、kind），可能还需要加一个启动参数跳过 TLS 验证，否则会连不上 kubelet：

```bash
# 给 Metrics Server 的 Deployment 加一个参数
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

装好等一两分钟就能用了：

```bash
kubectl top nodes          # 看每个节点的 CPU 和内存用量
kubectl top pods           # 看每个 Pod 的 CPU 和内存用量
kubectl top pods -A        # 所有 Namespace
```

另外补充一下，Metrics Server 只保留实时数据，不存历史。它的定位就是给 `kubectl top` 和 HPA 提供当前的资源用量。如果你需要看历史趋势和监控告警，那就需要 Prometheus + Grafana 了，两者是互补的关系。
