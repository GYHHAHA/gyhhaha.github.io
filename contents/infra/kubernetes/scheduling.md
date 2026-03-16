---
short_title: 高级调度
---

# 高级调度

## CronJob

CronJob 本质上就是一个定时任务调度器，它按照你设定的时间表周期性地创建 Job，Job 再创建 Pod 去执行具体任务。三者的关系是：CronJob → Job → Pod。典型使用场景包括：数据库定时备份、日志清理、定时报表生成、数据同步等。

### Cron 表达式

CronJob 使用标准的 Cron 表达式来定义执行时间，格式是五个字段：

```
┌───────────── 分钟 (0-59)
│ ┌───────────── 小时 (0-23)
│ │ ┌───────────── 日 (1-31)
│ │ │ ┌───────────── 月 (1-12)
│ │ │ │ ┌───────────── 星期 (0-6，0=周日)
│ │ │ │ │
*  *  *  *  *
```

几个常用的例子：

```yaml
"0 2 * * *"       # 每天凌晨 2:00
"*/5 * * * *"     # 每 5 分钟
"0 0 * * 0"       # 每周日午夜
"0 8 1 * *"       # 每月 1 号早上 8:00
"30 22 * * 1-5"   # 每周一到周五晚上 22:30
```

### 数据库备份

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: production
spec:
  schedule: "0 2 * * *" # 每天凌晨 2 点执行
  concurrencyPolicy: Forbid # 不允许并发执行
  successfulJobsHistoryLimit: 3 # 保留最近 3 个成功 Job
  failedJobsHistoryLimit: 3 # 保留最近 3 个失败 Job
  startingDeadlineSeconds: 600 # 错过调度 10 分钟内仍可补执行
  suspend: false # 是否暂停调度

  jobTemplate:
    spec:
      backoffLimit: 3 # 失败最多重试 3 次
      activeDeadlineSeconds: 3600 # 单次 Job 最长运行 1 小时
      template:
        spec:
          restartPolicy: OnFailure # CronJob 只能用 OnFailure 或 Never
          containers:
            - name: backup
              image: mysql:8.0
              command:
                - /bin/sh
                - -c
                - |
                  mysqldump -h $DB_HOST -u$DB_USER -p$DB_PASS \
                    --all-databases > /backup/db-$(date +%Y%m%d-%H%M%S).sql
                  echo "Backup completed"
              env:
                - name: DB_HOST
                  value: "mysql-service"
                - name: DB_USER
                  valueFrom:
                    secretKeyRef:
                      name: db-credentials
                      key: username
                - name: DB_PASS
                  valueFrom:
                    secretKeyRef:
                      name: db-credentials
                      key: password
              volumeMounts:
                - name: backup-volume
                  mountPath: /backup
          volumes:
            - name: backup-volume
              persistentVolumeClaim:
                claimName: backup-pvc
```

### 参数详解

- concurrencyPolicy（并发策略）：这个参数控制当上一次 Job 还没执行完时，新的调度时间又到了怎么办：

```yaml
concurrencyPolicy: Allow    # 默认值，允许并发，多个 Job 同时跑
concurrencyPolicy: Forbid   # 跳过本次调度，等上一个跑完
concurrencyPolicy: Replace  # 终止上一个 Job，启动新的
```

数据库备份这种场景一般用 `Forbid`，避免两个备份任务同时写同一个目录。日志清理这种幂等操作可以用 `Allow`。

- startingDeadlineSeconds：如果因为某些原因（比如集群负载高、Controller Manager 重启等）错过了调度时间，在这个时间窗口内仍然会补执行。超过这个时间就直接跳过。如果不设置，错过的任务可能会累积一起执行。

- successfulJobsHistoryLimit 和 failedJobsHistoryLimit：CronJob 每次执行会创建一个新的 Job 对象，时间长了会堆积很多。这两个参数控制保留多少历史 Job，方便排查问题的同时不至于资源泛滥。

### 常用命令

```bash
# 查看 CronJob
kubectl get cronjob -n production

# 查看 CronJob 创建的 Job 历史
kubectl get jobs -n production

# 查看某次执行的 Pod 日志
kubectl logs job/db-backup-28473920 -n production

# 手动触发一次执行（不用等到调度时间）
kubectl create job --from=cronjob/db-backup manual-backup -n production

