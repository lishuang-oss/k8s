K8s会通过各种Controller来管理Pod的生命周期，为了满足不同的业务场景，K8s开发了Deployment、ReplicaSet、DaemonSet、StatefuleSet、Job、cronJob等多种Controller ，这里我们首先来学习下最常用的Deployment，这是我们生产中用的最多的一个controller，适合用来发布无状态应用.

我们先来运行一个Deployment实例：

```
# 创建一个deployment，引用nginx的服务镜像，这里的副本数量默认是1，nginx容器镜像用的是latest
# 在K8s新版本开始，对服务api进行了比较大的梳理，明确了各个api的具体职责，而不像以前旧版本那样混为一谈
# kubectl create deployment nginx --image=nginx
deployment.apps/nginx created

# 查看创建结果
# kubectl  get deployments.apps 
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   0/1     1            0           6s

# kubectl  get rs   # <-- 看下自动关联创建的副本集replicaset
NAME              DESIRED   CURRENT   READY   AGE
nginx-f89759699   1         1         0       10s

# kubectl get pod   # <-- 查看生成的pod，注意镜像下载需要一定时间，耐心等待，注意观察pod名称的f89759699，是不是和上面rs的一样，对了，因为这里的pod就是由上面的rs创建出来，为什么要设置这么一个环节呢，后面会以实例来演示
NAME                    READY   STATUS              RESTARTS   AGE
nginx-f89759699-26fzd   0/1     ContainerCreating   0          13s

# kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
nginx-f89759699-26fzd   1/1     Running   0          98s


# 扩容pod的数量
# kubectl scale deployment nginx --replicas=2
deployment.apps/nginx scaled

# 查看扩容后的pod结果
# kubectl get pod
NAME                    READY   STATUS              RESTARTS   AGE
nginx-f89759699-26fzd   1/1     Running             0          112s
nginx-f89759699-9s4dw   0/1     ContainerCreating   0          2s

# 具体看下pod是不是分散运行在不同的node上呢
# kubectl get pod -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
nginx-f89759699-26fzd   1/1     Running   0          45m   172.20.0.16   10.0.1.202   <none>           <none>
nginx-f89759699-9s4dw   1/1     Running   0          43m   172.20.1.14   10.0.1.201   <none>           <none>


# 接下来替换下这个deployment里面nginx的镜像版本，来讲解下为什么需要rs副本集呢，这个很重要哦
# 我们先看看目前nginx是哪个版本，随便输入一个错误的uri，页面就会打印出nginx的版本号了
curl 10.68.86.85/1
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.19.4</center>
</body>
</html>

# 根据输出可以看到版本号是nginx/1.19.4，这里利用上面提到的命令docker-tag来看下nginx有哪些其他的版本，然后我在里面挑选了1.9.9这个tag号
# 注意命令最后面的 `--record` 参数，这个在生产中作为资源创建更新用来回滚的重要标记，强烈建议在生产中操作时都加上这个参数
# kubectl set image deployment/nginx  nginx=nginx:1.9.9 --record 
deployment.apps/nginx image updated

# 观察下pod的信息，可以看到旧nginx的2个pod逐渐被新的pod一个一个的替换掉
# kubectl  get pod -w
NAME                    READY   STATUS              RESTARTS   AGE
nginx-89fc8d79d-4z876   1/1     Running             0          41s
nginx-89fc8d79d-jd78f   0/1     ContainerCreating   0          3s
nginx-f89759699-9cx7l   1/1     Running             0          4h53m

# 我们再看下nginx的rs，可以看到现在有两个了
# kubectl get rs
NAME              DESIRED   CURRENT   READY   AGE
nginx-89fc8d79d   2         2         2       9m6s
nginx-f89759699   0         0         0       6h15m

# 看下现在nginx的描述信息，我们来详细分析下这个过程
# kubectl  describe deployment nginx
Name:                   nginx
Namespace:              default
CreationTimestamp:      Tue, 24 Nov 2020 09:40:54 +0800
Labels:                 app=nginx
......
RollingUpdateStrategy:  25% max unavailable, 25% max surge  # 注意这里，这个就是用来控制rs新旧版本迭代更新的一个频率，滚动更新的副本总数最大值(以2的基数为例)：2+2*25%=2.5 -- > 3，可用副本数最大值(默认值两个都是25%)：2-2*25%=1.5 --> 2
......
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  21m   deployment-controller  Scaled up replica set nginx-89fc8d79d to 1  # 启动1个新版本的pod
  Normal  ScalingReplicaSet  20m   deployment-controller  Scaled down replica set nginx-f89759699 to 1 # 上面完成就释放掉一个旧版本的
  Normal  ScalingReplicaSet  20m   deployment-controller  Scaled up replica set nginx-89fc8d79d to 2 # 然后再启动1个新版本的pod
  Normal  ScalingReplicaSet  20m   deployment-controller  Scaled down replica set nginx-f89759699 to 0 # 释放掉最后1个旧的pod


# 回滚
# 还记得我们上面提到的 --record  参数嘛，这里它就会发挥很重要的作用了
# 这里还以nginx服务为例，先看下当前nginx的版本号

# curl  10.68.18.121/1         
<html>
<head><title>404 Not Found</title></head>
<body bgcolor="white">
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.9.9</center>
</body>
</html>

# 升级nginx的版本
#  kubectl set image deployments/nginx nginx=nginx:1.19.5 --record 

# 已经升级完成
# curl  10.68.18.121/1         
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.19.5</center>
</body>
</html>

# 这里假设是我们在发版服务的新版本，结果线上反馈版本有问题，需要马上回滚，看看在K8s上怎么操作吧
# 首先通过命令查看当前历史版本情况，只有接了`--record`参数的命令操作才会有详细的记录，这就是为什么在生产中操作一定得加上的原因了
# kubectl rollout history deployment nginx 
deployment.apps/nginx 
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployments/nginx nginx=nginx:1.9.9 --record=true
3         kubectl set image deployments/nginx nginx=nginx:1.19.5 --record=true

# 根据历史发布版本前面的阿拉伯数字序号来选择回滚版本，这里我们回到上个版本号，也就是选择2 ，执行命令如下：
# kubectl rollout undo deployment nginx --to-revision=2
deployment.apps/nginx rolled back

# 等一会pod更新完成后，看下结果已经回滚完成了，怎么样，在K8s操作就是这么简单：
# curl  10.68.18.121/1                                 
<html>
<head><title>404 Not Found</title></head>
<body bgcolor="white">
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.9.9</center>
</body>
</html>

# 可以看到现在最新版本号是4了，具体版本看操作的命令显示是1.9.9 ,并且先前回滚过的版本号2已经没有了，因为它已经变成4了
# kubectl rollout history deployment nginx             
deployment.apps/nginx 
REVISION  CHANGE-CAUSE
1         <none>
3         kubectl set image deployments/nginx nginx=nginx:1.19.5 --record=true
4         kubectl set image deployments/nginx nginx=nginx:1.9.9 --record=true
```

