
## || 集群信息

centos01 - 10.50.50.131 - Master  
centos02 - 10.50.50.132 - Minion  
centos03 - 10.50.50.133 - Minion  

## || 各节点

### 1- 域名解析

```bash
# vim /etc/hosts
10.50.50.131    centos01
10.50.50.132    centos02
10.50.50.133    centos03
```

### 2- 配置 yum 源

```bash
# vim /etc/yum.repos.d/virt7-docker-common-release.repo
[virt7-docker-common-release]
name=virt7-docker-common-release
baseurl=http://cbs.centos.org/repos/virt7-docker-common-release/x86_64/os/
gpgcheck=0
```

### 3- 安装包

```bash
# yum -y install --enablerepo=virt7-docker-common-release kubernetes etcd flannel
```

### 4- 查看版本

```bash
# kube-apiserver --version
Kubernetes v1.5.2

# flanneld -version
0.7.1

# etcd -version
etcd Version: 3.1.9
Git SHA: 0f4a535
Go Version: go1.7.4
Go OS/Arch: linux/amd64
```

### 5- 修改主配置文件

```bash
# vim /etc/kubernetes/config
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=0"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://centos01:8080"
```

## || 主节点

### 1- Etcd 配置

```bash
# vim /etc/etcd/etcd.conf
ETCD_NAME=default
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"
```

### 2- Apiserver 配置

暂时去掉 ServiceAccount

```bash
# vim /etc/kubernetes/apiserver
KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"
KUBE_ETCD_SERVERS="--etcd-servers=http://centos01:2379"
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
KUBE_API_ARGS=""
```

### 3- 启动 Etcd，创建 flannel 网段

```bash
# systemctl start etcd
# systemctl enable etcd
# etcdctl mkdir /kube-centos/network
# etcdctl mk /kube-centos/network/config "{ \"Network\": \"172.30.0.0/16\", \"SubnetLen\": 24, \"Backend\": { \"Type\": \"vxlan\" } }"
```

### 4- Flannel 配置

```bash
# vim /etc/sysconfig/flanneld
FLANNEL_ETCD_ENDPOINTS="http://centos01:2379"
FLANNEL_ETCD_PREFIX="/kube-centos/network"
```

### 5- Master 各服务启动脚本

```bash
# vim master.sh
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler flanneld; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done

# sh master.sh
```

### 6- Web UI 界面（待计算节点配置完成后再做此步）

| 兼容性 | Kubernetes 1.4 | Kubernetes 1.5 | kubernetes 1.6 | Kubernetes 1.7 | 
| ------------  | ------------:  | ------------:  | ------------:  | ------------:  | 
| Dashboard 1.4 | ✓ | ✕ | ✕ | ✕ |
| Dashboard 1.5 | ✕ | ✓ | ✕ | ✕ |
| Dashboard 1.6 | ✕ | ✕ | ✓ | ? | 
| Dashboard 1.7 | ✕ | ✕ | ? | ✓ |

选择了没有授权验证的 1.5 简化版本

下载示例 Yaml 文件，再修改 apiserver-host 参数

```bash
# wget https://git.io/kube-dashboard-no-rbac

# vim kube-dashboard-no-rbac 
...
image: googlecontainer/kubernetes-dashboard-amd64:v1.5.1
...
args:
  - --apiserver-host=http://10.50.50.131:8080
...

# kubectl create -f kube-dashboard-no-rbac

# kubectl get pods -o=wide --namespace=kube-system
NAME                                    READY     STATUS    RESTARTS   AGE       IP            NODE
kubernetes-dashboard-1800905438-6v5cx   1/1       Running   0          8m        172.30.14.2   centos02
```

预览，还没配置监控 Heapster

![k8s-ui](http://ov30w4cpi.bkt.clouddn.com/k8s-ui-01.png)

## || 计算节点

### 1- Kubelete配置

```bash
# vim /etc/kubernetes/kubelet
KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_HOSTNAME="--hostname-override=centos02"
KUBELET_API_SERVER="--api-servers=http://centos01:8080"
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
KUBELET_ARGS="--cluster_dns=10.254.0.100 --cluster_domain=cluster.local"
```

### 2- Flannel配置

```bash
# vim /etc/sysconfig/flanneld
FLANNEL_ETCD_ENDPOINTS="http://centos01:2379"
FLANNEL_ETCD_PREFIX="/kube-centos/network"
```

### 3- Minion节点各服务启动脚本

```bash
# vim minion.sh
for SERVICES in kube-proxy kubelet flanneld docker; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done

# sh minion.sh
```

### 4- Kubectl配置

```bash
# kubectl config set-cluster default-cluster --server=http://centos01:8080
# kubectl config set-context default-context --cluster=default-cluster --user=default-admin
# kubectl config use-context default-context
```

## || 查看信息

```bash
# kubectl get nodes
NAME       STATUS    AGE
centos02   Ready     8d
centos03   Ready     8d

# etcdctl ls /kube-centos/network/subnets
/kube-centos/network/subnets/172.30.20.0-24
/kube-centos/network/subnets/172.30.14.0-24
/kube-centos/network/subnets/172.30.96.0-24

# etcdctl get /kube-centos/network/subnets/172.30.98.0-24
{"PublicIP":"10.50.50.131","BackendType":"vxlan","BackendData":{"VtepMAC":"5a:28:03:6a:68:fe"}}
# etcdctl get /kube-centos/network/subnets/172.30.5.0-24 
{"PublicIP":"10.50.50.132","BackendType":"vxlan","BackendData":{"VtepMAC":"c6:ce:c5:d0:2a:ce"}}
# etcdctl get /kube-centos/network/subnets/172.30.58.0-24
{"PublicIP":"10.50.50.133","BackendType":"vxlan","BackendData":{"VtepMAC":"b6:0d:7a:3b:2c:60"}}

# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}
```
