### 服务pod健康检测

利用liveness和readiness检测机制来设置更为精细的健康检测指标
liveness：存活指针，对pod进行存活检测，当pod发生异常进行重启。
readiness：就绪指针，当pod服务检测，当不满足检测条件，则从endpoint中剔除，把流量清除。

当然k8s还有默认检测机制：
每当启动一个容器时都会执行一个进程，此进程就是dockerfile的cmd和endpypoint当容器进程退出时返回状态吗非0，则会认为容器发生了故障，k8s就会根据restartpolicy来重启这个容器，以达到自愈效果。

```
# 先来生成一个pod的yaml配置文件，并对其进行相应修改
# kubectl run  busybox --image=busybox --dry-run=client -o yaml > testHealthz.yaml
# vim testHealthz.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox
  name: busybox
spec:
  containers:
  - image: busybox
    name: busybox
    resources: {}
    args:
    - /bin/sh
    - -c
    - sleep 10; exit 1       # 并添加pod运行指定脚本命令，模拟容器启动10秒后发生故障，退出状态码为1
  dnsPolicy: ClusterFirst
  restartPolicy: OnFailure # 将默认的Always修改为OnFailure
status: {}

执行配置创建pod
# kubectl apply -f testHealthz.yaml 
pod/busybox created

# 观察几分钟，利用-w 参数来持续监听pod的状态变化
# kubectl  get pod -w
NAME                     READY   STATUS              RESTARTS   AGE
busybox                  0/1     ContainerCreating   0          4s
busybox                  1/1     Running             0          6s
busybox                  0/1     Error               0          16s
busybox                  1/1     Running             1          22s
busybox                  0/1     Error               1          34s
busybox                  0/1     CrashLoopBackOff    1          47s
busybox                  1/1     Running             2          63s
busybox                  0/1     Error               2          73s
busybox                  0/1     CrashLoopBackOff    2          86s
busybox                  1/1     Running             3          109s
busybox                  0/1     Error               3          2m
busybox                  0/1     CrashLoopBackOff    3          2m15s
busybox                  1/1     Running             4          3m2s
busybox                  0/1     Error               4          3m12s
busybox                  0/1     CrashLoopBackOff    4          3m23s
busybox                  1/1     Running             5          4m52s
busybox                  0/1     Error               5          5m2s
busybox                  0/1     CrashLoopBackOff    5          5m14s

上面可以看到这个测试pod被重启了5次，然而服务始终正常不了，就会保持在CrashLoopBackOff了，等待运维人员来进行下一步错误排查
注：kubelet会以指数级的退避延迟（10s，20s，40s等）重新启动它们，上限为5分钟
这里我们是人为模拟服务故障来进行的测试，在实际生产工作中，对于业务服务，我们如何利用这种重启容器来恢复的机制来配置业务服务呢，答案是`liveness`检测
```



###### liveness

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness
spec:
  restartPolicy: OnFailure
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 10   # 容器启动 10 秒之后开始检测
      periodSeconds: 5          # 每隔 5 秒再检测一次

#启动进程首先创建文件/tmp/healthy,30s后删除，如果/tmp/healthy文件存在，则认为容器处于正常状态，反则发生故障。

检测的方法是：通过 cat 命令检查 /tmp/healthy 文件是否存在。如果命令执行成功，返回值为零，K8s 则认为本次 Liveness 检测成功；如果命令返回值非零，本次 Liveness 检测失败。

initialDelaySeconds: 10 指定容器启动 10 之后开始执行 Liveness 检测，我们一般会根据应用启动的准备时间来设置。比如某个应用正常启动要花 30 秒，那么 initialDelaySeconds 的值就应该大于 30。

periodSeconds: 5 指定每 5 秒执行一次 Liveness 检测。K8s 如果连续执行 3 次 Liveness 检测均失败，则会杀掉并重启容器。

从配置文件可知，最开始的 30 秒，/tmp/healthy 存在，cat 命令返回 0，Liveness 检测成功，这段时间 kubectl describe pod liveness 的 Events部分会显示正常的日志

接着判断失败后容器开始容器
[root@master Healthz]# kubectl get pod liveness  -w
NAME       READY   STATUS    RESTARTS   AGE
liveness   1/1     Running   0          17s
liveness   1/1     Running   1          82s
liveness   1/1     Running   2          2m38s
liveness   1/1     Running   3          3m52s
liveness   1/1     Running   4          5m7s
liveness   1/1     Running   5          6m22s

