RBAC


## || 所有节点

### 0- SELinux & iptables & hosts

关闭 SELinux 以便容器可以访问宿主机文件系统

```bash
# setenforce 0

# vim /etc/selinux/config
SELINUX=disabled
SELINUXTYPE=targeted
```

关闭防火墙

```bash
# systemctl stop firewalld && systemctl disable firewalld

# yum install iptables-services

# systemctl start iptables

# iptables -F

# service iptables save
```

域名解析

```bash
# vim /etc/hosts
10.50.50.139    k8s01
10.50.50.140    k8s02
10.50.50.141    k8s03
```

### 1- Docker

Kubernetes 官方推荐使用 Docker 1.12 版本，1.13 及 17.03+ 尚未经过测试

```bash
# yum install docker

# systemctl enable docker
```

kubeadm 初始化过程中会下载谷歌镜像，需配置好 docker 代理

```bash
# vim /etc/sysconfig/docker
...
http_proxy=http://duotai:5jbgxAIOs@hyatt.h.timonit.cn:16366
https_proxy=http://duotai:5jbgxAIOs@hyatt.h.timonit.cn:16366

# systemctl start docker

# docker version
Client:
 Version:         1.12.6
 API version:     1.24
 Package version: docker-1.12.6-48.git0fdc778.el7.centos.x86_64
 Go version:      go1.8.3
 Git commit:      0fdc778/1.12.6
 Built:           Thu Sep  7 18:00:07 2017
 OS/Arch:         linux/amd64

Server:
 Version:         1.12.6
 API version:     1.24
 Package version: docker-1.12.6-48.git0fdc778.el7.centos.x86_64
 Go version:      go1.8.3
 Git commit:      0fdc778/1.12.6
 Built:           Thu Sep  7 18:00:07 2017
 OS/Arch:         linux/amd64
```

### 2- 安装 kubectl & Kubelet & Kubeadm

添加谷歌的 kubernetes 源（需要fanqiang）

```bash
# vim /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg

# yum install kubelet kubeadm kubectl -y
```

