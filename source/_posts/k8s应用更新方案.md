---
title: k8s中部署gitlab
date: 2021-11-18 11:23:23
tags:
- k8s
---

本文将介绍在k8s中的灰度升级方案。
有下面3种方法

- kubectl rollout
- ingress(nginx) Header
- label 


**推荐使用`请求头 Header` 来区分流量 **


# 1. kubectl rollout

> 流量随机分发到同一个Deployment下不同版本的应用，随机让部分用户体验新版本功能，流量分发不可控.

1. 首先我们先启动一个deployment (myapp)

```yaml
# myapp1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:

## 默认
#  strategy:
#    rollingUpdate:
#      maxSurge: 25%
#      maxUnavailable: 25%
#    type: RollingUpdate

 strategy:
    rollingUpdate:
      maxSurge: 3
      maxUnavailable: 2
    type: RollingUpdate

  replicas: 5
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80

--- 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  labels:
    name: myapp
spec:
  rules:
  - host: myapp.ylb.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: myapp
            port: 
              number: 80
```

2. 查看

```bash
kubectl get pod | grep myapp
kubectl get deployment
kubectl get svc
kubectl get ingress

# 请求访问
for((i=0;i<1000;i++));do curl myapp.ylb.com; sleep 1; done
```

3. 暂停更新

```bash
kubectl set image deployment/myapp  myapp=ikubernetes/myapp:v2 --record  && kubectl rollout pause deployment/myapp --record
# 它会更新25%, 更新是随机的.


kubectl get rs -o wide | grep myapp
```

4. 执行全量升级

```bash
# 恢复到更新状态
kubectl rollout resume deployment/myapp

```

5. pause 状态回退

```bash
# 选设置到上个版本
kubectl set image deployment/myapp  myapp=ikubernetes/myapp:v2 --record

# 恢复状态
kubectl rollout resume deployment/myapp
```


6. 历史回滚

```bash
# 查看历史
kubectl rollout history deployment/myapp

deployment.apps/myapp 
REVISION  CHANGE-CAUSE
3         <none>
4         <none>

# 查看历史内容
kubectl rollout undo deployment/myapp --to-revision=3


# 版本回退

# 回退到上个版本
kubectl rollout undo deployment/myapp

# 回退到指定版本
kubectl rollout undo deployment/myapp --to-revision=3

```

# 2. ingress 控制

> 流量可根据权重或请求头区分，流量分发可控，适用于开放新功能到指定范围内的用户。


```yaml
## myapp2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp2
spec:
  selector:
    matchLabels:
      app: myapp2
  template:
    metadata:
      labels:
        app: myapp2
    spec:
      containers:
      - name: myapp2
        image: ikubernetes/myapp:v2
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: myapp2
spec:
  selector:
    app: myapp2
  ports:
  - port: 80
    targetPort: 80

--- 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp2
  labels:
    name: myapp2
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
## 基于权重
#    nginx.ingress.kubernetes.io/canary-weight: "80"
## 基于请求
    nginx.ingress.kubernetes.io/canary-by-header: "myapp-v2"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"

spec:
  rules:
  - host: myapp.ylb.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: myapp2
            port: 
              number: 80

```

1. 启动v1版本 `myapp1.yaml`, v2版本 `myapp2.yaml`.

2. 访问查看
```bash
# 查看正常版本v1
curl  myapp.ylb.com

# 查看升级版本v2. (通过添加头"myapp-v2: true")
curl  -H "myapp-v2: true" myapp.ylb.com
```



# 3. 通过 label 标签控制

1. 启动v1,v2 两个deployment, 并打好标签（app,func,version)
2. service 通过选择标签的公共子集(app,func) 来覆盖两组副本， 以便流量可以转发到两个应用.
3. 通过`replicas` 的数量比来控制流量。 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myappV1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      func: test
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        func: test
        version: v1
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myappV2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      func: test
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        func: test
        version: v2
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v2
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80

--- 

apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    func: test
  ports:
  - port: 80
    targetPort: 80

```

