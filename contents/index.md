---
site:
  hide_toc: true
---

## Prep

#### 1. HPA

```sh
kubectl -n autoscale autoscale deployment apache-server --name=apache-server --cpu=50% --min=1 --max=4
kubectl -n autoscale edit hpa apache-server
```

```yaml
spec:
  maxReplicas: 4
  # 新增下面3行
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 30
```

```sh
# 检查
kubectl -n autoscale get hpa apache-server
```

#### 2. Ingress

```sh
# 可能是nginx，如果查出来的是traefik，就用traefik
kubectl get ingressclasses.networking.k8s.io
vim ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
  namespace: sound-repeater
  # 将请求的 URL 路径重定向到/，配合下面的 path: "/echo" 使用，即访问/下的/echo。
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx #这里写上一步查询到的名字
  rules:
    - host: "example.org" #填写题目要求的域名
      http: # 因为上一行写了 – host 了，有一个-了，所以这里 http 前面的 - 要取消
        paths:
          - path: /echo #填写题目要求的 URL 路径，题目要求 http://example.org/echo
            pathType: Prefix
            backend:
              service:
                name: echoserver-service #填写题目要求的 Service
                port:
                  number: 8080 #填写题目要求的端口
```

```sh
kubectl apply -f ingress.yaml
curl http://example.org/echo
```

#### 3. Sidecar

```sh
kubectl get deployment synergy-leverager -o yaml > sidecar.yaml
vim sidecar.yaml
```

`dnsPolicy: ClusterFirst`上面添加

```yaml
  volumeMounts:
  - name: varlog
    mountPath: /var/log
- name: sidecar
  image: busybox:stable
  args: [/bin/sh, -c, 'tail -n+1 -f /var/log/synergy-leverager.log']
  volumeMounts:
  - name: varlog
    mountPath: /var/log
```

`status:`上面添加

```yaml
volumes:
  - name: varlog
    emptyDir: {}
```

```sh
kubectl apply -f sidecar.yaml
kubectl get deployment synergy-leverager
kubectl get pod | grep synergy-leverager
kubectl logs synergy-leverager-579b88ffdf-dq68l -c sidecar
```

#### 4. StorageClass

```sh
vim sc.yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ran-local-path #根据题目要求的修改
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
```

```sh
kubectl apply -f sc.yaml
kubectl get storageclass
```

#### 5. Service

```yaml
spec:
  containers:
    - image: vicuu/nginx:hello
      imagePullPolicy: IfNotPresent
      name: nginx
      # 新增下面3行
      ports:
        - containerPort: 80
          protocol: TCP
```

```sh
# 注意考试中需要创建的是 NodePort，还是 ClusterIP。
# 如果是 ClusterIP，则应改为 --type=ClusterIP
#--port 是 service 的端口号，--target-port 是 deployment 里 pod 的容器的端口号。
# --name 是 service 的名字
kubectl -n spline-reticulator expose deployment front-end --type=NodePort --port=80 --target-port=80 --name=front-end-svc
kubectl -n spline-reticulator get svc front-end-svc -o wide
curl 10.109.100.207:80 # ip用查出来的
```

#### 6. PriorityClass

```yaml
kubectl get priorityclass
vim priority.yaml
```

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 999999999
globalDefault: false
description: "one less"
```

```sh
kubectl apply -f priority.yaml
kubectl get priorityclass high-priority
kubectl get priorityclass
kubectl -n priority edit deployment busybox-logger
```

`dnsPolicy: ClusterFirst`上面添加

```yaml
priorityClassName: high-priority
```

```sh
kubectl -n priority get pod |grep busybox-logger
```

#### 7. Argo CD

```sh
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm search repo argo |grep argo-cd
helm template argocd argo/argo-cd --version 5.5.22 --namespace argocd --set crds.install=false > ~/argo-helm.yaml
helm install argocd argo/argo-cd --version 5.5.22 --namespace argocd --set crds.install=false
kubectl -n argocd get pods
```

#### 8. PVC

```sh
kubectl get pv # 看StorageClass
vim pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb #修改为题目要求的 pvc 名字
  namespace: mariadb #注意新增 namespace
spec:
  storageClassName: local-path #写上一步查到的
  accessModes:
    - ReadWriteOnce #按照题目要求修改，还有可能是 ReadWriteMany
  resources:
    requests:
      storage: 250Mi #一定要按照题目要求的大小设置
```

```sh
kubectl apply -f pvc.yaml
vim ~/mariadb-deployment.yaml
```

```yaml
volumes:
  - name: mariadb-data
    persistentVolumeClaim:
      claimName: "mariadb" # 修改
