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

### InitContainer

InitContainer（初始化容器）是在 Pod 中主容器启动之前运行的特殊容器。它先执行完初始化工作，成功退出后，主容器才会启动。如果 InitContainer 失败，Kubernetes 会反复重启它，直到成功为止，主容器绝不会提前启动。

InitContainer 和 postStart 的区别：

- 执行顺序的确定性：InitContainer 一定在主容器启动之前执行完毕，有严格保证。而 postStart 是和主容器的 EntryPoint 几乎同时触发的，没办法保证谁先执行完。
- 能力不同：postStart 只是主容器内执行一个 hook 命令，受限于主容器的镜像环境。InitContainer 是一个独立的容器，可以用完全不同的镜像，安装不同的工具，执行更复杂的操作。
- 适用场景：postStart 适合轻量级的操作，比如发个通知。InitContainer 适合有依赖关系的初始化，比如等待数据库就绪、下载配置文件、执行数据迁移等。

## 完整示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  # ====== 初始化容器（按顺序依次执行）======
  initContainers:
    # 第一步：等待数据库就绪
    - name: wait-for-db
      image: busybox
      command:
        - sh
        - -c
        - |
          until nc -z mysql-service 3306; do
            echo "Waiting for database..."
            sleep 2
          done
          echo "Database is ready!"

    # 第二步：下载配置文件
    - name: download-config
      image: busybox
      command:
        - sh
        - -c
        - wget -O /config/app.conf http://config-server/app.conf
      volumeMounts:
        - name: config-volume
          mountPath: /config

    # 第三步：执行数据库迁移
    - name: db-migration
      image: my-app:latest
      command: ["python", "manage.py", "migrate"]
      env:
        - name: DB_HOST
          value: "mysql-service"

  # ====== 主容器（所有 initContainer 成功后才启动）======
  containers:
    - name: app
      image: my-app:latest
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: config-volume
          mountPath: /config

  volumes:
    - name: config-volume
      emptyDir: {}
```

多个 InitContainer 是严格按顺序逐个执行的，不是并行的：

```
wait-for-db（成功）→ download-config（成功）→ db-migration（成功）→ 主容器 app 启动
```

任何一个失败，后面的都不会执行，Pod 状态会显示为 `Init:0/3`、`Init:1/3` 这样的进度。

```bash
# 查看 Pod 状态可以看到 init 进度
kubectl get pod my-app
# NAME     READY   STATUS     RESTARTS   AGE
# my-app   0/1     Init:1/3   0          10s   ← 第二个 init 容器正在执行
```

常见使用场景：

- 等待依赖服务就绪是最常见的场景，确保数据库、Redis、消息队列等上游服务可用后再启动应用，避免应用启动后连接失败不断重启。
- 准备数据或配置，比如从远程下载配置文件、从 Git 仓库拉取代码、解压数据包到共享 Volume 中。
- 权限和环境准备，比如修改文件目录权限、生成证书文件、执行 sysctl 调优等需要特权操作但主容器不需要特权的场景。

InitContainer 可以使用和主容器完全不同的镜像。比如你的主容器是一个精简的 Go 应用镜像，里面什么工具都没有，但你可以用一个带有 `wget`、`git`、`mysql-client` 的 InitContainer 来做准备工作。初始化完成后这个容器就退出了，不占运行时资源。

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

Metrics Server 只保留实时数据，不存历史。它的定位就是给 `kubectl top` 和 HPA 提供当前的资源用量。如果你需要看历史趋势和监控告警，那就需要 Prometheus + Grafana 了，两者是互补的关系。

## 服务发现

### Endpoint

`kubectl get ep` 是 Kubernetes 中用来查看 Endpoints（端点） 资源的命令。`ep` 是 `endpoints` 的缩写。

Endpoints 记录了一个 Service 背后实际对应的 Pod IP 和端口。当你创建一个 Service 并通过 selector 匹配到一组 Pod 时，Kubernetes 会自动创建一个同名的 Endpoints 对象，列出所有匹配且处于 Ready 状态的 Pod 地址。

常见用途包括：

- 排查 Service 连通性问题 — 如果一个 Service 无法访问，运行 `kubectl get ep <service-name>` 可以快速确认背后是否有 Pod 被关联。如果结果是 `<none>`，说明没有 Pod 匹配到这个 Service 的 selector，问题通常出在标签不匹配或 Pod 没有正常运行。

- 确认负载均衡目标 — 可以看到流量实际会被转发到哪些 Pod IP 和端口。

示例输出大概长这样：

```
NAME         ENDPOINTS                                      AGE
my-service   10.244.0.5:8080,10.244.0.6:8080                2d
```

如果想看更详细的信息，可以加 `-o wide` 或 `-o yaml`。

### 服务解析

```
Pod A → DNS(CoreDNS) → ClusterIP → kube-proxy(维护的 iptables/IPVS 规则) → Pod B
```

具体来说：

1. Pod A 发起请求，比如访问 `http://service-b:8080`

2. CoreDNS 把 `service-b` 解析成 ClusterIP，比如 `10.96.0.100`

3. 请求发到 ClusterIP，这时候内核拦截这个包

4. kube-proxy 事先写好的 iptables/IPVS 规则做 DNAT，把目标地址从 ClusterIP 替换成某个后端 Pod 的真实 IP

5. 请求到达 Pod B

Endpoint 不是流量经过的一个环节，它是一个数据源。kube-proxy watch Endpoint 的变化，然后据此更新 iptables 规则。所以 Endpoint 是在"旁边"影响规则的，不在数据流的路径上。另外 kube-proxy 本身也不转发流量，它只是负责维护规则。真正转发流量的是内核的 iptables 或 IPVS。控制面负责把规则准备好，数据面负责按规则转发流量。流量本身不经过 kube-proxy，也不经过 Endpoint 对象。

```
数据面（流量实际走的路）：
  Pod A → CoreDNS → ClusterIP → iptables/IPVS(DNAT) → Pod B

控制面（规则怎么来的）：
  API Server → Endpoint → kube-proxy → 写入 iptables/IPVS 规则
```

### CoreDNS

它是以 Pod 的形式跑在集群里的，通常部署在 `kube-system` namespace 下，一般是 2 个副本做高可用。

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

虽然它只跑在某几个节点上，但所有 Pod 都能访问它，因为它前面有一个 ClusterIP 类型的 Service（通常是 `10.96.0.10`）。每个 Pod 的 `/etc/resolv.conf` 里都会写上这个地址作为 nameserver。

### iptables 规则

每个节点都有一份。kube-proxy 以 DaemonSet 的形式跑在每个节点上，它 watch API Server 的 Service 和 Endpoint 变化，然后在本节点的内核里写入 iptables 规则。

```bash
# 在任意节点上可以看到规则
iptables -t nat -L -n | grep <service-name>
```

所以当 Pod A 发请求时，iptables 规则的匹配和 DNAT 都发生在 Pod A 所在的那个节点上，不需要跳到别的节点去查规则。

CoreDNS 是集中式的，只跑在少数几个节点上，但通过 Service 对所有 Pod 可达。iptables 规则是分布式的，每个节点都有一份完整的副本，流量在本地就能完成转发决策。

### 外部服务

创建没有 selector 的 Service，手动维护 Endpoint，让集群内的 Pod 像访问内部服务一样访问外部服务。

假设你有一个外部的数据库跑在 `10.0.0.100:3306`，或者一个第三方 API 在 `api.example.com`。Pod 里可以直接硬编码地址去访问，但这样有几个问题：地址变了要改代码重新部署，不同环境（dev/staging/prod）地址不一样要维护多套配置，而且没法利用 Kubernetes 的 DNS 和服务发现。

