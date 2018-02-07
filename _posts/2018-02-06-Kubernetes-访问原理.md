Kubernetes 集群内外访问原理


## 1- 通过 Service 访问 Pod

集群中 Pod 是不稳定的，随着故障 Pod 被集群拉起的新 Pod 替换，
随之带来 IP 的变化，因此通过 Service 统一对外提供服务

通过 Deployment 创建一组 Pod，使用 'labels' 将其和 Service 建立映射关系

创建 Deployment，设置 'labels' 为 'run: httpd'

```bash
# vim httpd.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: httpd
spec:
  replicas: 3
  template:
    metadata:
      labels:
        run: httpd
    spec:
      containers:
      - name: httpd
        image: httpd
        ports:
        - containerPort: 80

# kubectl apply -f httpd.yaml 
deployment "httpd" created

# kubectl get pods -o wide
NAME                    READY     STATUS    RESTARTS   AGE       IP            NODE
httpd-741508562-4h08z   1/1       Running   0          25s       10.244.1.28   f16-7-kubernetes02
httpd-741508562-j4lsm   1/1       Running   0          25s       10.244.1.29   f16-7-kubernetes02
httpd-741508562-jvgbg   1/1       Running   0          25s       10.244.2.22   f16-8-kubernetes03
```

每个 Pod 都有自己的 IP，但只能被集群中容器和节点访问

```bash
# curl 10.244.1.28
<html><body><h1>It works!</h1></body></html>
```

使用 'selector' 选定 'labels' 为 'run: httpd' 的 Pod

将 Service 的 8080 端口映射到 Pod 的 80 端口，且指定 TCP 协议

```bash
# vim httpd-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: httpd-svc
spec:
  selector:
    run: httpd
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 80

# kubectl apply -f httpd-svc.yaml 
service "httpd-svc" created

# kubectl get services
NAME         CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
httpd-svc    10.96.203.6   <none>        8080/TCP   1m
kubernetes   10.96.0.1     <none>        443/TCP    88d
```

由 Service 管理的 CLUSTER-IP 是持久稳定的

```bash
# curl 10.96.203.6:8080
<html><body><h1>It works!</h1></body></html>
```

查看详细信息，可以看到三个后端 Pod

```bash
# kubectl describe service httpd-svc
Name:                   httpd-svc
Namespace:              default
Labels:                 <none>
Annotations:            kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"httpd-svc","namespace":"default"},"spec":{"ports":[{"port":8080,"protocol":"TC...
Selector:               run=httpd
Type:                   ClusterIP
IP:                     10.96.203.6
Port:                   <unset> 8080/TCP
Endpoints:              10.244.1.28:80,10.244.1.29:80,10.244.2.22:80
Session Affinity:       None
Events:                 <none>
```


## 2- Cluster IP
 
Service 的 Cluster IP 是通过 iptables 实现的虚拟 IP

```bash
# iptables-save | grep 10.96.203.6
-A KUBE-SERVICES ! -s 10.244.0.0/16 -d 10.96.203.6/32 -p tcp -m comment --comment "default/httpd-svc: cluster IP" -m tcp --dport 8080 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.96.203.6/32 -p tcp -m comment --comment "default/httpd-svc: cluster IP" -m tcp --dport 8080 -j KUBE-SVC-RL3JAE4GN7VOGDGP
```

第一条源自 10.244.0.0/16（集群内 Pod IP）要访问 httpd-svc 则允许

其他源访问 httpd-svc，跳转到 'KUBE-SVC-RL3JAE4GN7VOGDGP' 规则

```bash
# iptables-save | grep KUBE-SVC-RL3JAE4GN7VOGDGP
:KUBE-SVC-RL3JAE4GN7VOGDGP - [0:0]
-A KUBE-SERVICES -d 10.96.203.6/32 -p tcp -m comment --comment "default/httpd-svc: cluster IP" -m tcp --dport 8080 -j KUBE-SVC-RL3JAE4GN7VOGDGP
-A KUBE-SVC-RL3JAE4GN7VOGDGP -m comment --comment "default/httpd-svc:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-CR63JSDONEUBE2YK
-A KUBE-SVC-RL3JAE4GN7VOGDGP -m comment --comment "default/httpd-svc:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-Y5EOPQEI6R2HKSKZ
-A KUBE-SVC-RL3JAE4GN7VOGDGP -m comment --comment "default/httpd-svc:" -j KUBE-SEP-XY6KDITI4WWURGO2
```

