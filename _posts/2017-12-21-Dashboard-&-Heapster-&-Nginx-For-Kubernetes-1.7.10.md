给 Kuberetes 集群添加 dashboard 及监控组件 heapster，使用 Nginx 做容器集群的 HTTPS 反向代理（SSL）

预览效果

![443index](http://ov30w4cpi.bkt.clouddn.com/443index.png)

## || Dashboard

### 1- 给 Kubernetes 集群添加 Dashboard

```bash
# cd /etc/kubernetes/manifests/

# wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

# kubectl create -f kubernetes-dashboard.yaml
```

### 2- 集群添加 Heapster

其中三个容器镜像可以从国内站点下载

```bash
# docker pull docker.io/zeastion/heapster-amd64:v1.4.2

# docker pull docker.io/zeastion/heapster-influxdb-amd64:v1.3.3

# docker pull docker.io/zeastion/heapster-grafana-amd64:v4.4.3
```

再改 Tag

```bash
# docker tag docker.io/zeastion/heapster-amd64:v1.4.2 gcr.io/google_containers/heapster-amd64:v1.4.2

# docker tag docker.io/zeastion/heapster-influxdb-amd64:v1.3.3 gcr.io/google_containers/heapster-influxdb-amd64:v1.3.3

# docker tag docker.io/zeastion/heapster-grafana-amd64:v4.4.3 gcr.io/google_containers/heapster-grafana-amd64:v4.4.3
```

可点击[下载](https://pan.baidu.com/s/1kUDM7XX)如下四个 yaml 文件

```bash
# mkdir influxdb && cd influxdb

# vim grafana.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: monitoring-grafana
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: grafana
    spec:
      containers:
      - name: grafana
        image: gcr.io/google_containers/heapster-grafana-amd64:v4.4.3
        ports:
        - containerPort: 3000
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/ssl/certs
          name: ca-certificates
          readOnly: true
        - mountPath: /var
          name: grafana-storage
        env:
        - name: INFLUXDB_HOST
          value: monitoring-influxdb
        - name: GF_SERVER_HTTP_PORT
          value: "3000"
          # The following env variables are required to make Grafana accessible via
          # the kubernetes api-server proxy. On production clusters, we recommend
          # removing these env variables, setup auth for grafana, and expose the grafana
          # service using a LoadBalancer or a public IP.
        - name: GF_AUTH_BASIC_ENABLED
          value: "false"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          value: Admin
        - name: GF_SERVER_ROOT_URL
          # If you're only using the API Server proxy, set this value instead:
          # value: /api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
          value: /
      volumes:
      - name: ca-certificates
        hostPath:
          path: /etc/ssl/certs
      - name: grafana-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: monitoring-grafana
  name: monitoring-grafana
  namespace: kube-system
spec:
  # In a production setup, we recommend accessing Grafana through an external Loadbalancer
  # or through a public IP.
  # type: LoadBalancer
  # You could also use NodePort to expose the service at a randomly-generated port
  # type: NodePort
  ports:
  - port: 80
    targetPort: 3000
  selector:
    k8s-app: grafana

# vim heapster.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: heapster
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: heapster
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: heapster
    spec:
      serviceAccountName: heapster
      containers:
      - name: heapster
        image: gcr.io/google_containers/heapster-amd64:v1.4.2
        imagePullPolicy: IfNotPresent
        command:
        - /heapster
        - --source=kubernetes:https://kubernetes.default
        - --sink=influxdb:http://monitoring-influxdb.kube-system.svc:8086
---
apiVersion: v1
kind: Service
metadata:
  labels:
    task: monitoring
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: Heapster
  name: heapster
  namespace: kube-system
spec:
  ports:
  - port: 80
    targetPort: 8082
  selector:
    k8s-app: heapster

# vim influxdb.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: monitoring-influxdb
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: influxdb
    spec:
      containers:
      - name: influxdb
        image: gcr.io/google_containers/heapster-influxdb-amd64:v1.3.3
        volumeMounts:
        - mountPath: /data
          name: influxdb-storage
      volumes:
      - name: influxdb-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    task: monitoring
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: monitoring-influxdb
  name: monitoring-influxdb
  namespace: kube-system
spec:
  ports:
  - port: 8086
    targetPort: 8086
  selector:
    k8s-app: influxdb

# cd ..

# vim heapster-rbac.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: heapster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:heapster
subjects:
- kind: ServiceAccount
  name: heapster
  namespace: kube-system

# kubectl create -f influxdb/
deployment "monitoring-grafana" created
service "monitoring-grafana" created
serviceaccount "heapster" created
deployment "heapster" created
service "heapster" created
deployment "monitoring-influxdb" created
service "monitoring-influxdb" created

# kubectl create -f heapster-rbac.yaml
clusterrolebinding "heapster" created

```

查看

```bash
# kubectl get pods --namespace=kube-system -o wide
NAME                                         READY     STATUS    RESTARTS   AGE       IP              NODE
etcd-f16-6-kubernetes01                      1/1       Running   0          42d       10.125.144.36   f16-6-kubernetes01
heapster-2277824765-vx1vw                    1/1       Running   0          14h       10.244.2.3      f16-8-kubernetes03
kube-apiserver-f16-6-kubernetes01            1/1       Running   0          42d       10.125.144.36   f16-6-kubernetes01
kube-controller-manager-f16-6-kubernetes01   1/1       Running   0          42d       10.125.144.36   f16-6-kubernetes01
kube-dns-2425271678-05w7k                    3/3       Running   0          42d       10.244.0.2      f16-6-kubernetes01
kube-flannel-ds-91sxr                        1/1       Running   0          42d       10.125.144.36   f16-6-kubernetes01
kube-flannel-ds-vxc2s                        1/1       Running   1          39d       10.125.144.38   f16-8-kubernetes03
kube-flannel-ds-z93h0                        1/1       Running   0          42d       10.125.144.37   f16-7-kubernetes02
kube-proxy-5hcql                             1/1       Running   0          42d       10.125.144.37   f16-7-kubernetes02
kube-proxy-nhjwx                             1/1       Running   1          39d       10.125.144.38   f16-8-kubernetes03
kube-proxy-q2msc                             1/1       Running   0          42d       10.125.144.36   f16-6-kubernetes01
kube-scheduler-f16-6-kubernetes01            1/1       Running   0          42d       10.125.144.36   f16-6-kubernetes01
kubernetes-dashboard-2932672137-5rt7f        1/1       Running   0          42d       10.244.0.4      f16-6-kubernetes01
monitoring-grafana-1219411114-9x682          1/1       Running   0          14h       10.244.1.2      f16-7-kubernetes02
monitoring-influxdb-3087546307-xdxjx         1/1       Running   0          14h       10.244.1.3      f16-7-kubernetes02
```


## || Nginx & SSL

安装 dashboard 和 heapster 后，集群有一个 service 提供页面服务

```bash
# kubectl get svc -n kube-system -o wide | grep dashboard
kubernetes-dashboard   10.105.246.67    <none>        443/TCP         42d       k8s-app=kubernetes-dashboard
```

为了外界可以访问，需要使用 nginx 把这个 CLUSTER-IP 反向代理到 NODE-IP

### 1- 安装

```bash
# yum install epel-release -y

# yum install nginx -y

# systemctl start nginx && systemctl enable nginx

# netstat -ntlp | grep nginx
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      17104/nginx: master 
tcp6       0      0 :::80                   :::*                    LISTEN      17104/nginx: master 
```

### 2- 防火墙

#### a- firewall

```bash
# firewall-cmd --add-service=http
# firewall-cmd --add-service=https
# firewall-cmd --runtime-to-permanent
```

#### b- iptables

```bash
# iptables -I INPUT -p tcp -m tcp --dport 80 -j ACCEPT
# iptables -I INPUT -p tcp -m tcp --dport 443 -j ACCEPT
```

安装完成后，nginx 默认支持 HTTP 转发，对 HTTPS 还不支持，需要配置 SSL

### 2- SSL

#### 创建 SSL 证书

服务器上已有 /etc/ssl/certs 目录，用来存放公共证书，再创建一个 /etc/ssl/private 目录保存私钥

```bash
# mkdir /etc/ssl/private
# chmod 700 /etc/ssl/private
```

使用 OpenSSL 生成自签名密钥和证书对

```bash
# openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
Generating a 2048 bit RSA private key
...........................+++
.................+++
writing new private key to '/etc/ssl/private/nginx-selfsigned.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:SHANGHAI
Locality Name (eg, city) [Default City]:SHANGHAI
Organization Name (eg, company) [Default Company Ltd]:MIGU
Organizational Unit Name (eg, section) []:MIGU
Common Name (eg, your name or your server's hostname) []:10.125.144.36
Email Address []:
```

注： 10.125.144.36 是本台宿主机 IP

#### 创建 Diffie-Hellman 组

```bash
# openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
.+.........
```

### 3- 配置 Nginx 使用 SSL

#### 创建配置文件

```bash
# vim /etc/nginx/conf.d/ssl.conf
server {
    listen 443 http2 ssl;
    listen [::]:443 http2 ssl;

    server_name 10.125.144.36;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
    ssl_dhparam /etc/ssl/certs/dhparam.pem;
}
```

#### 检查

```bash
# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

# systemctl restart nginx

# netstat -ntlp | grep nginx 
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      42693/nginx: master 
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      42693/nginx: master 
tcp6       0      0 :::80                   :::*                    LISTEN      42693/nginx: master 
tcp6       0      0 :::443                  :::*                    LISTEN      42693/nginx: master
```

网页访问 https://10.125.144.36 

![443notsafe](http://ov30w4cpi.bkt.clouddn.com/443notsafe.png)


## || 反向代理

### 1- Nginx 配置

把 dashboard 服务的 CLUSTER-IP 反向代理到 NODE-IP 上（443 端口）

查看 CLUSTER-IP

```bash
# kubectl get svc -n kube-system | grep dashboard
kubernetes-dashboard   10.105.246.67    <none>        443/TCP         42d
```

NODE-IP     10.125.144.36
CLUSTER-IP  10.105.246.67

配置文件 ssl.conf 后面追加 proxy_pass

```bash
# vim /etc/nginx/conf.d/ssl.conf
server {
    listen 443 http2 ssl;
    listen [::]:443 http2 ssl;

    server_name 10.125.144.36;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
    ssl_dhparam /etc/ssl/certs/dhparam.pem;


    root /usr/share/nginx/html;

    location / {
        proxy_pass https://10.105.246.67:443;
    }

    error_page 404 /404.html;
    location = /404.html {
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
    }
}
```

### 2- dashboard 添加用户

默认的 kubernetes-dashboard 用户权限较小，很多数据无法查看
添加一个管理员权限的 ServiceAccount：kubernetes-dashboard-admin

```bash
# vim /etc/kubernetes/manifests/kubernetes-dashboard-admin.rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-admin
  namespace: kube-system
  
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kube-system

# kubectl create -f /etc/kubernetes/manifests/kubernetes-dashboard-admin.rbac.yaml
serviceaccount "kubernetes-dashboard-admin" created
clusterrolebinding "kubernetes-dashboard-admin" created
```

获取登陆所需 token

```bash
# kubectl get secret -n kube-system | grep kubernetes-dashboard-admin
kubernetes-dashboard-admin-token-73pxk   kubernetes.io/service-account-token   3         19m

# kubectl describe secret kubernetes-dashboard-admin-token-73pxk -n kube-system
Name:           kubernetes-dashboard-admin-token-73pxk
Namespace:      kube-system
Labels:         <none>
Annotations:    kubernetes.io/service-account.name=kubernetes-dashboard-admin
                kubernetes.io/service-account.uid=d2ba0bc7-e6e8-11e7-9296-f44c7f1da78a

Type:   kubernetes.io/service-account-token

Data
====
ca.crt:         1025 bytes
namespace:      11 bytes
token:          eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi03M3B4ayIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImQyYmEwYmM3LWU2ZTgtMTFlNy05Mjk2LWY0NGM3ZjFkYTc4YSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.o2rPQ9LWwyS78Qe2trKJlGN0XESv8KQ1Ou1fEWkcRZNUiaFyWDmi5ZYCbTE8Ni36sEymYVjGE8kBLjRmFgzxRtWmEVJloUSD4gEwti6W_Zuoq7sqd0aM_6LXMiUN_IMZ0sJYf475NvTbNbC-GlQV21vCL5k5uAKBRXWuAXcThLDmFtjSoxHXh85uNoSVkREYjUNz9S8_08saaBb41OslOaUAujGAbx31FGu_Kf5JsXhQXYszeXbTffLFYZc3dmdZtw83i4vrG-2P17derwS2wjbDWyWstmjNcWRyxZm16yjzsUu3LG9vmf9QX-3qxfAtBRyiuMlV5KQ217-Z7fGgzg
```

复制 token 内容

### 3- 登陆页面

访问 https://10.125.144.36/

![443token](http://ov30w4cpi.bkt.clouddn.com/443token.png)

预览

![443index](http://ov30w4cpi.bkt.clouddn.com/443index.png)





 