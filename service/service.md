service

```
apiVersion: apps/v1   #api版本 
kind: Deployment   #选择什么控制器来创建
metadata:  #定义元数据
  name: nginx-deployment #deployment资源名称
  namespace: default #指定pod运行命名空间
spec:  #关于pod的详细定义
  selector:    #选择器
    matchLabels:  #给pod打标签
      app: nginx
  replicas: 2 #副本数
  template: 
    metadata:
      labels:   #这里的标签要和selector选择器匹配
        app: nginx  
    spec: #spec对象的容器信息
      containers:
      - name: nginx  #容器名称
        image: nginx:1.8
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx #这里的标签选择对应dp中的selector
  type: NodePort
```