```

```sh
kubectl apply -f ~/mariadb-deployment.yaml
kubectl -n mariadb get deployment
kubectl -n mariadb get pod
```

#### 9. Gateway

```sh
# 确认paths下面和secretName
kubectl get ingress web -o yaml
vim gateway.yaml
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
spec:
  gatewayClassName: nginx #从题目里【集群中安装了一个名为 nginx 的 GatewayClass】得知这里写 nginx。若题目写的是名为 traefik 的，则这里写 traefik
  listeners:
    - name: https
      protocol: HTTPS #这里一定要是 HTTPS
      port: 443 # 这个就是默认的 443，不需要改
      hostname: gateway.web.k8s.local #填写题目里要求的主机名
      tls:
        certificateRefs:
          - kind: Secret
            group: ""
            name: web-cert # 填写 ingress 里的 TLS secretName
```

```sh
kubectl apply -f gateway.yaml
kubectl get gateway
vim httproute.yaml
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
    - name: web-gateway #上面创建的 Gateway 名字
  hostnames:
    - "gateway.web.k8s.local" #题目要求的主机名
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: / # ingress 里的 paths path
      backendRefs:
        - name: web # ingress 里的 service name
          port: 80 # ingress 里的 service port number
```

```sh
kubectl apply -f httproute.yaml
kubectl get httproute
curl -Lk https://gateway.web.k8s.local:31443
kubectl delete ingress web
```

#### 10. NetworkPolicy

```sh
kubectl get ns frontend backend --show-labels
kubectl -n frontend get pod --show-labels
kubectl -n backend get pod --show-labels
kubectl -n backend get networkpolicies
cat ~/netpol/netpol1.yaml
cat ~/netpol/netpol2.yaml
cat ~/netpol/netpol3.yaml
kubectl apply -f ~/netpol/netpol2.yaml
kubectl -n backend get networkpolicies
```

#### 11. CRD

```sh
kubectl -n cert-manager get pods
kubectl get crd | grep cert-manager
kubectl get crd | grep cert-manager > ~/resources.yaml
cat resources.yaml
kubectl explain certificate.spec.subject
kubectl explain certificate.spec.subject > ~/subject.yaml
cat subject.yaml
```

#### 12. ConfigMap

```sh
kubectl -n nginx-static edit configmaps nginx-config
```

```yaml
ssl_protocols TLSv1.2 TLSv1.3; # 按照题目要求，添加 TLSv1.2
```

```sh
kubectl -n nginx-static rollout restart deployment nginx-static
kubectl -n nginx-static get pod
curl -k --tls-max 1.2 https://web.k8snginx.local
kubectl -n nginx-static edit configmaps nginx-config
```

`kind: ConfigMap`上面添加

```yaml
immutable: true
```

#### 13. Calico

```sh
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml
kubectl cluster-info dump | grep -i cluster-cidr
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
vi custom-resources.yaml
```

```yaml
cidr: 10.244.0.0/16 # 按照上面查出来的
```

```sh
kubectl create -f custom-resources.yaml
kubectl -n calico-system get pod # 等两分钟
```

#### 14. WordPress

```sh
kubectl -n relative-fawn scale deployment wordpress --replicas=0
kubectl -n relative-fawn get deployment wordpress
kubectl get nodes
kubectl describe node base
kubectl -n relative-fawn edit deployment wordpress
# 将配置文件里，2 个 containers 的 requests cpu 设置为 80m，内存设置为 200Mi
kubectl -n relative-fawn scale deployment wordpress --replicas=3
kubectl -n relative-fawn get pod # 等两分钟
kubectl -n relative-fawn get deployment
```

#### 15. etcd fix

```sh
sudo -i
vim /etc/kubernetes/manifests/kube-apiserver.yaml
# 修改 --etcd-servers=https://127.0.0.1:2379
systemctl daemon-reload
systemctl restart kubelet
# 等1-3分钟
kubectl get nodes
kubectl -n kube-system get pod
lscpu | grep CPU
vim /etc/kubernetes/manifests/kube-scheduler.yaml
# 修改为cpu: 200m
systemctl daemon-reload
systemctl restart kubelet
# 等1-3分钟
kubectl get nodes
kubectl -n kube-system get pod
exit * 2
```

#### 16. cri-dockerd

```sh
sudo dpkg -i ~/cri-dockerd_0.3.21.3-0.ubuntu-jammy_64.deb
sudo systemctl enable cri-docker
sudo systemctl start cri-docker
sudo systemctl status cri-docker # q退出
sudo vim /etc/sysctl.conf
```

末尾添加

```yaml
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
net.netfilter.nf_conntrack_max = 131072
```

```sh
sudo sysctl -p
cat /proc/sys/net/netfilter/nf_conntrack_max
```