通过 Service + Endpoint，Pod 只需要访问 `http://external-db:3306`，地址变了只改 Endpoint 就行，代码不用动。

1. 手动创建 Endpoint（指向固定 IP）

先创建一个没有 selector 的 Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  ports:
    - port: 3306
      targetPort: 3306
  # 注意：没有 selector
```

然后手动创建同名的 Endpoints 对象：

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db # 必须和 Service 同名
subsets:
  - addresses:
      - ip: 10.0.0.100
    ports:
      - port: 3306
```

这样 Pod 里访问 `external-db:3306` 就会被路由到 `10.0.0.100:3306`。

如果外部服务有多个实例，可以写多个 IP，Kubernetes 还会帮你做负载均衡：

```yaml
subsets:
  - addresses:
      - ip: 10.0.0.100
      - ip: 10.0.0.101
      - ip: 10.0.0.102
    ports:
      - port: 3306
```

2. ExternalName（指向域名）

如果外部服务是个域名而不是 IP，可以用 ExternalName 类型的 Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  type: ExternalName
  externalName: api.example.com
```

Pod 访问 `external-api` 时，CoreDNS 会返回一条 CNAME 记录指向 `api.example.com`。不需要创建 Endpoint。

但 ExternalName 有几个限制：不支持端口映射（Pod 必须知道目标端口），不走 kube-proxy 所以没有负载均衡，而且有些应用对 CNAME 解析有兼容性问题。

3. EndpointSlice（新版推荐）

Kubernetes 1.21 之后推荐用 EndpointSlice 代替 Endpoints，尤其是后端 IP 很多的时候性能更好：

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-db-slice
  labels:
    kubernetes.io/service-name: external-db # 关键：关联到 Service
addressType: IPv4
endpoints:
  - addresses:
      - "10.0.0.100"
  - addresses:
      - "10.0.0.101"
ports:
  - port: 3306
    protocol: TCP
```

实际场景举例

- 数据库迁移 — 数据库从集群外迁到集群内（或者反过来），Pod 的代码不用改，只需要更新 Endpoint 指向新地址，或者把无 selector 的 Service 改成有 selector 的 Service 指向集群内的 Pod。
- 多环境配置 — dev 环境指向本地数据库，staging 指向测试数据库，prod 指向生产数据库。三个环境用同一个 Service 名，只是 Endpoint 不同。
- 灰度切换 — 外部服务要换地址，先在 Endpoint 里加上新地址，等验证没问题了再删掉旧地址，整个过程 Pod 无感知。

```
Pod → CoreDNS 解析 external-db → ClusterIP
    → kube-proxy 的 iptables 规则 → 10.0.0.100:3306
```

### Ingress 匹配

```yaml
# ============================================
# 1. 按域名匹配（不同域名 → 不同服务）
# ============================================
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based
spec:
  ingressClassName: nginx
  rules:
    # api.example.com 的流量 → api-service
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080

    # web.example.com 的流量 → web-service
    - host: web.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 3000

    # admin.example.com 的流量 → admin-service
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 8081
```

```yaml
# ============================================
# 2. 按路径匹配（同一域名，不同路径 → 不同服务）
# ============================================
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          # api.example.com/users/* → user-service
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8080

          # api.example.com/orders/* → order-service
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8081

          # api.example.com/payments/* → payment-service
          - path: /payments
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 8082
```

```yaml
# ============================================
# 3. 域名 + 路径混合匹配
# ============================================
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mixed
spec:
  ingressClassName: nginx
  rules:
    # 前端站点
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 3000

    # API 按路径拆分到不同微服务
    - host: api.example.com
      http:
        paths:
          - path: /v1/users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8080
          - path: /v1/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8081
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2-service
                port:
                  number: 9090

    # 管理后台
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 8082
```

```yaml
# ============================================
# 4. 精确匹配 vs 前缀匹配
# ============================================
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: exact-vs-prefix
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          # Exact: 只匹配 /healthz 这一个路径，/healthz/detail 不匹配
          - path: /healthz
            pathType: Exact
            backend:
              service:
                name: health-check
                port:
                  number: 8080

          # Prefix: 匹配 /api 开头的所有路径
          # /api, /api/users, /api/users/123 都匹配
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080

          # 兜底：所有没匹配上的请求走这里
          - path: /
            pathType: Prefix
            backend:
              service:
                name: default-service
                port:
                  number: 8080
```

```yaml
# ============================================
# 5. 默认后端（所有不匹配的请求）
# ============================================
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: with-default-backend
spec:
  ingressClassName: nginx

  # 没有匹配到任何 rule 的请求走这里
  # 比如直接用 IP 访问、或者域名不在 rules 里
  defaultBackend:
    service:
      name: fallback-service
      port:
        number: 8080

  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

```yaml
# ============================================
# 6. TLS 终止（HTTPS）
# ============================================
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example
  annotations:
    # 配合 cert-manager 自动申请 Let's Encrypt 证书
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    # 一张证书可以覆盖多个域名
    - hosts:
        - api.example.com
        - www.example.com
      secretName: example-com-tls # 证书存在这个 Secret 里

  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 3000
```

```yaml
# ============================================
# 7. Nginx Ingress 特有的高级匹配（通过 annotation）
# ============================================
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-nginx
  annotations:
    # 正则匹配路径
    nginx.ingress.kubernetes.io/use-regex: "true"
    # 限流：每秒 10 个请求
    nginx.ingress.kubernetes.io/limit-rps: "10"
    # 重写路径：/api/users/123 → /users/123
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          # 正则：匹配 /api/ 后面的任意路径
          # /api/users/123 被 rewrite 成 /users/123 转给后端
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

## 配置

### ConfigMap

#### 创建

ConfigMap 是 Kubernetes 中用于存储非机密配置数据的资源对象，创建方式主要有以下几种：

1. 通过命令行直接创建（字面量）

```bash
kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2
```

2. 从文件创建

```bash
kubectl create configmap my-config --from-file=config.txt
# 也可以自定义 key 名
kubectl create configmap my-config --from-file=mykey=config.txt
```

3. 从目录创建

```bash
kubectl create configmap my-config --from-file=./config-dir/
```

目录下的每个文件名会成为 key，文件内容成为 value。

4. 通过 YAML 声明式创建（最常用）

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  namespace: default
data:
  # 简单键值对
  database_host: "mysql.example.com"
  database_port: "3306"
  # 也可以存储整个配置文件
  app.properties: |
    server.port=8080
    spring.datasource.url=jdbc:mysql://localhost:3306/mydb
```

然后执行：

```bash
kubectl apply -f configmap.yaml
```

#### 使用

创建好之后，可以在 Pod 中通过两种方式引用：

1. 作为环境变量注入：

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: my-config
        key: database_host
```

2. 作为 Volume 挂载成文件：

```yaml
# volumes — 定义"有什么东西可以用"
volumes:
  - name: config-volume
    configMap:
      name: my-config

# volumeMounts — 定义"把东西放到哪里"
containers:
  - name: app
    volumeMounts:
      - name: config-volume
        mountPath: /etc/config # 容器A挂到这里

  - name: sidecar
    volumeMounts:
      - name: config-volume
        mountPath: /opt/sidecar/conf # 容器B挂到另一个路径
```

常用查看命令

```bash
kubectl get configmap
kubectl describe configmap my-config
kubectl get configmap my-config -o yaml
```