**Deployment很重要，我们这里再来回顾下整个部署过程，加深理解**

![第5关 K8s攻克作战攻略之二-Deployment](https://p26.toutiaoimg.com/origin/pgc-image/0649a6c6677a48498640c791d7cc59c7?from=pc)





**10.0.1.201** **10.0.1.202**

1. kubectl 发送部署请求到 API Server
2. API Server 通知 Controller Manager 创建一个 deployment 资源（scale扩容）
3. Scheduler 执行调度任务，将两个副本 Pod 分发到 10.0.1.201 和 10.0.1.202
4. 10.0.1.201 和 10.0.1.202 上的 kubelet在各自的节点上创建并运行 Pod
5. 升级deployment的nginx服务镜像

> 这里补充一下：
>
> 这些应用的配置和当前服务的状态信息都是保存在ETCD中，执行kubectl get pod等操作时API Server会从ETCD中读取这些数据
>
> calico会为每个pod分配一个ip，但要注意这个ip不是固定的，它会随着pod的重启而发生变化
>
> **附：Node管理**
>
> 禁止pod调度到该节点上
>
> kubectl cordon <node name>
>
> **驱逐该节点上的所有pod** kubectl drain <node name> 该命令会删除该节点上的所有Pod（DaemonSet除外），在其他node上重新启动它们，通常该节点需要维护时使用该命令。直接使用该命令会自动调用kubectl cordon <node>命令。当该节点维护完成，启动了kubelet后，再使用kubectl uncordon <node>即可将该节点添加到kubernetes集群中。

上面我们是用命令行来创建的deployment，但在生产中，很多时候，我们是直接写好yaml配置文件，再通过kubectl apply -f xxx.yaml来创建这个服务，我们现在用yaml配置文件的方式实现上面deployment服务的创建

需要注意的是，yaml文件格式缩进和python语法类似，对于缩进格式要求很严格，任何一处错误，都会造成无法创建，这里教大家一招实用的技巧来生成规范的yaml配置

```
# 这条命令是不是很眼熟，对了，这就是上面创建deployment的命令，我们在后面加上`--dry-run -o yaml`,--dry-run代表这条命令不会实际在K8s执行，-o yaml是会将试运行结果以yaml的格式打印出来，这样我们就能轻松获得yaml配置了

# kubectl create deployment nginx --image=nginx --dry-run -o yaml       
apiVersion: apps/v1     # <---  apiVersion 是当前配置格式的版本
kind: Deployment     #<--- kind 是要创建的资源类型，这里是 Deployment
metadata:        #<--- metadata 是该资源的元数据，name 是必需的元数据项
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:        #<---    spec 部分是该 Deployment 的规格说明
  replicas: 1        #<---  replicas 指明副本数量，默认为 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:        #<---   template 定义 Pod 的模板，这是配置文件的重要部分
    metadata:        #<---     metadata 定义 Pod 的元数据，至少要定义一个 label。label 的 key 和 value 可以任意指定
      creationTimestamp: null
      labels:
        app: nginx
    spec:           #<---  spec 描述 Pod 的规格，此部分定义 Pod 中每一个容器的属性，name 和 image 是必需的
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

我们这里用这个yaml文件来创建nginx的deployment试试，我们先删除掉先用命令行创建的nginx

```
# 在K8s上命令行删除一个资源直接用delete参数
# kubectl delete deployment nginx
deployment.apps "nginx" deleted

# 可以看到关联的rs副本集也被自动清空了
# kubectl  get rs
No resources found in default namespace.

# 相关的pod也没了
# kubectl get pod 
No resources found in default namespace.
```

生成nginx.yaml文件

```
# kubectl create deployment nginx --image=nginx --dry-run -o yaml > nginx.yaml
我们注意到执行上面命令时会有一条告警提示... --dry-run is deprecated and can be replaced with --dry-run=client.  ,虽然并不影响我们生成正常的yaml配置，但如果看着不爽可以按命令提示将--dry-run换成--dry-run=client
# 接着我们vim nginx.yaml，将replicas: 1的数量改成replicas: 2

# 开始创建，我们后面这类基于yaml文件来创建资源的命令统一都用apply了
# kubectl  apply -f nginx.yaml 
deployment.apps/nginx created

# 查看创建的资源，这个有个小技巧，同时查看多个资源可以用,分隔，这样一条命令就可以查看多个资源了
# kubectl get deployment,rs,pod
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   2/2     2            2           116s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-f89759699   2         2         2       116s

NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-f89759699-bzwd2   1/1     Running   0          116s
pod/nginx-f89759699-qlc8q   1/1     Running   0          116s
```

基于这两种资源创建的方式作个总结：

```
基于命令的方式：
1.简单直观快捷，上手快。
2.适合临时测试或实验。

基于配置文件的方式：
1.配置文件描述了 What，即应用最终要达到的状态。
2.配置文件提供了创建资源的模板，能够重复部署。
3.可以像管理代码一样管理部署。
4.适合正式的、跨环境的、规模化部署。
5.这种方式要求熟悉配置文件的语法，有一定难度。
```