一共安装五个包，不能 fanqiang 的话，点击[这里下载](https://pan.baidu.com/s/1dEHXOLN)

启动 kubelet 后会因为缺少配置文件一直报错，暂时不用理会

```bash
# systemctl enable kubelet && systemctl start kubelet
```


## || Master 节点

### 1- 准备容器镜像

运行 kubeadm 之前，先把需要的镜像准备好，否则 kubeadm init 会卡顿很久

```bash
# docker pull gcr.io/google_containers/pause-amd64:3.0

# docker pull gcr.io/google_containers/etcd-amd64:3.0.17

# docker pull gcr.io/google_containers/kube-scheduler-amd64:v1.7.6

# docker pull gcr.io/google_containers/kube-controller-manager-amd64:v1.7.6

# docker pull gcr.io/google_containers/kube-apiserver-amd64:v1.7.6

# docker pull gcr.io/google_containers/kube-proxy-amd64:v1.7.6

# docker pull gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.4

# docker pull gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.4

# docker pull gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.4

# docker pull gcr.io/google_containers/kubernetes-dashboard-init-amd64:v1.0.0

# docker pull gcr.io/google_containers/kubernetes-dashboard-amd64:v1.7.0

```

如果不能 fanqiang，可以从我的镜像站下载，再通过 tag 改成 google 的镜像名

以 pause-amd64 镜像为例

```bash
# docker pull zeastion/pause-amd64:3.0

# docker tag zeastion/pause-amd64:3.0 gcr.io/google_containers/pause-amd64:3.0

# docker rmi zeastion/pause-amd64:3.0
```

### 2- 初始化

网络组件选择 Flannel，因此 --pod-network-cidr 参数配置为 10.244.0.0/16

```bash
# kubeadm init --pod-network-cidr=10.244.0.0/16         
[kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
[init] Using Kubernetes version: v1.7.6
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks
[preflight] WARNING: firewalld is active, please ensure ports [6443 10250] are open or your cluster may not function correctly
[kubeadm] WARNING: starting in 1.8, tokens expire after 24 hours by default (if you require a non-expiring token use --token-ttl 0)
[certificates] Generated CA certificate and key.
[certificates] Generated API server certificate and key.
[certificates] API Server serving cert is signed for DNS names [k8s01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.50.50.139]
[certificates] Generated API server kubelet client certificate and key.
[certificates] Generated service account token signing key and public key.
[certificates] Generated front-proxy CA certificate and key.
[certificates] Generated front-proxy client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[apiclient] Created API client, waiting for the control plane to become ready
[apiclient] All control plane components are healthy after 35.003182 seconds
[token] Using token: 342e1e.aa80457d33898003
[apiconfig] Created RBAC rules
[addons] Applied essential addon: kube-proxy
[addons] Applied essential addon: kube-dns

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run (as a regular user):

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token 593d4e.94b88e40f8c54d73 10.50.50.139:6443
```

**注：**

-a- 若有如下报错

```bash
[preflight] Some fatal errors occurred:
        /proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1

https://github.com/moby/moby/issues/24809

# echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
```

-b- 保存好凭证用于添加节点

```bash
kubeadm join --token 593d4e.94b88e40f8c54d73 10.50.50.139:6443
```

### 3- 添加 kubectl 配置
 
```bash
# mkdir -p /root/.kube

# cp -i /etc/kubernetes/admin.conf /root/.kube/config
```

### 4- 添加网络组件 Flannel

查看 messages 日志会一直有一个网络报错

```bash
Sep 15 09:44:56 k8s01 kubelet: E0915 09:44:56.414852   39587 kubelet.go:2136] Container runtime network not ready:
NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

官方有相关说明 https://github.com/kubernetes/kubernetes/issues/43815

```bash
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
clusterrole "flannel" created
clusterrolebinding "flannel" created
serviceaccount "flannel" created
configmap "kube-flannel-cfg" created
daemonset "kube-flannel-ds" created
```

网络的 pod 安装成功后，还会装上 kube-dns 的 pod，查看当前状态

```bash
# kubectl get pods --namespace=kube-system
NAME                            READY     STATUS    RESTARTS   AGE
etcd-k8s01                      1/1       Running   0          17m
kube-apiserver-k8s01            1/1       Running   0          17m
kube-controller-manager-k8s01   1/1       Running   0          17m
kube-dns-2425271678-rrp36       3/3       Running   0          17m
kube-flannel-ds-q5bc1           1/1       Running   0          5m
kube-proxy-lq776                1/1       Running   0          17m
kube-scheduler-k8s01            1/1       Running   0          17m
```

### 5- Dashboard

根据适配表，选择 head 版本

https://github.com/kubernetes/dashboard

| 兼容性 | Kubernetes 1.4 | Kubernetes 1.5 | kubernetes 1.6 | Kubernetes 1.7 | 
| ------------  | ------------:  | ------------:  | ------------:  | ------------:  | 
| Dashboard 1.4 | ✓ | ✕ | ✕ | ✕ |
| Dashboard 1.5 | ✕ | ✓ | ✕ | ✕ |
| Dashboard 1.6 | ✕ | ✕ | ✓ | ? | 
| Dashboard HEAD | ✕ | ✕ | ? | ✓ |

```bash
# cd /etc/kubernetes/manifests/

# wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

# kubectl create -f kubernetes-dashboard.yaml 
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role "kubernetes-dashboard-minimal" created
rolebinding "kubernetes-dashboard-minimal" created
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created
```

**注：**

目前还有问题 https://github.com/kubernetes/dashboard/issues/2397

回答的办法并不能解决


## || Nodes 计算节点

### 1- 预先下载镜像

```bash
# docker pull gcr.io/google_containers/pause-amd64:3.0

# docker pull gcr.io/google_containers/kube-proxy-amd64:v1.7.6
```

### 2- 加入集群

```bash
# kubeadm join --token 593d4e.94b88e40f8c54d73 10.50.50.139:6443
[kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
[preflight] Running pre-flight checks
[discovery] Trying to connect to API Server "10.50.50.139:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://10.50.50.139:6443"
[discovery] Cluster info signature and contents are valid, will use API Server "https://10.50.50.139:6443"
[discovery] Successfully established connection with API Server "10.50.50.139:6443"
[bootstrap] Detected server version: v1.7.6
[bootstrap] The server supports the Certificates API (certificates.k8s.io/v1beta1)
[csr] Created API client to obtain unique certificate for this node, generating keys and certificate signing request
[csr] Received signed certificate from the API server, generating KubeConfig...
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"

Node join complete:
* Certificate signing request sent to master and response
  received.
* Kubelet informed of new secure connection details.

Run 'kubectl get nodes' on the master to see this machine join.
```


## || 测试

1- 查看 api

https://10.50.50.139:6443/api/v1

2- sock-shop 

https://github.com/microservices-demo/microservices-demo

```bash
# kubectl create namespace sock-shop
namespace "sock-shop" created

# kubectl apply -n sock-shop -f "https://github.com/microservices-demo/microservices-demo/blob/master/deploy/kubernetes/complete-demo.yaml?raw=true"
deployment "carts-db" created
service "carts-db" created
deployment "carts" created
service "carts" created
deployment "catalogue-db" created
service "catalogue-db" created
deployment "catalogue" created
service "catalogue" created
deployment "front-end" created
service "front-end" created
deployment "orders-db" created
service "orders-db" created
deployment "orders" created
service "orders" created
deployment "payment" created
service "payment" created
deployment "queue-master" created
service "queue-master" created
deployment "rabbitmq" created
service "rabbitmq" created
deployment "shipping" created
service "shipping" created
deployment "user-db" created
service "user-db" created
deployment "user" created
service "user" created
```

查看其信息

```bash
# kubectl -n sock-shop get svc front-end
NAME        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
front-end   10.105.162.95   <nodes>       80:30001/TCP   2m

# kubectl get po --namespace=sock-shop -o wide
NAME                            READY     STATUS    RESTARTS   AGE       IP            NODE
carts-511261774-8zx0m           1/1       Running   0          10m       10.244.2.3    k8s02
carts-db-549516398-7g4fb        1/1       Running   0          10m       10.244.2.2    k8s02
catalogue-4293036822-lk1wd      1/1       Running   0          10m       10.244.2.5    k8s02
catalogue-db-1846494424-43lm2   1/1       Running   0          10m       10.244.2.4    k8s02
front-end-2337481689-4jwnz      1/1       Running   0          10m       10.244.2.7    k8s02
orders-208161811-4ndhm          1/1       Running   0          10m       10.244.2.8    k8s02
orders-db-2069777334-h09vs      1/1       Running   0          10m       10.244.2.6    k8s02
payment-3050936124-mxxxd        1/1       Running   0          10m       10.244.2.9    k8s02
queue-master-2067646375-9pn6k   1/1       Running   0          10m       10.244.2.10   k8s02
rabbitmq-241640118-kw4ws        1/1       Running   0          10m       10.244.2.14   k8s02
shipping-3132821717-644hc       1/1       Running   0          10m       10.244.2.11   k8s02
user-1574605338-xzrhn           1/1       Running   0          10m       10.244.2.12   k8s02
user-db-2947298815-xbhjx        1/1       Running   0          10m       10.244.2.13   k8s02
```

网页访问

![sock-shop](http://ov30w4cpi.bkt.clouddn.com/sock-shop.png)


## || 删除节点

在 Master 上

```bash
# kubectl drain <node name> --delete-local-data --force --ignore-daemonsets

# kubectl delete node <node name>
```

在 Node 上

```bash
# kubeadm reset
```







2- Kubectl

下载最新的稳定版（目前为 1.7.6）

```bash
# curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

# chmod +x kubectl 

# mv kubectl /usr/local/bin/kubectl
```