#### 更新

ConfigMap（CM）的更新有多种方式，根据场景和需求选择合适的方法：

1. `kubectl edit`——直接编辑

```bash
kubectl edit configmap my-config -n my-namespace
```

会打开编辑器直接修改，保存后立即生效。适合临时调试，不推荐在生产环境使用。

2. `kubectl apply`——声明式更新（推荐）

修改 YAML 文件后 apply：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  app.properties: |
    key1=new-value1
    key2=new-value2
```

```bash
kubectl apply -f configmap.yaml
```

这是最标准的 GitOps 友好方式，配合版本控制使用。

3. `kubectl create --dry-run` + `kubectl apply`——从文件/字面值快速更新

从文件更新：

```bash
kubectl create configmap my-config \
  --from-file=app.properties=./app.properties \
  --dry-run=client -o yaml | kubectl apply -f -
```

从字面值更新：

```bash
kubectl create configmap my-config \
  --from-literal=key1=new-value1 \
  --from-literal=key2=new-value2 \
  --dry-run=client -o yaml | kubectl apply -f -
```

这个模式非常实用，可以方便地用脚本或 CI 自动化。

4. `kubectl patch`——局部更新

只改某个字段，不影响其他内容：

```bash
# JSON patch
kubectl patch configmap my-config -p '{"data":{"key1":"new-value"}}'

# 只更新一个 key，其余不变
kubectl patch configmap my-config \
  --type merge -p '{"data":{"key1":"updated"}}'
```

适合只需要改一两个 key 的场景。

5. `kubectl replace`——整体替换

```bash
kubectl replace -f configmap.yaml
```

与 `apply` 不同，`replace` 是完全替换而不是合并。如果 YAML 里少了某个 key，更新后那个 key 就会消失。需要注意这个区别。

6. Kustomize 方式

在 `kustomization.yaml` 中用 `configMapGenerator` 管理：

```yaml
configMapGenerator:
  - name: my-config
    literals:
      - key1=value1
      - key2=value2
```

Kustomize 默认会在 ConfigMap 名字后加 hash 后缀（如 `my-config-abc123`），每次内容变化名字也变，从而自动触发 Deployment 的滚动更新。这是一种优雅的不可变配置方案。

```bash
kubectl apply -k ./
```

7. Helm 方式

在 Helm chart 的 `values.yaml` 中管理配置：

```yaml
# values.yaml
config:
  key1: value1
  key2: value2
```

模板中引用：

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  {{- range $k, $v := .Values.config }}
  {{ $k }}: {{ $v | quote }}
  {{- end }}
```

升级时：

```bash
helm upgrade my-release ./my-chart -f values.yaml
```

配合 Deployment 模板中加 checksum annotation 可以自动触发滚动更新：

```yaml
annotations:
  checksum/config:
    { { include (print $.Template.BasePath "/configmap.yaml") . | sha256sum } }
```

总结一下各方式的适用场景：

- 临时调试用 `edit` 或 `patch`
- 标准流程用 `apply` + YAML 文件纳入 Git 管理
- 脚本/CI 中用 `create --dry-run | apply` 的管道模式
- 生产环境推荐 Kustomize 或 Helm，利用 hash/checksum 机制确保配置变更能被 Pod 感知到

#### 热更新

ConfigMap 的热更新（即不重启 Pod 就能让应用感知到配置变化）有几种常见方法：

1. Volume 挂载方式（Kubernetes 原生支持）

当 ConfigMap 以 Volume 形式挂载到 Pod 中时，Kubernetes 会自动更新挂载的文件内容（默认周期大约 1-2 分钟，由 kubelet 的 `syncFrequency` 控制）。但需要注意，应用本身需要能感知文件变化并重新加载，Kubernetes 只负责更新文件。另外，如果使用 `subPath` 挂载，则不会自动更新。

2. Sidecar / 文件监听工具

在 Pod 中运行一个 sidecar 容器或使用 inotify 等文件监听机制，检测到挂载文件变化后，通知主进程重新加载配置。常见做法包括用一个轻量脚本监听文件变化，然后向应用发送信号（如 SIGHUP）或调用应用的 reload 接口。

3. Reloader 类控制器

使用开源工具如 Stakater Reloader 或 wave，它们会 watch ConfigMap/Secret 的变化，自动触发关联 Deployment 的滚动更新（本质上是修改 Pod annotation 来触发重建）。这种方式严格来说是"滚动重启"而非真正的热更新，但对应用透明且可靠。

4. 应用内主动轮询

应用自身定期读取配置文件或通过 Kubernetes API 直接 watch ConfigMap 对象的变化，检测到变更后主动重新加载。Spring Cloud Kubernetes、Viper（Go）等框架都支持这种模式。

5. 环境变量方式的局限

如果 ConfigMap 是通过 `envFrom` 或 `env.valueFrom` 注入为环境变量的，更新 ConfigMap 后环境变量不会自动更新，必须重启 Pod 才能生效。这是一个常见的坑。

总结对比：

- 要做到真正的热更新（不重启），推荐 Volume 挂载 + 应用侧感知文件变化的方式。
- 如果应用本身不支持配置热加载，Reloader 触发滚动更新是最实用的方案。
- 环境变量方式无法热更新，需要避免依赖它来做动态配置。

#### 一致性

这是一个很常见的问题——本地开发环境和远程（staging/production）的环境变量保持同步。下面从不同层面介绍几种实践方案：

1. 使用 `.env` 文件 + 模板管理

维护一个 `.env.example` 或 `.env.template` 文件提交到仓库，列出所有需要的变量名（值留空或填默认值）。开发者 clone 后复制为 `.env` 并填入本地值。关键是每次远程新增变量时，同步更新模板文件，并在 CI 中检查是否遗漏。

```bash
# .env.example（提交到 git）
DATABASE_URL=
REDIS_URL=
API_KEY=
FEATURE_FLAG_X=false
```

可以写一个简单脚本来比对 `.env` 和 `.env.example` 的 key 是否一致，在启动时或 CI 中做校验。

2. 从远程同步拉取

直接从远程环境拉取变量集合到本地，避免手动维护：

- Kubernetes 场景：用脚本从 ConfigMap/Secret 导出

```bash
kubectl get configmap my-app-config -o json \
  | jq -r '.data | to_entries[] | "\(.key)=\(.value)"' > .env
```

- 云平台工具：如 `aws ssm`、`gcloud secrets`、`az keyvault` 等 CLI 可以批量导出参数到 `.env` 文件
- Doppler / Infisical / Vault 等专用工具可以一条命令拉取指定环境的所有变量

3. 集中式密钥/配置管理平台

使用专门的配置管理工具统一管理所有环境的变量：

- Doppler：`doppler run -- your-command`，自动注入对应环境的变量，本地和远程用同一个 source of truth
- HashiCorp Vault：适合敏感信息管理，通过 CLI 或 SDK 在各环境统一获取
- Infisical：开源替代方案，支持 `.env` 同步和 CLI 注入
- AWS Parameter Store / Secrets Manager：配合 SDK 或 wrapper 脚本在本地和远程统一读取

这类方案的核心思路是：变量只在一个地方定义，各环境按需拉取，从根本上消除不同步的问题。

4. CI/CD 中做校验

在流水线中加一步自动检查，确保配置完整性：