所有 'KUBE-SVC-RL3JAE4GN7VOGDGP' 规则中：

- 0.33332999982(1/3)概率跳转到 'KUBE-SEP-CR63JSDONEUBE2YK' 规则
- 剩下的 2/3 中，0.5(1/3)概率跳转到 'KUBE-SEP-Y5EOPQEI6R2HKSKZ' 规则
- 其他的(1/3)跳转到 'KUBE-SEP-XY6KDITI4WWURGO2' 规则

分别查看这三个规则对应后端

```bash
-A KUBE-SEP-CR63JSDONEUBE2YK -p tcp -m comment --comment "default/httpd-svc:" -m tcp -j DNAT --to-destination 10.244.1.28:80
-A KUBE-SEP-Y5EOPQEI6R2HKSKZ -p tcp -m comment --comment "default/httpd-svc:" -m tcp -j DNAT --to-destination 10.244.1.29:80
-A KUBE-SEP-XY6KDITI4WWURGO2 -p tcp -m comment --comment "default/httpd-svc:" -m tcp -j DNAT --to-destination 10.244.2.22:80
```

实现了从 Cluster IP 到 Pod IP 的均衡负载（轮询）

集群中每个节点都配置了相同 iptables 规则，保证集群内都可以正常访问 Service


## 3- 通过 DNS 访问 Service

查看 kube-dns 组件

```bash
# kubectl get svc -n kube-system -o wide | grep dns
kube-dns               10.96.0.10       <none>        53/UDP,53/TCP   89d       k8s-app=kube-dns
```

当新的 Service 创建后，kube-dns 会添加该 Service 的 DNS 记录，
可以通过 '<Service_Name>.<Namespace_Name>' 访问 Service，
相同 namespace 可以直接访问 '<Service_name>'

```bash
# kubectl run busybox --rm -it --image=busybox /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget httpd-svc.default:8080
Connecting to httpd-svc.default:8080 (10.96.203.6:8080)
index.html           100% |*****************************************************************************************|    45   0:00:00 ETA
/ # 
/ # nslookup httpd-svc
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      httpd-svc
Address 1: 10.96.203.6 httpd-svc.default.svc.cluster.local
/ # 
```

可见 httpd-svc 的完整域名为 'httpd-svc.default.svc.cluster.local'


## 4- 集群外部访问 Service

- 通过 'ClusterIP' - 集群内的节点和 Pod 可以访问，但集群外部无法联通

- 通过 'NodePort' - Service 使用集群节点的静态端口对外提供服务，外部访问 '<NodeIP>:<NodePort>'

在一个 Yaml 文件中创建 Deployment 和 Service，多种资源使用 '---' 分隔

```bash
# vim myhttpd.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: myhttpd
spec:
  replicas: 3
  template:
    metadata:
      labels:
        run: httpd
    spec:
      containers:
      - name: httpd
        image: httpd
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: myhttpd-svc
spec:
  type: NodePort
  selector:
    run: httpd
  ports:
  - protocol: TCP
    nodePort: 30000
    port: 8080
    targetPort: 80
```

文件中定义了三种端口

- 'nodePort'   - 节点上监听端口
- 'port'       - ClusterIP 监听端口
- 'targetPort' - Pod 监听端口

```bash
# kubectl apply -f myhttpd.yaml 
deployment "myhttpd" created
service "myhttpd-svc" created

# kubectl get deployment -o wide
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINER(S)   IMAGE(S)   SELECTOR
myhttpd   3         3         3            3           32s       httpd          httpd      run=httpd

# kubectl get pods -o wide
NAME                      READY     STATUS    RESTARTS   AGE       IP            NODE
myhttpd-741508562-lt4kr   1/1       Running   0          3h        10.244.1.32   f16-7-kubernetes02
myhttpd-741508562-p3h5w   1/1       Running   0          3h        10.244.1.33   f16-7-kubernetes02
myhttpd-741508562-sm9ft   1/1       Running   0          3h        10.244.2.26   f16-8-kubernetes03

# kubectl get svc myhttpd-svc
NAME          CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
myhttpd-svc   10.104.223.18   <nodes>       8080:30000/TCP   1m

# netstat -ntlp | grep 30000
tcp6       0      0 :::30000                :::*                    LISTEN      3390/kube-proxy
```

