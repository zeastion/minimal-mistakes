Kubernetes 管理业务更新

## || Rolling Update

滚动更新开始至更新部分副本，确保成功后再更新更多副本，最终完成所有副本更新。
整个过程中，始终有副本在运行，保证了业务不中断。

```yaml
# vim rollingupdatehttp.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: ruhttpd
spec:
  replicas: 5
  template:
    metadata:
      labels:
        run: ruhttpd
    spec:
      containers:
      - name: ruhttpd
        image: httpd:2.2.31
        ports:
        - containerPort: 80
```
```bash
# kubectl apply -f rollingupdatehttp.yaml
deployment "ruhttpd" created

# kubectl get deployment ruhttpd -o wide
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINER(S)   IMAGE(S)       SELECTOR
ruhttpd   5         5         5            5           26s       ruhttpd        httpd:2.2.31   run=ruhttpd

# kubectl get replicaset -o wide
NAME                 DESIRED   CURRENT   READY     AGE       CONTAINER(S)   IMAGE(S)       SELECTOR
ruhttpd-3766725260   5         5         5         39s       ruhttpd        httpd:2.2.31   pod-template-hash=3766725260,run=ruhttpd

# kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
ruhttpd-3766725260-555mj   1/1       Running   0          6m
ruhttpd-3766725260-8r9bq   1/1       Running   0          6m
ruhttpd-3766725260-gz1rx   1/1       Running   0          6m
ruhttpd-3766725260-mskpb   1/1       Running   0          6m
ruhttpd-3766725260-xvhbx   1/1       Running   0          6m
```

整个过程包括

- 创建 Deployment 'ruhttpd'
- 创建 ReplicaSet 'ruhttpd-3766725260'
- 创建 5 个 Pod （镜像为 'httpd:2.2.31'）

将配置文件 'rollingupdatehttp.yaml' 中 'image: httpd:2.2.31' 替换为 'image: httpd:2.2.32'

再次应用

```bash
# kubectl apply -f rollingupdatehttp.yaml
deployment "ruhttpd" configured

# kubectl get deployment ruhttpd -o wide
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINER(S)   IMAGE(S)       SELECTOR
ruhttpd   5         7         3            4           14m       ruhttpd        httpd:2.2.32   run=ruhttpd

# kubectl get replicaset -o wide
NAME                 DESIRED   CURRENT   READY     AGE       CONTAINER(S)   IMAGE(S)       SELECTOR
ruhttpd-1380764847   5         5         5         29s       ruhttpd        httpd:2.2.32   pod-template-hash=1380764847,run=ruhttpd
ruhttpd-3766725260   0         0         0         14m       ruhttpd        httpd:2.2.31   pod-template-hash=3766725260,run=ruhttpd

# kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
ruhttpd-1380764847-4rn73   1/1       Running   0          13s
ruhttpd-1380764847-f7p40   1/1       Running   0          34s
ruhttpd-1380764847-fx1b3   1/1       Running   0          13s
ruhttpd-1380764847-hshkt   1/1       Running   0          34s
ruhttpd-1380764847-mcb3v   1/1       Running   0          34s
```

创建了新的 ReplicaSet 'ruhttpd-1380764847'，镜像为 'httpd:2.2.32'，创建了 5 个新的 Pod 替换了原 ReplicaSet 'ruhttpd-3766725260' 管理的 Pod

具体过程

```bash
# kubectl describe deployment ruhttpd
...
Events:
  FirstSeen     LastSeen        Count   From                    SubObjectPath   Type            Reason                  Message
  ---------     --------        -----   ----                    -------------   --------        ------                  -------
  22m           22m             1       deployment-controller                   Normal          ScalingReplicaSet       Scaled up replica set ruhttpd-3766725260 to 5
  8m            8m              1       deployment-controller                   Normal          ScalingReplicaSet       Scaled up replica set ruhttpd-1380764847 to 2
  8m            8m              1       deployment-controller                   Normal          ScalingReplicaSet       Scaled down replica set ruhttpd-3766725260 to 4
  8m            8m              1       deployment-controller                   Normal          ScalingReplicaSet       Scaled up replica set ruhttpd-1380764847 to 3
  8m            8m              1       deployment-controller                   Normal          ScalingReplicaSet       Scaled down replica set ruhttpd-3766725260 to 3
  8m            8m              1       deployment-controller                   Normal          ScalingReplicaSet       Scaled up replica set ruhttpd-1380764847 to 4
  8m            8m              1       deployment-controller                   Normal          ScalingReplicaSet       Scaled down replica set ruhttpd-3766725260 to 2
  8m            8m              1       deployment-controller                   Normal          ScalingReplicaSet       Scaled up replica set ruhttpd-1380764847 to 5
  8m            8m              1       deployment-controller                   Normal          ScalingReplicaSet       Scaled down replica set ruhttpd-3766725260 to 1
  8m            8m              1       deployment-controller                   Normal          ScalingReplicaSet       Scaled down replica set ruhttpd-3766725260 to 0
```