```bash
# 比对 .env.example 的 key 和远程 ConfigMap 的 key
LOCAL_KEYS=$(grep -v '^#' .env.example | cut -d= -f1 | sort)
REMOTE_KEYS=$(kubectl get cm my-app-config -o json \
  | jq -r '.data | keys[]' | sort)

diff <(echo "$LOCAL_KEYS") <(echo "$REMOTE_KEYS")
```

如果有差异就让 CI 失败，强制开发者同步更新。

5. 代码层面做防御

在应用启动时校验必要的环境变量是否齐全：

```python
# Python 示例
REQUIRED_VARS = ["DATABASE_URL", "REDIS_URL", "API_KEY"]

missing = [v for v in REQUIRED_VARS if not os.getenv(v)]
if missing:
    raise RuntimeError(f"Missing env vars: {missing}")
```

```typescript
// TypeScript 示例 - 用 zod 做校验
const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string(),
  API_KEY: z.string().min(1),
});
envSchema.parse(process.env);
```

这样即使本地漏配了变量，启动时就能发现，而不是运行时出诡异错误。

对于大多数团队，实用的做法是：用 Doppler/Infisical 之类的工具做统一管理（如果团队规模支持），或者用 `.env.example` 模板 + 同步脚本 + 启动时校验 的轻量组合。关键原则是让"变量缺失"这件事尽早暴露，而不是依赖人工记忆去同步。

### Secret

Secret 和 ConfigMap 非常相似，两者 API 结构几乎一样，核心差异在于：

- Secret 数据以 Base64 编码存储（注意 Base64 不是加密，只是编码）
- Secret 在 etcd 中可以配置静态加密（EncryptionConfiguration）
- Secret 挂载到 Pod 后存放在 tmpfs（内存文件系统）中，不会写入磁盘
- `kubectl get secret my-secret -o yaml` 看到的值是编码后的，不会直接暴露明文

#### 创建

1. 命令行直接创建

```bash
kubectl create secret generic my-secret \
  --from-literal=username=admin \
  --from-literal=password=P@ssw0rd123
```

Kubernetes 会自动帮你做 Base64 编码。

2. 从文件创建

```bash
# 比如存储 TLS 证书
kubectl create secret generic my-secret \
  --from-file=tls.crt=./server.crt \
  --from-file=tls.key=./server.key
```

3. YAML 声明式创建

这里有两种写法：

用 `data` 字段（需要自己手动 Base64 编码）：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: YWRtaW4= # echo -n "admin" | base64
  password: UEBzc3cwcmQxMjM= # echo -n "P@ssw0rd123" | base64
```

用 `stringData` 字段（直接写明文，更方便）：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  username: admin
  password: P@ssw0rd123
```

`stringData` 创建后 Kubernetes 会自动转成 Base64 存到 `data` 里，两者效果一样，`stringData` 只是写起来更方便。

#### 类型

| type                                  | 用途                     |
| ------------------------------------- | ------------------------ |
| `Opaque`                              | 通用类型，最常用（默认） |
| `kubernetes.io/tls`                   | 存储 TLS 证书和私钥      |
| `kubernetes.io/dockerconfigjson`      | 拉取私有镜像的认证信息   |
| `kubernetes.io/basic-auth`            | 基本认证（用户名/密码）  |
| `kubernetes.io/service-account-token` | ServiceAccount Token     |

例如创建 TLS 类型的 Secret：

```bash
kubectl create secret tls my-tls-secret \
  --cert=./tls.crt \
  --key=./tls.key
```

创建拉取私有镜像用的 Secret：

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass
```

#### 使用

和 ConfigMap 一样，也是两种主要方式：

1. 作为环境变量

```yaml
containers:
  - name: app
    env:
      - name: DB_USER
        valueFrom:
          secretKeyRef: # 注意这里是 secretKeyRef，不是 configMapKeyRef
            name: my-secret
            key: username
      - name: DB_PASS
        valueFrom:
          secretKeyRef:
            name: my-secret
            key: password
```

也可以一次性把所有 key 都导入为环境变量：

```yaml
containers:
  - name: app
    envFrom:
      - secretRef:
          name: my-secret
```

2. 作为 Volume 挂载

```yaml
volumes:
  - name: secret-volume
    secret:
      secretName: my-secret

containers:
  - name: app
    volumeMounts:
      - name: secret-volume
        mountPath: /etc/secrets
        readOnly: true # 敏感数据建议只读
```

挂载后每个 key 会变成 `/etc/secrets/` 下的一个文件，文件内容是解码后的明文。

3. 拉取私有镜像

```yaml
spec:
  imagePullSecrets:
    - name: my-registry-secret
  containers:
    - name: app
      image: registry.example.com/my-app:latest
```

```bash
kubectl get secret
kubectl describe secret my-secret        # 只显示 key 和大小，不显示值
kubectl get secret my-secret -o yaml      # 可以看到 Base64 编码的值
# 解码查看
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
```

#### subPath

`subPath` 解决的是一个很常见的痛点：挂载 Volume 时不想覆盖目标目录下的其他文件。假设你想把 ConfigMap 里的一个配置文件挂载到 `/etc/nginx/nginx.conf`，直觉写法是：

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /etc/nginx
```

但这样会导致 `/etc/nginx` 目录被整个替换，原来目录里的其他文件（比如 `mime.types`、`conf.d/` 等）全部消失，只剩下你 ConfigMap 里的内容。`subPath` 可以让你只挂载 Volume 中的某个文件/子目录到目标路径，而不影响目标目录中的其他内容：

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /etc/nginx/nginx.conf # 精确到文件
    subPath: nginx.conf # Volume 中的哪个 key/文件
```

这样 `/etc/nginx/` 下原有的文件完全不受影响，只是 `nginx.conf` 这一个文件被替换了。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    worker_processes 1;
    events { worker_connections 1024; }
    http { server { listen 80; } }
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  volumes:
    - name: config-volume
      configMap:
        name: nginx-config
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
```

`subPath` 不仅能挑单个文件，也能挑子目录。比如一个 PVC 里有多个目录，不同容器可以各自挂载不同子目录：

```yaml
containers:
  - name: app
    volumeMounts:
      - name: shared-data
        mountPath: /app/data
        subPath: app-data # 只挂 PVC 中的 app-data/ 子目录

  - name: logs
    volumeMounts:
      - name: shared-data
        mountPath: /var/log
        subPath: log-data # 只挂 PVC 中的 log-data/ 子目录
```

注意：使用 `subPath` 挂载的文件不会随 ConfigMap/Secret 的更新而自动热更新。普通的整目录挂载是可以自动感知更新的，但 `subPath` 不行。既然 subPath 不能热更新，那就不用 subPath，而是让原始文件变成一个软链接，指向一个可以热更新的目录。

