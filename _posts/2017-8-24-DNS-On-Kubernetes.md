Kubernetes DNS schedules a DNS Pod and Service on the cluster, and configures the kubelets to tell individual containers to use the DNS Service’s IP to resolve DNS names.

## || 简介

Kubernetes 使用 skydns 提供 DNS 服务，包含四个组件：
> * etcd: DNS 存储
> * kube2sky: 将 Kubernetes Master 中的 Service 注册到 etcd
> * skyDNS: 提供 DNS 域名解析服务
> * healthz: 提供对 skydns 服务的健康检查功能

## || 配置 DNS

### 1- 创建 RC
skydns 的 RC 配置文件中共包含 4 个容器：etcd、kube2sky、skydns、exechealthz

其中：

--kube-master-url 参数指定 Master 节点物理 IP 和端口

--domain 参数设置 Kubernetes 集群中 Service 所属域名

```bash
# vim skydns-rc.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kube-dns-v11
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    version: v11
    kubernetes.io/cluster-service: "true"
spec:
  replicas: 1
  selector:
    k8s-app: kube-dns
    version: v11
  template:
    metadata:
      labels:
        k8s-app: kube-dns
        version: v11
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
      - name: etcd
        image: googlecontainer/etcd-amd64:2.2.5
        resources:
          limits:
            cpu: 100m
            memory: 50Mi
          requests:
            cpu: 100m
            memory: 50Mi
        command:
        - /usr/local/bin/etcd
        - -data-dir
        - /tmp/data
        - -listen-client-urls
        - http://127.0.0.1:2379,http://127.0.0.1:4001
        - -advertise-client-urls
        - http://127.0.0.1:2379,http://127.0.0.1:4001
        - -initial-cluster-token
        - skydns-etcd
        volumeMounts:
        - name: etcd-storage
          mountPath: /tmp/data
      - name: kube2sky
        image: googlecontainer/kube2sky:1.15
        resources:
          limits:
            cpu: 100m
            memory: 50Mi
          requests:
            cpu: 100m
            memory: 50Mi
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /readiness
            port: 8081
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        args:
          - --kube-master-url=http://10.50.50.131:8080
          - --domain=cluster.local
      - name: skydns
        image: googlecontainer/skydns:2015-10-13-8c72f8c
        resources:
          limits:
            cpu: 100m
            memory: 50Mi
          requests:
            cpu: 100m
            memory: 50Mi
        args:
        - -machines=http://127.0.0.1:4001
        - -addr=0.0.0.0:53
        - -ns-rotate=false
        - -domain=cluster.local
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
      - name: healthz
        image: googlecontainer/exechealthz-amd64:1.0
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
        args:
        - -cmd=nslookup kubernetes.default.svc.cluster.local 127.0.0.1 >/dev/null
        - -port=8080
        ports:
        - containerPort: 8080
          protocol: TCP
      volumes:
      - name: etcd-storage
        emptyDir: {}
      dnsPolicy: Default
      
# kubectl create -f skydns-rc.yaml
```

### 2- 创建 Service
需要指定一个固定 ClusterIP，然后所有 kubelete 节点都以此 IP 为 DNS 地址

```bash
# vim skydns-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "kubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.254.0.100
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
    
# kubectl create -f skydns-svc.yaml
```

### 3- 查看 RC & Service

注意 Service 中生效的 ClusterIP

```bash
# kubectl get pods --namespace=kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
kube-dns-v11-2tsfk                      4/4       Running   0          1h
kubernetes-dashboard-1800905438-6v5cx   1/1       Running   0          1d

# kubectl get svc --namespace=kube-system
NAME                   CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kube-dns               10.254.0.100     <none>        53/UDP,53/TCP   1h
kubernetes-dashboard   10.254.213.135   <none>        80/TCP          1d
```

### 4- 修改计算节点 kubelet 启动参数

每台计算节点 kubelet 添加两个启动参数：

--cluster_dns=10.254.0.100

--cluster_domain=cluster.local

```bash
# vim /etc/kubernetes/kubelet
...
KUBELET_ARGS="--cluster_dns=10.254.0.100 --cluster_domain=cluster.local"
...
```

重启

```bash
# systemctl restart kubelet
```

## || 测试

创建一个 busybox Pod 来测试 DNS 功能是否正常

```bash
# vim busybox.yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: googlecontainer/busybox
    command:
      - sleep
      - "3600"
```
用 busybox 解析
```bash
# kubectl exec busybox -- nslookup kube-dns.kube-system
Server:    10.254.0.100
Address 1: 10.254.0.100

Name:      kube-dns.kube-system
Address 1: 10.254.0.100
```