两个 ReplicaSet 每次替换一个 Pod ，还可以通过参数 'maxSurge' & 'maxUnavailable' 精细控制更替的 Pod 数量


## || 回滚

'revision' 保存每次应用更新的配置信息，从而可以回滚到某个特定版次

默认情况下 Kubernetes 会保留最近几个 revision 。可以在 Deployment 配置文件中，使用 'revisionHistoryLimit' 属性控制 revision 数量

写三个配置文件，分别使用不同版本镜像，对 Deployment 升级

```yaml
# vim httpd.v1.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: httpd
spec:
  revisionHistoryLimit: 10
  replicas: 3
  template:
    metadata:
      labels:
        run: httpd
    spec:
      containers:
      - name: httpd
        image: httpd:2.4.16
        ports:
        - containerPort: 80

# vim httpd.v2.yaml
...
        image: httpd:2.4.17
...

# vim httpd.v3.yaml
...
        image: httpd:2.4.18
...
```

部署并更新应用，使用 '--record' 参数记录版本命令，即下面的 'CHANGE-CAUSE'

```bash
# kubectl apply -f httpd.v1.yaml --record
deployment "httpd" created

# kubectl get deployment httpd -o wide
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINER(S)   IMAGE(S)       SELECTOR
httpd     3         3         3            0           15s       httpd          httpd:2.4.16   run=httpd

# kubectl apply -f httpd.v2.yaml --record
deployment "httpd" configured

# kubectl get deployment httpd -o wide
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINER(S)   IMAGE(S)       SELECTOR
httpd     3         4         1            3           30s       httpd          httpd:2.4.17   run=httpd

# kubectl apply -f httpd.v3.yaml --record
deployment "httpd" configured

# kubectl get deployment httpd -o wide
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINER(S)   IMAGE(S)       SELECTOR
httpd     3         4         1            3           44s       httpd          httpd:2.4.18   run=httpd
```

查看 revision 历史记录

```bash
# kubectl rollout history deployment httpd
deployments "httpd"
REVISION        CHANGE-CAUSE
1               kubectl apply --filename=httpd.v1.yaml --record=true
2               kubectl apply --filename=httpd.v2.yaml --record=true
3               kubectl apply --filename=httpd.v3.yaml --record=true
```

回退到特定版本

```bash
# kubectl rollout undo deployment httpd --to-revision=2
deployment "httpd" rolled back

# kubectl get deployment httpd -o wide
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINER(S)   IMAGE(S)       SELECTOR
httpd     3         3         3            3           52m       httpd          httpd:2.4.17   run=httpd
```

再查看历史版本记录

```bash
# kubectl rollout history deployment httpd
deployments "httpd"
REVISION        CHANGE-CAUSE
1               kubectl apply --filename=httpd.v1.yaml --record=true
3               kubectl apply --filename=httpd.v3.yaml --record=true
4               kubectl apply --filename=httpd.v2.yaml --record=true
```


## || 健康检查

K8s 默认的健康检查机制是侦测容器进程结束时的返回码，若非零则认为故障，根据 'restartPolicy' 重启容器

但这种机制的缺陷在于，某些时候业务故障不一定是容器进程退出，还有系统超载，资源死锁等情况，此时需要更细致的探测方法发现问题，并快速重启容器处理故障，或将故障 Pod 排出 Service 负载均衡。

### Liveness 探测

可以通过自定义的 'Liveness' 规则，更细致判断容器业务是否健康

#### command


