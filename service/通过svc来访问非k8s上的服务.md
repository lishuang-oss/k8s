**通过svc来访问非k8s上的服务**

```
创建service后，会自动创建对应的endpoint，在这里的关键在于endpoint，这里面的关键在于selector对应到了例如上面deployment上的标签，那么如果没有定义这个selector，那么系统是不会自动创建endpoint的。
那么我们可不可以自己创建endopint呢？答案是可以的，我们可以通过创建不带selector的service，然后创建同样名称的endpoint，来关联k8s集群之外的服务。
```

```
我在别的节点上用yum安装了一个nginx下面一起来看看吧,节点ip是10.0.0.51

apiVersion: v1
kind: Service
metadata:
  name: mysvc  
  namespace: default
spec:
  type: ClusterIP
  ports:
  - port: 90
    protocol: TCP

---

apiVersion: v1
kind: Endpoints
metadata:
  name: mysvc  #名称要与service的一致，才能够建立关系
  namespace: default
subsets:
- addresses:
  - ip: 10.0.0.51   #添加到endpoint的地址
    nodeName: 10.0.0.51  #名称可以自行定义
  ports:
  - port: 80   #对应10.0.0.51:80
    protocol: TCP
    
[root@master svc]# kubectl get svc
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        64d
mysql           ClusterIP   10.104.93.193   <none>        3306/TCP       48d
mysvc           ClusterIP   10.98.162.69    <none>        90/TCP         10m
[root@master svc]# kubectl get endpoints
NAME            ENDPOINTS                             AGE
kubernetes      10.0.0.110:6443                       64d
mysql           172.165.11.13:3306                    48d
mysvc           10.0.0.51:80                          7m51s
nginx-service   172.165.11.15:80,172.171.205.129:80   141m
```