我们看一下pod事件就一目了然：
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  7m25s                  default-scheduler  Successfully assigned default/liveness to node2
  Normal   Pulled     7m18s                  kubelet            Successfully pulled image "busybox" in 6.888739745s
  Normal   Pulled     6m3s                   kubelet            Successfully pulled image "busybox" in 3.582767726s
  Normal   Created    4m48s (x3 over 7m18s)  kubelet            Created container liveness
  Normal   Started    4m48s (x3 over 7m18s)  kubelet            Started container liveness
  Normal   Pulled     4m48s                  kubelet            Successfully pulled image "busybox" in 3.714031394s
  Warning  Unhealthy  4m7s (x9 over 6m47s)   kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
  Normal   Killing    4m7s (x3 over 6m37s)   kubelet            Container liveness failed liveness probe, will be restarted
  Normal   Pulling    2m22s (x5 over 7m25s)  kubelet            Pulling image "busybox"
```

###### readiness

```
我们可以通过readiness检测来告诉k8s什么时候可以将pod加入到服务service的负载均衡池中，对外提供服务，这个在生产场景服务新版本时非常重要，当我们上线的新版本发生程序错误时，readiness会通过检测发布，从而不导入流量到pod内，因为有时候我们需要保留服务出错的现场来查询日志，定位问题，告知开发来修复程序。
```

Readiness 检测的配置语法与 Liveness 检测完全一样，下面是个例子：

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    test: readiness
  name: readiness
spec:
  containers:
  - name: readiness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30;rm -f /tmp/healthy;sleep 600
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 10
      periodSeconds: 5
```

我们来看一下运行pod之后会有哪些变化

```
# kubectl apply -f readiness.yaml 
pod/liveness created

# 观察，在刚开始创建时，文件并没有被删除，所以检测一切正常
# kubectl  get pod
NAME                     READY   STATUS    RESTARTS   AGE
liveness                 1/1     Running   0          50s

# 然后35秒后，文件被删除，这个时候READY状态就会发生变化，K8s会断开Service到pod的流量
# kubectl  describe pod liveness 
......
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  56s               default-scheduler  Successfully assigned default/liveness to 10.0.1.203
  Normal   Pulling    56s               kubelet            Pulling image "busybox"
  Normal   Pulled     40s               kubelet            Successfully pulled image "busybox"
  Normal   Created    40s               kubelet            Created container liveness
  Normal   Started    40s               kubelet            Started container liveness
  Warning  Unhealthy  5s (x2 over 10s)  kubelet            Readiness probe failed: cat: can't open '/tmp/healthy': No such file or directory

# 可以看到pod的流量被断开，这时候即使服务出错，对外界来说也是感知不到的，这时候我们运维人员就可以进行故障排查了
# kubectl  get pod
NAME                     READY   STATUS    RESTARTS   AGE
liveness                 0/1     Running   0          61s
```

###### liveness和readiness的三种使用方式

```
 readinessProbe:  # 定义只有http检测容器6222端口请求返回是 200-400，则接收下面的Service web-svc 的请求
            httpGet:
              scheme: HTTP
              path: /check
              port: 6222
            initialDelaySeconds: 10   # 容器启动 10 秒之后开始探测，注意观察下g1的启动成功时间
            periodSeconds: 5          # 每隔 5 秒再探测一次
            timeoutSeconds: 5         # http检测请求的超时时间
            successThreshold: 1       # 检测到有1次成功则认为服务是`就绪`
            failureThreshold: 3       # 检测到有3次失败则认为服务是`未就绪`
          livenessProbe:  # 定义只有http检测容器6222端口请求返回是 200-400，否则就重启pod
            httpGet:
              scheme: HTTP
              path: /check
              port: 6222
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 3

#-----------------------------------------
            
          readinessProbe:
            exec:
              command:
              - sh
              - -c
              - "redis-cli ping"
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 1
            successThreshold: 1
            failureThreshold: 3
          livenessProbe:
            exec:
              command:
              - sh
              - -c
              - "redis-cli ping"
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 3

#-----------------------------------------

        readinessProbe:
          tcpSocket:
            port: 9092
          initialDelaySeconds: 15
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 9092
          initialDelaySeconds: 15
          periodSeconds: 10
```



###### 两者区别比较

```
Liveness 检测和 Readiness 检测是两种 Health Check 机制，如果不特意配置，Kubernetes 将对两种检测采取相同的默认行为，即通过判断容器启动进程的返回值是否为零来判断检测是否成功。

两种检测的配置方法完全一样，支持的配置参数也一样。不同之处在于检测失败后的行为：Liveness 检测是重启容器；Readiness 检测则是将容器设置为不可用，不接收 Service 转发的请求。

Liveness 检测和 Readiness 检测是独立执行的，二者之间没有依赖，所以可以单独使用，也可以同时使用。用 Liveness 检测判断容器是否需要重启以实现自愈；用 Readiness 检测判断容器是否已经准备好对外提供服务。
```