```yaml
# vim livenesstest.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: livenesstest
  name: livenesstest
spec:
  restartPolicy: OnFailure
  containers:
  - name: livenesstest
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 10
      periodSeconds: 5
```
```bash
# kubectl apply -f livenesstest.yaml
pod "livenesstest" created

# kubectl describe pod livenesstest
...
Events:
  FirstSeen     LastSeen        Count   From                            SubObjectPath                   Type            Reason    Message
  ---------     --------        -----   ----                            -------------                   --------        ------    -------
  15m           15m             1       kubelet, f16-7-kubernetes02                                     Normal          SuccessfulMountVolume      MountVolume.SetUp succeeded for volume "default-token-c4m8h"
  13m           13m             1       default-scheduler                                               Normal          Scheduled Successfully assigned livenesstest to f16-7-kubernetes02
  14m           6m              7       kubelet, f16-7-kubernetes02     spec.containers{livenesstest}   Normal          Killing   Killing container with id docker://livenesstest:pod "livenesstest_default(ca69e447-1a98-11e8-a441-f44c7f1da78a)" container "livenesstest" is unhealthy, it will be killed and re-created.
  6m            3m              15      kubelet, f16-7-kubernetes02     spec.containers{livenesstest}   Warning         BackOff   Back-off restarting failed container
  6m            3m              15      kubelet, f16-7-kubernetes02                                     Warning         FailedSyncError syncing pod
  15m           3m              8       kubelet, f16-7-kubernetes02     spec.containers{livenesstest}   Normal          Pulling   pulling image "busybox"
  15m           3m              8       kubelet, f16-7-kubernetes02     spec.containers{livenesstest}   Normal          Started   Started container
  15m           3m              8       kubelet, f16-7-kubernetes02     spec.containers{livenesstest}   Normal          Pulled    Successfully pulled image "busybox"
  15m           3m              8       kubelet, f16-7-kubernetes02     spec.containers{livenesstest}   Normal          Created   Created container
  14m           2m              24      kubelet, f16-7-kubernetes02     spec.containers{livenesstest}   Warning         Unhealthy Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory'

# kubectl get pod livenesstest
NAME           READY     STATUS    RESTARTS   AGE
livenesstest   1/1       Running   10         27m
```

如上，'livenessProbe' 部分是自定义的 Liveness 探测规则：

- 探测命令是 'cat /tmp/healthy' ，其执行的返回值为零则认为探测成功，否则异常
- 参数 'initialDelaySeconds： 10' 指定容器启动 10s 后开始执行探测，给业务一个容器启动的准备时间
- 参数 'periodSeconds: 5' 指定每 5s 执行一次探测，若连续三次 Liveness 探测失败，则杀掉并重启容器

#### HTTP request

使用 liveness 做 HTTP 请求探测

```yaml
# vim liveness-http.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/liveness
    args:
    - /server
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 5
      periodSeconds: 3
```

kubelet 等待五秒后开始每隔三秒向容器发送一个 HTTP GET 请求，目标端口 '8080'，路径 '/healthz'。如果返回值在 200 ~ 400 之间，则认为健康，其他返回码则识别为失败，重建容器。

```bash
# kubectl apply -f liveness-http.yaml
pod "liveness-http" created

# kubectl get pod liveness-http
NAME            READY     STATUS    RESTARTS   AGE
liveness-http   1/1       Running   0          15s

# kubectl describe pod liveness-http
...
Events:
  FirstSeen     LastSeen        Count   From                    SubObjectPath                   Type            Reason                     Message
  ---------     --------        -----   ----                    -------------                   --------        ------                     -------
  54s           54s             1       default-scheduler                                       Normal          Scheduled          Successfully assigned liveness-http to k8s03
  54s           54s             1       kubelet, k8s03                                          Normal          SuccessfulMountVolume      MountVolume.SetUp succeeded for volume "default-token-sqbcl"
  47s           21s             2       kubelet, k8s03          spec.containers{liveness}       Normal          Pulled                     Successfully pulled image "k8s.gcr.io/liveness"
  46s           21s             2       kubelet, k8s03          spec.containers{liveness}       Normal          Created                    Created container
  46s           20s             2       kubelet, k8s03          spec.containers{liveness}       Normal          Started                    Started container
  53s           3s              3       kubelet, k8s03          spec.containers{liveness}       Normal          Pulling                    pulling image "k8s.gcr.io/liveness"
  33s           3s              6       kubelet, k8s03          spec.containers{liveness}       Warning         Unhealthy          Liveness probe failed: HTTP probe failed with statuscode: 500
  27s           3s              2       kubelet, k8s03          spec.containers{liveness}       Normal          Killing                    Killing container with id docker://liveness:pod "liveness-http_default(4d801590-1d00-11e8-a384-000c29c533ff)" container "liveness" is unhealthy, it will be killed and re-created.
```