# 临时暂停调度
kubectl patch cronjob db-backup -n production -p '{"spec":{"suspend":true}}'

# 恢复调度
kubectl patch cronjob db-backup -n production -p '{"spec":{"suspend":false}}'
```

### 生产问题

有几个生产中容易遇到的问题。

- 时区问题：CronJob 默认使用集群控制平面的时区（通常是 UTC），在 1.27+ 版本中可以通过 `timeZone` 字段指定时区：

```yaml
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Shanghai" # 1.27+ 支持
```

- restartPolicy 限制：CronJob 的 Pod 只能设置 `OnFailure` 或 `Never`，不能用 `Always`，因为任务执行完就应该退出。

- Job 堆积：如果任务执行时间比调度间隔还长，又没设置 `concurrencyPolicy`，会不断创建新 Job 导致资源耗尽。所以生产环境务必设置好并发策略和 `activeDeadlineSeconds`。

## 污点与容忍

### 污点（Taint）

污点是打在 Node 上的标记，作用是排斥 Pod，告诉调度器"不要随便把 Pod 调度到我这里来"。

#### 污点的格式

```
key=value:effect
```

三个部分：key 是标识名，value 是值（可以为空），effect 是排斥策略。

#### 三种 Effect

NoSchedule：新的 Pod 不会调度到这个节点上，但已经在运行的 Pod 不受影响。这是最常用的。

PreferNoSchedule：尽量不调度到这个节点，但如果没有其他可用节点，还是可以调度过来。是一种软限制。

NoExecute：最严格，不仅新 Pod 不会调度过来，已经在运行的、没有对应容忍的 Pod 也会被驱逐。

#### 操作命令

```bash
# 添加污点
kubectl taint nodes node-1 gpu=true:NoSchedule
kubectl taint nodes node-2 env=production:NoExecute
kubectl taint nodes node-3 disk=ssd:PreferNoSchedule

# value 可以为空
kubectl taint nodes node-1 special-node:NoSchedule

# 查看节点的污点
kubectl describe node node-1 | grep Taints

# 删除污点（末尾加减号）
kubectl taint nodes node-1 gpu=true:NoSchedule-
kubectl taint nodes node-1 special-node:NoSchedule-
```

#### Master 节点的污点

你可能注意过，默认情况下 Pod 不会调度到 Master 节点，就是因为 kubeadm 初始化时自动给 Master 打了污点：

```bash
kubectl describe node master | grep Taints
# Taints: node-role.kubernetes.io/control-plane:NoSchedule
```

如果是测试环境想让 Master 也跑业务 Pod，可以去掉这个污点：

```bash
kubectl taint nodes master node-role.kubernetes.io/control-plane:NoSchedule-
```

### 容忍（Toleration）

容忍是配在 Pod 上的，表示"我能接受某个节点上的污点，允许把我调度过去"。

注意：容忍不是说"我一定要去那个节点"，只是说"我不排斥那个节点"。最终去不去还要看调度器的综合判断。

#### 基本用法

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-app
spec:
  tolerations:
    # 精确匹配：key、value、effect 都必须一致
    - key: "gpu"
      operator: "Equal" # 默认值
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: app
      image: nvidia/cuda:12.0-base
```

#### 两种匹配方式

Equal（精确匹配）：key、value、effect 必须完全一致

```yaml
tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
# 只匹配 gpu=true:NoSchedule 这一个污点
```

Exists（只匹配 key）：只要 key 存在就行，不关心 value

```yaml
tolerations:
  - key: "gpu"
    operator: "Exists"
    effect: "NoSchedule"
# 匹配所有 key 为 gpu 且 effect 为 NoSchedule 的污点
# 不管 value 是 true、false 还是空
```

#### 特殊写法

```yaml
# 容忍某个 key 的所有 effect
tolerations:
  - key: "gpu"
    operator: "Exists"
    # 不写 effect，表示匹配所有 effect

# 容忍所有污点（万能容忍）
tolerations:
  - operator: "Exists"
    # 不写 key 也不写 effect，匹配一切污点
```

DaemonSet 中的系统组件（比如 kube-proxy、网络插件）通常就配置了万能容忍，确保每个节点都能运行。

#### tolerationSeconds

当节点有 `NoExecute` 污点时，配合 `tolerationSeconds` 可以控制 Pod 在被驱逐前还能停留多久：

```yaml
tolerations:
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300 # 节点不可达后，Pod 再坚持 5 分钟才被驱逐
```