各节点均可访问

```bash
# curl F16-6-kubernetes01:30000
<html><body><h1>It works!</h1></body></html>

# curl F16-7-kubernetes02:30000
<html><body><h1>It works!</h1></body></html>

# curl F16-8-kubernetes03:30000
<html><body><h1>It works!</h1></body></html>
```

查看 iptalbes 规则

```bash
# iptables-save | grep 30000
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/myhttpd-svc:" -m tcp --dport 30000 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/myhttpd-svc:" -m tcp --dport 30000 -j KUBE-SVC-7P7YXYLZA3ZMJCMD
```

访问节点 30000 端口的请求应用规则 'KUBE-SVC-7P7YXYLZA3ZMJCMD'

```bash
# iptables-save | grep KUBE-SVC-7P7YXYLZA3ZMJCMD
:KUBE-SVC-7P7YXYLZA3ZMJCMD - [0:0]
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/myhttpd-svc:" -m tcp --dport 30000 -j KUBE-SVC-7P7YXYLZA3ZMJCMD
-A KUBE-SERVICES -d 10.104.223.18/32 -p tcp -m comment --comment "default/myhttpd-svc: cluster IP" -m tcp --dport 8080 -j KUBE-SVC-7P7YXYLZA3ZMJCMD
-A KUBE-SVC-7P7YXYLZA3ZMJCMD -m comment --comment "default/myhttpd-svc:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-5JFNR2OVUEO7ADZR
-A KUBE-SVC-7P7YXYLZA3ZMJCMD -m comment --comment "default/myhttpd-svc:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-MXGVDRPRKF4FCFQ7
-A KUBE-SVC-7P7YXYLZA3ZMJCMD -m comment --comment "default/myhttpd-svc:" -j KUBE-SEP-3ZRQHJQFDZH6HPK7
```

规则 'KUBE-SVC-7P7YXYLZA3ZMJCMD' 以各 1/3 比率分配到 

'KUBE-SEP-5JFNR2OVUEO7ADZR';

'KUBE-SEP-MXGVDRPRKF4FCFQ7';

'KUBE-SEP-3ZRQHJQFDZH6HPK7'

查看这三条规则

```bash
# iptables-save | grep KUBE-SEP-5JFNR2OVUEO7ADZR
:KUBE-SEP-5JFNR2OVUEO7ADZR - [0:0]
-A KUBE-SEP-5JFNR2OVUEO7ADZR -s 10.244.1.32/32 -m comment --comment "default/myhttpd-svc:" -j KUBE-MARK-MASQ
-A KUBE-SEP-5JFNR2OVUEO7ADZR -p tcp -m comment --comment "default/myhttpd-svc:" -m tcp -j DNAT --to-destination 10.244.1.32:80

# iptables-save | grep KUBE-SEP-MXGVDRPRKF4FCFQ7
:KUBE-SEP-MXGVDRPRKF4FCFQ7 - [0:0]
-A KUBE-SEP-MXGVDRPRKF4FCFQ7 -s 10.244.1.33/32 -m comment --comment "default/myhttpd-svc:" -j KUBE-MARK-MASQ
-A KUBE-SEP-MXGVDRPRKF4FCFQ7 -p tcp -m comment --comment "default/myhttpd-svc:" -m tcp -j DNAT --to-destination 10.244.1.33:80

# iptables-save | grep KUBE-SEP-3ZRQHJQFDZH6HPK7
:KUBE-SEP-3ZRQHJQFDZH6HPK7 - [0:0]
-A KUBE-SEP-3ZRQHJQFDZH6HPK7 -s 10.244.2.26/32 -m comment --comment "default/myhttpd-svc:" -j KUBE-MARK-MASQ
-A KUBE-SEP-3ZRQHJQFDZH6HPK7 -p tcp -m comment --comment "default/myhttpd-svc:" -m tcp -j DNAT --to-destination 10.244.2.26:80
```

最终请求落到了 Pod 上