镜像 'k8s.gcr.io/liveness' 代码中，前十秒返回 200，之后返回 500。

#### TCP

第三种方式是 TCP Socket，kubelet 会尝试在容器特定端口打开一个 socket，如果能建立连接则判定业务健康。结合下面 Readiness 一起实验。

### Readiness 探测

Readiness 探测用于判断容器是否达到满足对外提供服务的条件，探测成功则该容器加入 Service，否则设为不可用，不接受 Service 转发请求

```yaml
# vim readinesstest.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: readinesstest
  name: readinesstest
spec:
  restartPolicy: OnFailure
  containers:
  - name: readinesstest
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 10
      periodSeconds: 5
```

查看实施过程

```bash
# kubectl apply -f readinesstest.yaml
pod "readinesstest" created

# kubectl get pod readinesstest
NAME            READY     STATUS    RESTARTS   AGE
readinesstest   0/1       Running   0          8s

# kubectl get pod readinesstest
NAME            READY     STATUS    RESTARTS   AGE
readinesstest   1/1       Running   0          18s

# kubectl get pod readinesstest
NAME            READY     STATUS    RESTARTS   AGE
readinesstest   0/1       Running   0          51s
```

分为三个阶段

- 刚创建时，'READY' 状态为不可用
- 15 秒后，第一次探测结果成功，设置 'READY' 为可用
- 30 秒后，文件被删除，经过连续 3 次 Readiness 探测均失败，集群将 'READY' 状态设为不可用

详细过程

```bash
# kubectl describe pod readiness
...
Events:
  FirstSeen     LastSeen        Count   From                            SubObjectPath                   Type            Reason    Message
  ---------     --------        -----   ----                            -------------                   --------        ------    -------
  11m           11m             1       kubelet, f16-8-kubernetes03                                     Normal          SuccessfulMountVolume      MountVolume.SetUp succeeded for volume "default-token-c4m8h"
  11m           11m             1       kubelet, f16-8-kubernetes03     spec.containers{readinesstest}  Normal          Pulling   pulling image "busybox"
  11m           11m             1       kubelet, f16-8-kubernetes03     spec.containers{readinesstest}  Normal          Pulled    Successfully pulled image "busybox"
  11m           11m             1       kubelet, f16-8-kubernetes03     spec.containers{readinesstest}  Normal          Created   Created container
  11m           11m             1       kubelet, f16-8-kubernetes03     spec.containers{readinesstest}  Normal          Started   Started container
  10m           10m             1       default-scheduler                                               Normal          Scheduled Successfully assigned readinesstest to f16-8-kubernetes03
  10m           53s             120     kubelet, f16-8-kubernetes03     spec.containers{readinesstest}  Warning         Unhealthy Readiness probe failed: cat: can't open '/tmp/healthy': No such file or directory'
```

相比与 Liveness 探测后根据成功与否判断是否重启容器，实现自愈； Readiness 探测根据结果设置 Pod 'READY' 状态，开关对外服务接口。

因此，可以将 Readiness 当作 Pod 加入 Service 对外服务前的 ‘把关人’：例如业务扩展新副本的场景下，新拉起的副本启动后还需要进行加载数据、连接数据库等初始化工作，这个缓冲时间可以使用 Readiness 探测来判定容器是否就绪，避免请求落到还没准备好的 Pod 上。

将 liveness 和 readiness 结合使用，做 TCP Socket 检测

```yaml
# vim test-tcp.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-tcp
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: k8s.gcr.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

```bash
# kubectl apply -f test-tcp.yaml
pod "test-tcp" created

# kubectl get pod test-tcp
NAME       READY     STATUS    RESTARTS   AGE
test-tcp   1/1       Running   0          3m

# kubectl describe pod test-tcp
...
    Liveness:           tcp-socket :8080 delay=15s timeout=1s period=20s #success=1 #failure=3
    Readiness:          tcp-socket :8080 delay=5s timeout=1s period=10s #success=1 #failure=3