如果不设置 `tolerationSeconds`，表示永远容忍，不会被驱逐。

### 实战场景

假设集群中有三类节点：普通节点、GPU 节点、生产专用节点。

```bash
# 给 GPU 节点打污点
kubectl taint nodes gpu-node-1 gpu=true:NoSchedule
kubectl taint nodes gpu-node-2 gpu=true:NoSchedule

# 给生产节点打污点
kubectl taint nodes prod-node-1 env=production:NoExecute
kubectl taint nodes prod-node-2 env=production:NoExecute
```

场景一：GPU 训练任务，必须跑在 GPU 节点

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-training
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-training
  template:
    metadata:
      labels:
        app: ml-training
    spec:
      tolerations:
        - key: "gpu"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      nodeSelector: # 容忍 + nodeSelector 配合使用
        node-type: gpu # 容忍让它"能去"，nodeSelector 让它"必须去"
      containers:
        - name: training
          image: ml-training:latest
          resources:
            limits:
              nvidia.com/gpu: 1
```

这里有个关键点：单独配容忍并不能保证 Pod 一定去 GPU 节点，它只是不排斥。要确保一定调度到 GPU 节点，需要配合 `nodeSelector` 或者亲和性（Affinity）一起使用。

场景二：生产应用，只跑在生产节点

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: prod-app
  template:
    metadata:
      labels:
        app: prod-app
    spec:
      tolerations:
        - key: "env"
          operator: "Equal"
          value: "production"
          effect: "NoExecute"
          tolerationSeconds: 600 # 节点出问题时给 10 分钟优雅退出
      nodeSelector:
        env: production
      containers:
        - name: app
          image: prod-app:latest
```

因为生产节点用了 `NoExecute`，普通的 Pod 即使被手动调度上去也会被立即驱逐，只有配了容忍的生产应用才能存活。

场景三：监控 DaemonSet，每个节点都要跑

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
spec:
  selector:
    matchLabels:
      app: monitor
  template:
    metadata:
      labels:
        app: monitor
    spec:
      tolerations:
        - operator: "Exists" # 万能容忍，不管什么污点都能上
      containers:
        - name: monitor
          image: prometheus/node-exporter:latest
```

整个判断流程是这样的：调度器看一个节点时，先检查节点上有哪些污点，再检查 Pod 上有哪些容忍。如果节点的每一个污点都能被 Pod 的容忍匹配到，Pod 就可以调度到这个节点；只要有一个污点没有被容忍，Pod 就不能去（NoSchedule）、尽量不去（PreferNoSchedule）、或者去了也会被赶走（NoExecute）。

这个大纲结构很清晰，内容也是准确的。我补充一些细节来详细讲解。

## 亲和性

前面讲的污点和容忍是从 Node 角度"排斥" Pod，而亲和性反过来，是从 Pod 角度表达"我想去哪"。三种亲和性分别解决不同问题：NodeAffinity 控制 Pod 和 Node 的关系，PodAffinity 让 Pod 和 Pod 靠近，PodAntiAffinity 让 Pod 和 Pod 远离。

### NodeAffinity

NodeAffinity 是 `nodeSelector` 的增强版，功能更灵活。它决定 Pod 可以调度到哪些节点。

#### 两种策略

- RequiredDuringSchedulingIgnoredDuringExecution（硬性要求）：必须满足条件才能调度，不满足就不调度，Pod 会一直 Pending。名字中的 "IgnoredDuringExecution" 是说调度之后如果节点标签变了，已经在运行的 Pod 不会被驱逐。

- PreferredDuringSchedulingIgnoredDuringExecution（软性偏好）：尽量满足，满足不了也能调度到其他节点。可以设置权重（1-100），权重越高优先级越大。

#### 匹配类型

```yaml
# In：标签值在列表中（最常用）
- key: "zone"
  operator: In
  values: ["east", "west"]

# NotIn：标签值不在列表中
- key: "zone"
  operator: NotIn
  values: ["north"]

# Exists：标签存在就行，不关心值
- key: "gpu"
  operator: Exists

# DoesNotExist：标签不存在
- key: "deprecated"
  operator: DoesNotExist

# Gt：标签值大于指定数字（值会被解析为整数）
- key: "core-count"
  operator: Gt
  values: ["4"]

