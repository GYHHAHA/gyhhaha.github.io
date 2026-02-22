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