1. ConfigMap 整目录挂载到一个单独的路径（这样可以热更新）
2. 用 initContainer 把原容器里的目标文件替换成软链接，指向那个 ConfigMap 目录里的文件

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    worker_processes 2;
    events { worker_connections 1024; }
    http { server { listen 80; } }
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  volumes:
    - name: config-volume
      configMap:
        name: nginx-config
    - name: nginx-dir
      emptyDir: {}

  initContainers:
    - name: setup
      image: nginx
      command:
        - sh
        - -c
        - |
          # 把原始 /etc/nginx 的内容拷贝到 emptyDir
          cp -aL /etc/nginx/* /mnt/nginx-dir/
          # 删除原来的 nginx.conf
          rm -f /mnt/nginx-dir/nginx.conf
          # 创建软链接，指向 configMap 挂载的目录
          ln -s /mnt/config/nginx.conf /mnt/nginx-dir/nginx.conf
      volumeMounts:
        - name: nginx-dir
          mountPath: /mnt/nginx-dir
        - name: config-volume
          mountPath: /mnt/config

  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        # 用 emptyDir（里面有软链接）替换整个 /etc/nginx
        - name: nginx-dir
          mountPath: /etc/nginx
        # ConfigMap 整目录挂载，支持热更新
        - name: config-volume
          mountPath: /mnt/config
```

流程拆解如下：

```
initContainer 阶段：
  原始 /etc/nginx/* → 拷贝到 emptyDir
  删除 emptyDir 中的 nginx.conf
  创建软链接：nginx.conf → /mnt/config/nginx.conf

运行时：
  /etc/nginx/          ← emptyDir（保留了原有的 mime.types 等文件）
  /etc/nginx/nginx.conf  ← 软链接 → /mnt/config/nginx.conf
  /mnt/config/         ← ConfigMap 整目录挂载（支持热更新）
```

当你 `kubectl edit configmap nginx-config` 修改内容后，`/mnt/config/nginx.conf` 会自动更新（Kubernetes 的整目录挂载机制），而软链接会跟着指向新内容。

这个方案解决了文件内容的热更新，但 nginx 本身不会自动 reload 配置。你还需要配合一个 sidecar 或者用 `inotifywait` 之类的工具监听文件变化，然后执行 `nginx -s reload`。所以完整方案通常是：软链接解决文件更新 + sidecar 解决进程 reload。

## 存储

### Volumes

#### HostPath

在 Kubernetes 中，`hostPath` 是一种 Volume 类型，它将宿主节点（Node）文件系统上的某个文件或目录挂载到 Pod 中。

简单来说，Pod 通过 `hostPath` 可以直接访问它所运行的那台物理机或虚拟机上的文件系统路径。

常见用途：

- 运行需要访问 Docker 内部机制的容器（如挂载 `/var/lib/docker`）
- 运行 cAdvisor 等监控工具（需要访问 `/sys`）
- 让容器检查某个 hostPath 是否存在，以判断运行环境

基本示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: my-volume
  volumes:
    - name: my-volume
      hostPath:
        path: /mnt/data # 宿主机上的路径
        type: DirectoryOrCreate # 可选的类型检查
```

`type` 字段的可选值：

- 空字符串（默认）：不做任何检查
- `DirectoryOrCreate`：路径不存在则自动创建目录
- `Directory`：路径必须已存在且是目录
- `FileOrCreate`：不存在则创建文件
- `File`：必须已存在且是文件
- `Socket`、`CharDevice`、`BlockDevice`：对应类型的设备/套接字

需要注意的风险：

- 不同 Node 上同一路径的内容可能不同，Pod 调度到不同节点会导致行为不一致。
- 它直接暴露了宿主机文件系统，存在安全隐患（如果挂载了 `/` 等敏感路径，容器可能影响整个节点）。
- 一般不推荐在生产环境中使用，除非有明确的运维需求（比如 DaemonSet 类的系统组件）。

如果只是需要持久化存储，推荐使用 `PersistentVolume` / `PersistentVolumeClaim` 等更安全的方案。

#### EmptyDir

`emptyDir` 是 Kubernetes 中一种临时性 Volume，它在 Pod 被调度到某个节点时自动创建，初始内容为空，Pod 被删除时随之销毁。

核心特点：

- 生命周期与 Pod 绑定——Pod 存在它就在，Pod 删除它就没了。
- 同一个 Pod 内的多个容器可以共享同一个 `emptyDir`，实现容器间数据交换。
- 容器崩溃重启不会丢失 `emptyDir` 中的数据（因为 Pod 还在），但 Pod 被删除或重新调度就会丢失。

典型用途：

- 容器间共享数据：比如一个 sidecar 容器写日志，另一个容器读取并上传。
- 临时缓存/暂存空间：排序、计算等中间结果的临时存放。
- 检查点恢复：长时间计算中保存中间状态，容器崩溃后可以从中恢复。

基本示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "echo hello > /shared/data.txt && sleep 3600"]
      volumeMounts:
        - mountPath: /shared
          name: shared-vol
    - name: reader
      image: busybox
      command: ["sh", "-c", "cat /shared/data.txt && sleep 3600"]
      volumeMounts:
        - mountPath: /shared
          name: shared-vol
  volumes:
    - name: shared-vol
      emptyDir: {}
```

上面两个容器通过 `emptyDir` 共享了 `/shared` 目录。

可选配置：

- `medium: "Memory"`：将 `emptyDir` 存放在内存（tmpfs）中，速度极快，但占用 Pod 的内存限额，节点重启数据也会丢失。
- `sizeLimit`：限制占用的空间大小。

```yaml
emptyDir:
  medium: Memory
  sizeLimit: 128Mi
```

和 `hostPath` 的关键区别： `emptyDir` 不依赖节点上的特定路径，更安全、更可移植，适合大多数临时存储场景。

### NFS

在 K8s 中使用 NFS 主要有两种方式，从简单到规范：

1. 直接在 Pod 中挂载 NFS

最简单，适合测试或临时使用：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: nfs-vol
  volumes:
    - name: nfs-vol
      nfs:
        server: 192.168.1.100 # NFS 服务器地址
        path: /shared/data # NFS 导出的路径
        readOnly: false
```

缺点是每个 Pod 都要硬编码 NFS 地址和路径，不好维护。

2. 通过 PV + PVC 使用（推荐）

把 NFS 的细节封装到 PV 里，Pod 只需要声明 PVC，解耦存储和应用。

- 创建 PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany # NFS 天然支持多节点读写
  nfs:
    server: 192.168.1.100
    path: /shared/data
```

- 创建 PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

- Pod 中引用 PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: nfs-vol
  volumes:
    - name: nfs-vol
      persistentVolumeClaim:
        claimName: nfs-pvc
```

这样多个 Pod（哪怕在不同节点）都可以挂载同一个 `nfs-pvc`，共享同一份数据。

K8s 节点上需要安装 NFS 客户端，否则挂载会失败：

```bash
# Ubuntu / Debian
sudo apt install nfs-common

# CentOS / RHEL
sudo yum install nfs-utils
```

比较：

```markdown
直接挂载 NFS → 简单快速，但 NFS 地址散落在各个 Pod 定义中
PV + PVC → 多一层抽象，但解耦了存储细节，换存储后端时 Pod 不用改
```

生产环境建议用 PV/PVC 的方式。如果 NFS 卷很多，还可以部署一个 NFS Provisioner 来实现动态创建 PV，不用每次手动建。

### PV 和 PVC

#### 生命周期

1. 构建（Provisioning）

- 静态构建：管理员手动预先创建 PV，指定容量、访问模式、存储后端等参数，PV 创建后处于 Available 状态等待被绑定。
- 动态构建：当用户提交 PVC 且没有匹配的静态 PV 时，集群根据 PVC 中指定的 StorageClass 自动创建 PV。这是生产环境中更常用的方式。

2. 绑定（Binding）

PVC 提交后，控制器会寻找满足条件（容量、访问模式、StorageClass 等）的 PV 进行绑定。PV 和 PVC 是一对一的关系，绑定后 PV 状态变为 Bound。如果找不到匹配的 PV 且没有合适的 StorageClass，PVC 会一直处于 Pending 状态。

3. 使用（Using）

Pod 通过在 volumes 中引用 PVC 来使用存储。此时 PV 正在被 Pod 挂载使用。Kubernetes 提供了存储保护机制——如果用户尝试删除一个正在被 Pod 使用的 PVC，系统不会立即删除，而是延迟到 Pod 不再使用该 PVC 时才执行。

4. 回收策略（Reclaiming）

当 PVC 被删除后，PV 的处理方式取决于回收策略：

- Retain（保留）：PV 进入 Released 状态，数据保留，但不能被新的 PVC 绑定。管理员需要手动清理数据并决定是删除还是重新使用该 PV。这是最安全的策略。
- Delete（删除）：自动删除 PV 及其对应的后端存储资源（如 AWS EBS、GCE PD 等）。动态构建的 PV 默认使用这个策略。
- Recycle（回收）：对卷执行基本的数据清除（相当于 `rm -rf /thevolume/*`），然后让 PV 重新变为 Available。不过这个策略已经被废弃了，官方建议用动态构建替代。

#### 静态构建案例

下面是一个完整的例子，使用 NFS 作为存储后端，依次创建 PV、PVC，然后在 Pod 中挂载使用。

```yaml
# ============================================
# 1. 创建 PersistentVolume (PV)
# ============================================
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-nfs-pv
  labels:
    type: nfs
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany # 多节点读写
  persistentVolumeReclaimPolicy: Retain # 回收策略：保留
  storageClassName: nfs-storage
  nfs:
    server: 192.168.1.100 # NFS 服务器地址
    path: /data/nfs-share # NFS 共享目录

---
# ============================================
# 2. 创建 PersistentVolumeClaim (PVC)
# ============================================
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-nfs-pvc
spec:
  accessModes:
    - ReadWriteMany # 必须与 PV 的 accessModes 匹配
  resources:
    requests:
      storage: 2Gi # 请求 2Gi，PV 有 5Gi，满足条件
  storageClassName: nfs-storage # 必须与 PV 的 storageClassName 一致

---
# ============================================
# 3. 创建 Pod，挂载 PVC
# ============================================
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
spec:
  containers:
    - name: my-app
      image: nginx:1.24
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nfs-data # 引用下面 volumes 中定义的名称
          mountPath: /usr/share/nginx/html # 容器内挂载路径
  volumes:
    - name: nfs-data
      persistentVolumeClaim:
        claimName: my-nfs-pvc # 引用上面创建的 PVC 名称
```

验证

```bash
# 查看 PV 状态，应为 Bound
kubectl get pv my-nfs-pv

# 查看 PVC 状态，应为 Bound
kubectl get pvc my-nfs-pvc

# 查看 Pod 是否正常运行
kubectl get pod my-app-pod

# 进入 Pod 验证挂载
kubectl exec -it my-app-pod -- df -h /usr/share/nginx/html
```

PVC 能成功绑定 PV 需要满足几个条件：storageClassName 一致、accessModes 匹配、PV 的容量 ≥ PVC 请求的容量。在这个例子中，PV 提供 5Gi，PVC 请求 2Gi，storageClassName 都是 `nfs-storage`，访问模式都是 `ReadWriteMany`，所以能成功绑定。

#### 动态构建案例

如果你想用动态构建的方式，就不需要手动创建 PV，只需要提前创建好 StorageClass，然后 PVC 引用该 StorageClass，集群会自动创建 PV。

```yaml
# ============================================
# 1. 创建 StorageClass（管理员一次性配置）
# ============================================
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs # 存储供应商（根据环境修改）
parameters:
  type: gp3 # AWS EBS 卷类型
  fsType: ext4 # 文件系统类型
reclaimPolicy: Delete # PVC 删除时自动删除 PV 和后端存储
allowVolumeExpansion: true # 允许后续扩容
volumeBindingMode: WaitForFirstConsumer # 等 Pod 调度后再创建卷（推荐）

---
# ============================================
# 2. 创建 PVC（自动触发 PV 的动态创建）
# ============================================
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
    - ReadWriteOnce # 单节点读写，EBS 典型模式
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-storage # 引用上面的 StorageClass

---
# ============================================
# 3. 创建 Deployment，挂载 PVC
# ============================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-web-app
  template:
    metadata:
      labels:
        app: my-web-app
    spec:
      containers:
        - name: app
          image: nginx:1.24
          ports:
            - containerPort: 80
          volumeMounts:
            - name: app-storage
              mountPath: /var/www/data # 容器内挂载路径
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
      volumes:
        - name: app-storage
          persistentVolumeClaim:
            claimName: app-data-pvc # 引用上面的 PVC

---
# ============================================
# 补充：使用 StatefulSet 的场景（每个副本独立 PV）
# ============================================
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-database
spec:
  serviceName: my-database
  replicas: 3
  selector:
    matchLabels:
      app: my-database
  template:
    metadata:
      labels:
        app: my-database
    spec:
      containers:
        - name: db
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "my-secret-pw" # 生产环境请用 Secret
          volumeMounts:
            - name: db-data
              mountPath: /var/lib/mysql
  # volumeClaimTemplates 会为每个副本自动创建独立的 PVC
  volumeClaimTemplates:
    - metadata:
        name: db-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
        storageClassName: fast-storage
```

静态构建需要管理员手动创建 PV，而动态构建只需要配置好 StorageClass，之后创建 PVC 时集群会自动创建匹配的 PV，不需要手动干预。

```bash
# 部署资源
kubectl apply -f dynamic-example.yaml

# 查看 StorageClass
kubectl get sc fast-storage

# 查看 PVC — 状态应为 Bound
kubectl get pvc app-data-pvc

# 查看自动创建的 PV
kubectl get pv
# 会看到一个名称类似 pvc-xxxx-xxxx 的 PV，由集群自动创建

# 如果用了 StatefulSet，查看每个副本的 PVC
kubectl get pvc
# 会看到 db-data-my-database-0、db-data-my-database-1、db-data-my-database-2
```

- 关于 provisioner：不同环境使用不同的 provisioner，比如 AWS 用 `kubernetes.io/aws-ebs` 或 `ebs.csi.aws.com`，阿里云用 `diskplugin.csi.alibabacloud.com`，本地集群可以用 NFS provisioner 等。
- 关于 volumeBindingMode：设置为 `WaitForFirstConsumer` 时，PV 会等到 Pod 被调度到具体节点后才创建，这样可以保证存储和 Pod 在同一个可用区，避免跨区挂载失败。
- 关于 StatefulSet：示例中补充了 StatefulSet 的 `volumeClaimTemplates`，这是有状态应用（如数据库）的常见用法，每个副本会自动获得一个独立的 PVC 和 PV，副本扩缩容时互不影响。

#### StorageClass

Provisioner 是动态构建的核心组件，它负责监听 PVC 的创建事件，然后自动创建对应的 PV 和后端存储。

Kubernetes 本身不内置 NFS 的 Provisioner，所以需要额外部署一个。常用的是 `nfs-subdir-external-provisioner`，它的工作原理是：当有新的 PVC 创建时，它会在 NFS 服务器上自动创建一个子目录作为 PV 的存储路径。

nfs-provisioner 是具体的 NFS 制备器部署，通常以 Deployment 的方式运行在集群中。它需要知道 NFS 服务器地址和共享路径：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-provisioner
  template:
    metadata:
      labels:
        app: nfs-provisioner
    spec:
      serviceAccountName: nfs-provisioner-sa
      containers:
        - name: provisioner
          image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.1.100 # NFS 服务器地址
            - name: NFS_PATH
              value: /data/nfs-share # NFS 共享目录
          volumeMounts:
            - name: nfs-root
              mountPath: /persistentvolumes
      volumes:
        - name: nfs-root
          nfs:
            server: 192.168.1.100
            path: /data/nfs-share
```

Provisioner 需要操作 PV、PVC 等集群资源，所以必须配置相应的权限：

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-provisioner-sa

---
# ClusterRole — 定义权限
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-provisioner-role
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]

---
# ClusterRoleBinding — 绑定权限到 ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nfs-provisioner-binding
subjects:
  - kind: ServiceAccount
    name: nfs-provisioner-sa
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-provisioner-role
  apiGroup: rbac.authorization.k8s.io
```

创建 StorageClass 并指向我们部署的 NFS Provisioner：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-dynamic
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # 与 Provisioner 中设置的名称一致
parameters:
  archiveOnDelete: "true" # 删除 PVC 时保留数据（重命名目录而非删除）
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

PVC 一直 Pending 通常有这几个原因：

- Provisioner 没有正常运行：`kubectl get pods` 检查 provisioner Pod 状态
- StorageClass 名称不匹配：PVC 引用的 storageClassName 和 StorageClass 的 name 不一致
- RBAC 权限不足：provisioner 没有权限创建 PV，查看 provisioner Pod 的日志 `kubectl logs <provisioner-pod>`
- NFS 服务端问题：NFS 服务未启动、共享目录不存在、防火墙拦截等

最后创建 PVC 和 Pod 来验证整个流程是否正常：

```yaml
# 测试 PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs-dynamic # 引用上面的 StorageClass

---
# 测试 Pod
apiVersion: v1
kind: Pod
metadata:
  name: test-nfs-pod
spec:
  containers:
    - name: test
      image: busybox
      command:
        [
          "sh",
          "-c",
          "echo 'NFS is working!' > /mnt/test.txt && cat /mnt/test.txt && sleep 3600",
        ]
      volumeMounts:
        - name: nfs-vol
          mountPath: /mnt
  volumes:
    - name: nfs-vol
      persistentVolumeClaim:
        claimName: test-nfs-pvc
```

验证命令：

```bash
# 检查 PVC 是否 Bound
kubectl get pvc test-nfs-pvc

# 检查自动创建的 PV
kubectl get pv

# 检查 Pod 日志，应输出 "NFS is working!"
kubectl logs test-nfs-pod

# 去 NFS 服务器上查看，会发现自动创建了子目录
ls /data/nfs-share/
```

整个动态制备的链路是这样的：PVC 创建 → Kubernetes 发现没有匹配的 PV → 根据 storageClassName 找到 StorageClass → StorageClass 中的 provisioner 指向 NFS Provisioner → Provisioner 在 NFS 上创建子目录并创建 PV → PV 与 PVC 自动绑定 → Pod 挂载使用。管理员只需要一次性部署好 Provisioner、RBAC 和 StorageClass，之后开发人员只需要创建 PVC 就可以了。

## 认证与授权

### 认证

#### 账户类型

Kubernetes 中有两种完全不同的账户类型：

- User Accounts（用户账户）：这是给人用的，比如管理员、开发人员通过 kubectl 操作集群时使用的身份。User Account 有几个特点：它是全局性的，不属于任何 namespace；Kubernetes 本身不管理 User Account，没有 User 这个 API 对象，通常由外部系统管理（比如证书、OIDC、LDAP 等）。常见的认证方式包括：
  - X509 客户端证书：最常见的方式，kubeconfig 里的 client-certificate 就是这个
  - Bearer Token：静态 token 文件或 OIDC token
  - Webhook Token：对接外部认证系统

```bash
# 例如用证书方式创建用户
openssl genrsa -out dev-user.key 2048
openssl req -new -key dev-user.key -out dev-user.csr -subj "/CN=dev-user/O=dev-group"
# 然后提交 CSR 给 Kubernetes CA 签发
```

- Service Accounts（服务账户）：这是给 Pod 里运行的程序用的，用于程序访问 API Server 时的身份认证。Service Account 是 namespace 级别的 Kubernetes 资源，可以通过 API 直接创建和管理。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
```

#### 自动化管理

1. Service Account Admission Controller

这是一个准入控制器，作用于 Pod 创建阶段。当你创建 Pod 时，如果没有指定 serviceAccountName，它会自动注入 default Service Account。同时它还会自动挂载 SA 对应的 token 到 Pod 中（路径是 `/var/run/secrets/kubernetes.io/serviceaccount/`）。

简单说就是：你创建 Pod 时什么都不配，这个控制器会自动帮你把身份信息塞进去。

```yaml
# Pod 创建时如果没写 serviceAccountName
# Admission Controller 自动补上：
spec:
  serviceAccountName: default # 自动添加
  volumes: # 自动挂载 token
    - name: kube-api-access-xxxxx
      projected:
        sources:
          - serviceAccountToken:
              path: token
          - configMap:
              name: kube-root-ca.crt
          - downwardAPI:
              items:
                - path: namespace
```

2. Token Controller

运行在 Controller Manager 中，负责管理 Service Account 的 token。在早期版本中它会为每个 SA 创建一个永久的 Secret token。在 1.22+ 之后，Kubernetes 转向了 TokenRequest API，由 kubelet 为每个 Pod 申请一个有时效的、有受众限制的 token，更安全。

Token Controller 现在主要做两件事：当 SA 创建时确保相关资源就绪；当 SA 被删除时清理关联的 Secret。

3. Service Account Controller

这个也运行在 Controller Manager 中，职责很简单：确保每个 namespace 都有一个名为 default 的 Service Account。如果你新建了一个 namespace，它会自动创建 default SA；如果 default SA 被误删了，它也会自动重建。

```bash
# 创建一个新 namespace 后自动会有 default SA
kubectl create namespace test
kubectl get sa -n test
# NAME      SECRETS   AGE
# default   0         1s    ← 自动创建的
```

#### 工作流程

把这些串起来，一个 Pod 从创建到访问 API Server 的认证流程是这样的：

1. Service Account Controller 确保每个 namespace 都有 default SA
2. 用户创建 Pod，可以指定 SA，也可以不指定
3. Admission Controller 检查 Pod，如果没指定 SA 就注入 default，并配置 token 挂载
4. kubelet 通过 TokenRequest API 获取一个短时效 token 挂载到 Pod 中
5. Pod 内的程序读取 `/var/run/secrets/kubernetes.io/serviceaccount/token`，用这个 token 访问 API Server
6. API Server 验证 token，确认身份后再进入授权（RBAC）阶段

```bash
# 在 Pod 内部可以这样验证
cat /var/run/secrets/kubernetes.io/serviceaccount/token
cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# 用 token 访问 API Server
curl -s --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  https://kubernetes.default.svc/api/v1/namespaces/default/pods
```

生产环境中有几个要注意的点：不要让 Pod 使用 default SA，而是为每个应用创建专用的 SA 并配合 RBAC 授权最小权限；如果 Pod 不需要访问 API Server，可以设置 `automountServiceAccountToken: false` 禁止自动挂载 token，减少攻击面。

### 授权

RBAC 的逻辑其实很简单：先定义"能做什么"（Role），再定义"谁能做"（Binding），把权限和主体绑定起来。核心原则就是最小权限：只给需要的资源和操作的权限，优先用 Role + RoleBinding 限定在 namespace 内，只有真正需要集群级别操作时才用 ClusterRole + ClusterRoleBinding。

#### 资源

1. Role（角色）—— namespace 级别的权限定义

Role 定义了在某个特定 namespace 内可以对哪些资源执行哪些操作。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev # 只在 dev namespace 生效
rules:
  - apiGroups: [""] # 核心 API 组，包含 Pod、Service 等
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
```

2. ClusterRole（集群角色）—— 集群级别的权限定义

跟 Role 的区别是 ClusterRole 不受 namespace 限制，它可以定义对集群级资源（如 Node、PV、Namespace 本身）的权限，也可以用于跨所有 namespace 的资源访问。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-node-reader # 没有 namespace 字段
rules:
  - apiGroups: [""]
    resources: ["nodes"] # Node 是集群级资源，只能用 ClusterRole
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list"]
```

Kubernetes 内置了一些常用的 ClusterRole，比如 `cluster-admin`（最高权限）、`view`（只读）、`edit`（编辑）、`admin`（namespace 管理员）。

```bash
# 查看内置的 ClusterRole
kubectl get clusterrole
```

3. RoleBinding（角色绑定）—— namespace 级别的绑定

RoleBinding 把 Role 或 ClusterRole 绑定到具体的主体（User、Group、ServiceAccount），让他们在指定 namespace 内获得权限。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: dev
subjects: # "谁"
  - kind: User # 可以是 User、Group、ServiceAccount
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount # 也可以绑定 SA
    name: my-app-sa
    namespace: dev
roleRef: # "获得什么权限"
  kind: Role
  name: pod-reader # 引用上面定义的 Role
  apiGroup: rbac.authorization.k8s.io
```

一个重要的点：RoleBinding 也可以引用 ClusterRole，这样做的效果是让 ClusterRole 的权限限定在某个 namespace 内。这在实际中非常常用——定义一次 ClusterRole，然后通过 RoleBinding 在不同 namespace 中复用。

```yaml
# RoleBinding 引用 ClusterRole，权限被限定在 dev namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-view-binding
  namespace: dev
subjects:
  - kind: User
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole # 引用的是 ClusterRole
  name: view # Kubernetes 内置的只读角色
  apiGroup: rbac.authorization.k8s.io
```

4. ClusterRoleBinding（集群角色绑定）—— 集群级别的绑定

把 ClusterRole 绑定到主体，权限在整个集群生效。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-binding # 没有 namespace
subjects:
  - kind: User
    name: admin-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin # 集群最高权限
  apiGroup: rbac.authorization.k8s.io
```

#### 关系总结

简单来说就是一个二维组合：

|          | namespace 级别 | 集群级别           |
| -------- | -------------- | ------------------ |
| 定义权限 | Role           | ClusterRole        |
| 绑定权限 | RoleBinding    | ClusterRoleBinding |

其中 RoleBinding 可以引用 Role 或 ClusterRole，而 ClusterRoleBinding 只能引用 ClusterRole。

#### 场景示例

一个典型的多团队权限管理方案，每个团队只能操作自己 namespace 内的资源，互不干扰：

```yaml
# 1. 创建一个通用的开发者 ClusterRole（定义一次）
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"] # Secret 只给读权限

---
# 2. 团队 A 在自己的 namespace 中使用这个角色
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-dev-binding
  namespace: team-a
subjects:
  - kind: Group
    name: team-a-developers # 绑定到组，方便管理
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer
  apiGroup: rbac.authorization.k8s.io

---
# 3. 团队 B 在自己的 namespace 中复用同一个角色
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-b-dev-binding
  namespace: team-b
subjects:
  - kind: Group
    name: team-b-developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

排查权限：

```bash
# 查看某用户是否有某个权限
kubectl auth can-i get pods --as dev-user -n dev
# yes / no

# 查看某用户的所有权限
kubectl auth can-i --list --as dev-user -n dev

# 查看某个 namespace 的所有 RoleBinding
kubectl get rolebinding -n dev

# 查看某个 ClusterRoleBinding 的详情
kubectl describe clusterrolebinding admin-binding
```

#### User详解

Kubernetes 的 User 和 Linux 的用户完全不是一回事。Linux 用户是操作系统层面的，存在 `/etc/passwd` 里，用来登录服务器。Kubernetes 的 User 是 API Server 层面的身份标识，用来控制"谁能通过 API 操作集群资源"。

你可以用 root 账号 SSH 登录到 Master 节点，但如果你的 kubeconfig 里没有合法的凭证，kubectl 照样无法操作集群。反过来，你不需要登录任何服务器，只要有 kubeconfig 文件，在自己的笔记本上就能远程管理集群。

Kubernetes 没有 User 这个 API 对象，你无法通过 `kubectl create user` 来创建用户。User 只是一个身份标识字符串，来源于你提供的认证凭证。

最常见的方式是 X509 客户端证书，证书里的 CN（Common Name）字段就是用户名，O（Organization）字段就是用户组：

```bash
# 生成证书时，CN=zhangsan 就是用户名，O=team-a 就是组
openssl req -new -key zhangsan.key -out zhangsan.csr \
  -subj "/CN=zhangsan/O=team-a"
```

API Server 收到请求后从证书里提取出 `zhangsan` 这个字符串，然后去 RBAC 里查这个字符串有什么权限。就这么简单——User 只是一个名字，不是一个存储在集群中的实体。

假设团队 A 有三个开发人员，实际操作流程是这样的：

```bash
# 1. 为每个人生成证书（通常由集群管理员操作）

# 张三
openssl genrsa -out zhangsan.key 2048
openssl req -new -key zhangsan.key -out zhangsan.csr \
  -subj "/CN=zhangsan/O=team-a"

# 李四
openssl genrsa -out lisi.key 2048
openssl req -new -key lisi.key -out lisi.csr \
  -subj "/CN=lisi/O=team-a"

# 王五
openssl genrsa -out wangwu.key 2048
openssl req -new -key wangwu.key -out wangwu.csr \
  -subj "/CN=wangwu/O=team-a"

# 2. 用集群 CA 签发证书
# （通过 Kubernetes CSR API 或直接用 CA 密钥签发）
```

然后管理员为每个人生成各自的 kubeconfig 文件：

```yaml
# zhangsan 的 kubeconfig
apiVersion: v1
kind: Config
clusters:
  - cluster:
      server: https://api-server:6443
      certificate-authority-data: <CA证书>
    name: my-cluster
users:
  - name: zhangsan
    user:
      client-certificate-data: <张三的证书> # 身份就在这里
      client-key-data: <张三的私钥>
contexts:
  - context:
      cluster: my-cluster
      user: zhangsan
      namespace: team-a
    name: zhangsan-context
current-context: zhangsan-context
```

张三拿到这个 kubeconfig 文件后，在自己电脑上就能操作集群了：

```bash
export KUBECONFIG=~/zhangsan-kubeconfig.yaml
kubectl get pods    # API Server 识别出身份是 zhangsan
```

权限绑定可以绑定个人，也可以绑定组。因为三个人的证书里都有 `O=team-a`，可以直接按组授权，不需要一个一个绑定：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-binding
  namespace: team-a
subjects:
  # 按组绑定，team-a 所有人都有权限
  - kind: Group
    name: team-a # 对应证书中的 O 字段
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

如果某个人需要额外权限，可以单独再加一个绑定：

```yaml
subjects:
  - kind: User
    name: zhangsan # 对应证书中的 CN 字段
    apiGroup: rbac.authorization.k8s.io
```

Kubernetes 的 User 就是"拿着合法凭证访问 API Server 的人"，它不存储在集群里，而是从证书、Token 等认证信息中提取出来的一个身份字符串。和登录 Linux 服务器的用户没有关系，和能不能操作集群有关系。