# Lt：标签值小于指定数字
- key: "core-count"
  operator: Lt
  values: ["32"]
```

`Gt` 和 `Lt` 是 NodeAffinity 独有的，PodAffinity 中没有。

#### 完整示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      affinity:
        nodeAffinity:
          # 硬性要求：必须在 east 或 west 区域
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: "topology.kubernetes.io/zone"
                    operator: In
                    values: ["east", "west"]
                  - key: "node-type"
                    operator: NotIn
                    values: ["spot"] # 不要调度到竞价实例
              # 多个 nodeSelectorTerms 之间是 OR 关系
              # 同一个 nodeSelectorTerms 内多个 matchExpressions 是 AND 关系

          # 软性偏好：优先选择 SSD 节点
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80 # 权重 1-100
              preference:
                matchExpressions:
                  - key: "disk-type"
                    operator: In
                    values: ["ssd"]
            - weight: 20 # SSD 优先级更高
              preference:
                matchExpressions:
                  - key: "cpu-type"
                    operator: In
                    values: ["high-perf"]

      containers:
        - name: app
          image: nginx:1.24
```

这段配置的意思是：Pod 必须调度到 east 或 west 区域的非竞价实例节点上（硬性），在满足条件的节点中优先选 SSD 节点（权重80），其次选高性能 CPU 节点（权重20）。

```
nodeSelectorTerms 之间 → OR（满足任意一组就行）
  matchExpressions 之间 → AND（同一组内必须全部满足）
```

### PodAffinity

PodAffinity 控制的是 Pod 和 Pod 之间的关系——让某些 Pod 调度到同一拓扑域内。典型场景是让 Web 服务和它的 Redis 缓存跑在同一个节点或同一个可用区，减少网络延迟。

#### topologyKey

这是 PodAffinity 特有的概念，topologyKey 定义了"靠近"的范围：

```yaml
topologyKey: "kubernetes.io/hostname"              # 同一个节点
topologyKey: "topology.kubernetes.io/zone"          # 同一个可用区
topologyKey: "topology.kubernetes.io/region"        # 同一个地域
```

#### 完整示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
    spec:
      affinity:
        podAffinity:
          # 硬性：必须和 redis 在同一个可用区
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values: ["redis"]
              topologyKey: "topology.kubernetes.io/zone"

          # 软性：尽量和 redis 在同一个节点
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: "app"
                      operator: In
                      values: ["redis"]
                topologyKey: "kubernetes.io/hostname"

      containers:
        - name: frontend
          image: web-frontend:latest
```

调度器会先找到所有 label 为 `app=redis` 的 Pod 所在的节点，然后根据 topologyKey 确定范围，把新的 Pod 调度到同一范围内。

### PodAntiAffinity

和 PodAffinity 相反，让 Pod 之间互相远离。最典型的场景是高可用部署——同一个应用的多个副本分散到不同节点或不同可用区。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-broker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      affinity:
        podAntiAffinity:
          # 硬性：同一个节点上不能有两个 kafka broker
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values: ["kafka"]
              topologyKey: "kubernetes.io/hostname"

          # 软性：尽量分散到不同可用区
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: "app"
                      operator: In
                      values: ["kafka"]
                topologyKey: "topology.kubernetes.io/zone"

      containers:
        - name: kafka
          image: bitnami/kafka:3.6
```

这样 3 个 Kafka Broker 一定不会跑在同一个节点上（硬性），并且尽量分散到不同可用区（软性），实现了最大程度的高可用。

### 生产配置

把三种亲和性结合在一起使用：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        tier: backend
    spec:
      affinity:
        # 节点亲和：必须在生产节点，优先 SSD
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: "env"
                    operator: In
                    values: ["production"]
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80
              preference:
                matchExpressions:
                  - key: "disk-type"
                    operator: In
                    values: ["ssd"]

        # Pod 亲和：尽量和 Redis 在同一可用区
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 90
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: "app"
                      operator: In
                      values: ["redis"]
                topologyKey: "topology.kubernetes.io/zone"

        # Pod 反亲和：自己的副本必须分散到不同节点
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values: ["order-service"]
              topologyKey: "kubernetes.io/hostname"

      # 同时配合容忍，因为生产节点有污点
      tolerations:
        - key: "env"
          operator: "Equal"
          value: "production"
          effect: "NoExecute"

      containers:
        - name: order
          image: order-service:latest
```