...
Events:
  FirstSeen     LastSeen        Count   From                    SubObjectPath                   Type            Reason                     Message
  ---------     --------        -----   ----                    -------------                   --------        ------                     -------
  3m            3m              1       default-scheduler                                       Normal          Scheduled          Successfully assigned test-tcp to k8s03
  3m            3m              1       kubelet, k8s03                                          Normal          SuccessfulMountVolume      MountVolume.SetUp succeeded for volume "default-token-sqbcl"
  3m            3m              1       kubelet, k8s03          spec.containers{goproxy}        Normal          Pulled                     Container image "k8s.gcr.io/goproxy:0.1" already present on machine
  3m            3m              1       kubelet, k8s03          spec.containers{goproxy}        Normal          Created                    Created container
  3m            3m              1       kubelet, k8s03          spec.containers{goproxy}        Normal          Started                    Started container
```

## || Liveness & Readiness 在 Rolling Update 场景中实践

在业务更新过程中，新镜像可能存在潜在的未知故障，原副本全部被问题副本替换，会造成业务不可用的严重后果。使用自定义的健康检查，新副本只有通过 Readiness 探测，才会被添加到 Service，若未通过，则现有副本不会被全部替换，保障业务正常运行。

```yaml
# vim app.v1.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 10
  template:
    metadata:
      labels:
        run: app
    spec:
      containers:
      - name: app
        image: busybox
        args:
        - /bin/sh
        - -c
        - sleep 10; touch /tmp/healthy; sleep 30000
        readinessProbe:
          exec:
            command:
              - cat
              - /tmp/healthy
          initialDelaySeconds: 10
          periodSeconds: 5
```

```bash
# kubectl apply -f app.v1.yaml --record
deployment "app" created

# kubectl get deployment app
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
app       10        10        10           10          1m

# kubectl get pod
NAME                                   READY     STATUS    RESTARTS   AGE
app-2780995820-07n3v                   1/1       Running   0          1m
app-2780995820-45xmg                   1/1       Running   0          1m
app-2780995820-4w268                   1/1       Running   0          1m
app-2780995820-8ctnp                   1/1       Running   0          1m
app-2780995820-gdn0v                   1/1       Running   0          1m
app-2780995820-h0h3f                   1/1       Running   0          1m
app-2780995820-rckl7                   1/1       Running   0          1m
app-2780995820-tbcgn                   1/1       Running   0          1m
app-2780995820-vd5bf                   1/1       Running   0          1m
app-2780995820-wjjrc                   1/1       Running   0          1m
```

新的业务配置如下

```yaml
# vim app.v2.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 10
  template:
    metadata:
      labels:
        run: app
    spec:
      containers:
      - name: app
        image: busybox
        args:
        - /bin/sh
        - -c
        - sleep 3000
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 10
          periodSeconds: 5
```

进行更新

```bash
# kubectl apply -f app.v2.yaml --record
deployment "app" configured

# kubectl get deployment app
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
app       10        13        5            8           11m

# kubectl get pod
NAME                                   READY     STATUS    RESTARTS   AGE
app-2780995820-07n3v                   1/1       Running   0          11m
app-2780995820-45xmg                   1/1       Running   0          11m
app-2780995820-4w268                   1/1       Running   0          11m
app-2780995820-8ctnp                   1/1       Running   0          11m
app-2780995820-h0h3f                   1/1       Running   0          11m
app-2780995820-rckl7                   1/1       Running   0          11m
app-2780995820-tbcgn                   1/1       Running   0          11m
app-2780995820-wjjrc                   1/1       Running   0          11m
app-3350497563-7gdkq                   0/1       Running   0          1m
app-3350497563-b3x5q                   0/1       Running   0          1m
app-3350497563-b8b62                   0/1       Running   0          1m
app-3350497563-dkvnf                   0/1       Running   0          1m
app-3350497563-tp4jr                   0/1       Running   0          1m
```

集群拉起了 5 个新副本，但 'READY' 状态异常，原副本数降为 8 个。

在 deployment 输出状态中：

- DESIRED 10 表示期望状态为 10 个 'READY' 的副本；
- CURRENT 13 表示当前副本总数为 13；
- UP-TO-DATE 5 表示已完成更新的副本数，即 5 个新增副本；
- AVAILABLE 8 表示当前处于 'READY' 状态的副本数，即 8 个旧副本。

```bash
# kubectl describe deployment app
...
Replicas:               10 desired | 5 updated | 13 total | 8 available | 5 unavailable
...
RollingUpdateStrategy:  25% max unavailable, 25% max surge
...
OldReplicaSets: app-2780995820 (8/8 replicas created)
NewReplicaSet:  app-3350497563 (5/5 replicas created)
...
```

这个场景中，新副本无法通过 Readiness 探测，所以会一直维持这样的状态，虽然有两个旧副本被销毁，但健康检查及时终止更新，避免了错误的新副本对现有业务的影响。

回滚到原来的稳定状态

```bash
# kubectl rollout history deployment app
deployments "app"
REVISION        CHANGE-CAUSE
1               kubectl apply --filename=app.v1.yaml --record=true
2               kubectl apply --filename=app.v2.yaml --record=true

