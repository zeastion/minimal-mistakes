


## || 所有节点

### 0- SELinux & iptables

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

一共安装五个包，不能 fanqiang 的话，点击这里下载

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

  kubeadm join --token 342e1e.aa80457d33898003 10.50.50.139:6443
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
kubeadm join --token 342e1e.aa80457d33898003 10.50.50.139:6443
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
serviceaccount "flannel" created
configmap "kube-flannel-cfg" created
daemonset "kube-flannel-ds" created

# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
clusterrole "flannel" created
clusterrolebinding "flannel" created
```

网络的 pod 安装成功后，还会装上 kube-dns 的 pod，查看当前状态

```bash
# kubectl get pods --all-namespaces
NAMESPACE     NAME                            READY     STATUS    RESTARTS   AGE
kube-system   etcd-k8s01                      1/1       Running   0          3m
kube-system   kube-apiserver-k8s01            1/1       Running   0          3m
kube-system   kube-controller-manager-k8s01   1/1       Running   0          3m
kube-system   kube-dns-2425271678-r4zr1       3/3       Running   0          4m
kube-system   kube-flannel-ds-f84jf           2/2       Running   0          1m
kube-system   kube-proxy-3z5gt                1/1       Running   0          4m
kube-system   kube-scheduler-k8s01            1/1       Running   0          3m
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

```

# 待续
***
```bash
# kubectl get pods --all-namespaces
NAMESPACE     NAME                                         READY     STATUS    RESTARTS   AGE
...
kube-system   kubernetes-dashboard-head-2208681577-76wqc   1/1       Running   0          57s
```


## || Nodes 计算节点


2- Kubectl

下载最新的稳定版（目前为 1.7.6）

```bash
# curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

# chmod +x kubectl 

# mv kubectl /usr/local/bin/kubectl
```
