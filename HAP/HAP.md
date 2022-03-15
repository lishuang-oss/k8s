我们知道，初始Pod的数量是可以设置的，同时业务也分流量高峰和低峰，那么怎么即能不过多的占用K8s的资源，又能在服务高峰时自动扩容pod的数量呢，在K8s上的答案是Horizontal Pod Autoscaling，简称HPA 自动水平伸缩，这里只以我们常用的CPU计算型服务来作为HPA的测试，这基本满足了大部分业务服务需求，其它如vpa纵向扩容，还有基于业务qps等特殊指标扩容这个在后面计划会以独立高级番外篇来作教程。

自动水平伸缩，是指运行在k8s上的应用负载(POD)，可以根据资源使用率进行自动扩容、缩容，它依赖metrics-server服务pod使用资源指标收集；我们知道应用的资源使用率通常都有高峰和低谷，所以k8s的HPA特性应运而生；它也是最能体现区别于传统运维的优势之一，不仅能够弹性伸缩，而且完全自动化！

我们在生产中通常用得最多的就是基于服务pod的cpu使用率metrics来自动扩容pod数量，下面来以生产的标准来实战测试下（注意：使用HPA前我们要确保K8s集群的dns服务和metrics服务是正常运行的，并且我们所创建的服务需要配置指标分配）

```
# pod内资源分配的配置格式如下：
# 默认可以只配置requests，但根据生产中的经验，建议把limits资源限制也加上，因为对K8s来说，只有这两个都配置了且配置的值都要一样，这个pod资源的优先级才是最高的，在node资源不够的情况下，首先是把没有任何资源分配配置的pod资源给干掉，其次是只配置了requests的，最后才是两个都配置的情况，仔细品品
      resources:
        limits:   # 限制单个pod最多能使用1核（1000m 毫核）cpu以及2G内存
          cpu: "1"
          memory: 2Gi
        requests: # 保证这个pod初始就能分配这么多资源
          cpu: "1"
          memory: 2Gi
```

我们先不做上面配置的改动，看看直接创建hpa会产生什么情况：

```
# 为deployment资源web创建hpa，pod数量上限3个，最低1个，在pod平均CPU达到50%后开始扩容
kubectl  autoscale deployment web --max=3 --min=1 --cpu-percent=50

#过一会看下这个hpa资源的描述(截取这下面一部分)
# 下面提示说到，HPA缺少最小资源分配的request参数
Conditions:
  Type           Status  Reason                   Message
  ----           ------  ------                   -------
  AbleToScale    True    SucceededGetScale        the HPA controller was able to get the target's current scale
  ScalingActive  False   FailedGetResourceMetric  the HPA was unable to compute the replica count: missing request for cpu
Events:
  Type     Reason                        Age                     From                       Message
  ----     ------                        ----                    ----                       -------
  Warning  FailedComputeMetricsReplicas  3m46s (x12 over 6m33s)  horizontal-pod-autoscaler  invalid metrics (1 invalid out of 1), first error is: failed to get cpu utilization: missing request for cpu
  Warning  FailedGetResourceMetric       89s (x21 over 6m33s)    horizontal-pod-autoscaler  missing request for cpu
```

我们现在以上面创建的deployment资源web来实践下hpa的效果，首先用我们学到的方法导出web的yaml配置，并增加资源分配配置增加：

```
# cat web.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: web
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - image: nginx
        name: nginx
        resources:
          limits:   # 因为我这里是测试环境，所以这里CPU只分配50毫核（0.05核CPU）和20M的内存
            cpu: "50m"
            memory: 20Mi
          requests: # 保证这个pod初始就能分配这么多资源
            cpu: "50m"
            memory: 20Mi
```

更新web资源：

```
# kubectl  apply -f web.yaml              
deployment.apps/web configured
```

然后创建hpa：

```
# kubectl  autoscale deployment web --max=3 --min=1 --cpu-percent=50         
horizontalpodautoscaler.autoscaling/web autoscaled

# 等待一会，可以看到相关的hpa信息（K8s上metrics服务收集所有pod资源的时间间隔大概在60s的时间）
# kubectl get hpa -w
NAME   REFERENCE        TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
web    Deployment/web   <unknown>/50%   1         3         1          39s
web    Deployment/web   0%/50%          1         3         1          76s
```

我们来模拟业务流量增长，看看hpa自动伸缩的效果：

```
# 我们启动一个临时pod，来模拟大量请求
# kubectl run -it --rm busybox --image=busybox -- sh
/ # while :;do wget -q -O- http://web;done

# 等待2 ~ 3分钟，注意k8s为了避免频繁增删pod，对副本的增加速度有限制
# kubectl get hpa web -w
NAME   REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
web    Deployment/web   0%/50%    1         3         1          11m
web    Deployment/web   102%/50%   1         3         1          14m
web    Deployment/web   102%/50%   1         3         3          14m

# 看下hpa的描述信息下面的事件记录
# kubectl describe hpa web
Events:
  Type     Reason                        Age                From                       Message
  ----     ------                        ----               ----                       -------
...
  Normal   SuccessfulRescale             62s                horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization (percentage of request) above target
```

好了，HPA的自动扩容已经见过了，现在停掉压测，观察下HPA的自动收缩功能：

```
# 可以看到，在业务流量高峰下去后，HPA并不急着马上收缩pod数量，而是等待5分钟后，再进行收敛，这是稳妥的作法，是k8s为了避免频繁增删pod的一种手段
# kubectl get hpa web -w
NAME   REFERENCE        TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
web    Deployment/web   102%/50%   1         3         3          16m
web    Deployment/web   0%/50%     1         3         3          16m
web    Deployment/web   0%/50%     1         3         3          20m
web    Deployment/web   0%/50%     1         3         1          21m
```