# kubectl rollout undo deployment app --to-revision=1
deployment "app" rolled back

# kubectl get deployment app
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
app       10        10        10           10          16h

# kubectl get pod
NAME                                   READY     STATUS    RESTARTS   AGE
app-2780995820-07n3v                   1/1       Running   1          16h
app-2780995820-45xmg                   1/1       Running   1          16h
app-2780995820-4w268                   1/1       Running   1          16h
app-2780995820-8ctnp                   1/1       Running   1          16h
app-2780995820-h0h3f                   1/1       Running   1          16h
app-2780995820-lhlcx                   1/1       Running   0          1m
app-2780995820-rckl7                   1/1       Running   1          16h
app-2780995820-tbcgn                   1/1       Running   1          16h
app-2780995820-wjjrc                   1/1       Running   1          16h
app-2780995820-x2xr0                   1/1       Running   0          1m
```

在更新过程中，一次替换掉的旧副本数和一次增加的新副本数，可以使用 'maxSurge'、'maxUnavailable' 参数，根据业务场景自定义。

<b>maxSurge</b>

此参数控制滚动更新过程中，副本总数超过 'DESIRED' 的上限。可以是具体的整数，也可以是百分比，向上取整（默认为 25%）。

如上例中， DESIRED = 10，副本总数上限为 roundUp(10 + 10 * 25%) = 13 = 'CURRENT'

<b>maxUnavailable</b>

此参数控制滚动更新过程中，不可用的副本占 'DESIRED' 的最大比例。可以是整数，也可以是百本比，向下取整（默认为 25%）。

如上例中， DESIRED = 10，可用副本至少为 10 - roundDown(10 * 25%) = 8 = 'AVAILABLE'

> maxSurge 越大，初始创建的新副本数量就越多，maxUnavailable 越大，初始销毁的旧副本数量就越多

```yaml
# vim app.v3.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: app
spec:
  strategy:
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 50%
  replicas: 10
  template:
    metadata:
      labels:
        run: app
    spec:
      containers:
      - name: app
        image: busybox
        args:
        - /bin/sh
        - -c
        - sleep 3000
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 10
          periodSeconds: 5
```

```bash
# kubectl apply -f app.v3.yaml --record
deployment "app" configured

# kubectl get deployment app
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
app       10        15        10           5           17h

# kubectl get pods
NAME                                   READY     STATUS    RESTARTS   AGE
app-2780995820-45xmg                   1/1       Running   2          17h
app-2780995820-4w268                   1/1       Running   2          17h
app-2780995820-8ctnp                   1/1       Running   2          17h
app-2780995820-tbcgn                   1/1       Running   2          17h
app-2780995820-wjjrc                   1/1       Running   2          17h
app-3350497563-55twm                   0/1       Running   1          52m
app-3350497563-8d9s8                   0/1       Running   1          52m
app-3350497563-8mlsq                   0/1       Running   1          52m
app-3350497563-9ztg1                   0/1       Running   1          52m
app-3350497563-d53wt                   0/1       Running   1          52m
app-3350497563-djft4                   0/1       Running   1          52m
app-3350497563-h2xns                   0/1       Running   1          52m
app-3350497563-hn5tq                   0/1       Running   1          52m
app-3350497563-pp340                   0/1       Running   1          52m
app-3350497563-rdzvm                   0/1       Running   1          52m

# kubectl describe deployment app
...
Replicas:               10 desired | 10 updated | 15 total | 5 available | 10 unavailable
...
RollingUpdateStrategy:  50% max unavailable, 50% max surge
...
OldReplicaSets: app-2780995820 (5/5 replicas created)
NewReplicaSet:  app-3350497563 (10/10 replicas created)
...

# kubectl rollout history deployment app
deployments "app"
REVISION        CHANGE-CAUSE
3               kubectl apply --filename=app.v1.yaml --record=true
4               kubectl apply --filename=app.v3.yaml --record=true

# kubectl rollout undo deployment app --to-revision=3
deployment "app" rolled back
```
