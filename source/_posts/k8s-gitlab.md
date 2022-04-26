---
title: k8s中部署gitlab
date: 2021-09-23 15:13:43
tags:
- gitlab
- k8s
- devops
---



**本实验比较消耗资源, 请确保集群有足够的资源, 不然可能由于deployment 中的 resource 资源不够而启动不成功. 实在不成功建议 `注释探针` 试试.**



- 存储：  ceph
- k8s:  v1.19.4
- namespace:  kube-ops



<!-- more -->

# 创建 StorageClass 
> 供 gitlab 数据的动态存储.  要创建 default  namespace 中

- storageclass for ceph
- secret for ceph



1. ceph-sc.yaml

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ceph-rdb
provisioner: ceph.com/rbd
parameters:
  # ceph 的地址
  monitors: 10.2.6.63:6789,10.2.6.64:6789,10.2.6.65:6789 
  # ceph 中使用的pool
  pool: k8s

  # admin 连接证书创建在 kube-system 中
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  # user 的认证要在每个使用的namespace 中创建
  userId: kube
  userSecretName: ceph-user-secret

  imageFormat: "2"
  imageFeatures: "layering"
```

2. ceph-secret.yaml

```yaml
apiVersion: v1
data:
  # key 值改成你自己对应的值
  key: QVFDUDdoUmdyWDZoSlJBQUk2ZzFhSTlVNUFpTTFLMHNHbWM4RUE9PQ==
kind: Secret
metadata:
  name: ceph-secret
  namespace: kube-system

type: kubernetes.io/rbd
```

3. ceph-user-secret.yaml 

```yaml
apiVersion: v1
data:
  key: QVFDZS9oUmdZSlVESVJBQW80Y3RnN2RKc0lKdHQ1SkU1QjUwc0E9PQ==
kind: Secret
metadata:
  name: ceph-user-secret
  # namespace: kube-ops
type: kubernetes.io/rbd

```




```bash
# 创建
kubectl apply -f ceph-sc.yaml
kubectl apply -f ceph-secret.yaml
kubectl apply -f ceph-user-secret.yaml -n kube-ops

# 查看
kubectl get sc 
NAME           PROVISIONER      AGE
ceph-rdb       ceph.com/rbd     9s
```


# 创建 PVC


## Redis PVC
gitlab-redis-pvc.yaml
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: gitlab-redis-pvc
  # namespace: kube-ops           # 指定要运行在的 namespace
  annotations:
    volume.beta.kubernetes.io/storage-class: "ceph-rdb"  # 指定 storage-class 的名称,不写则使用默认的.
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi    # 定义 pvc需要多大的存储
```


## postgresql PVC
gitlab-postgresql-pvc.yaml
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: gitlab-postgresql-pvc
  # namespace: kube-ops           # 指定要运行在的 namespace
  annotations:
    volume.beta.kubernetes.io/storage-class: "ceph-rdb"  # 指定 storage-class 的名称,不写则使用默认的.
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi    # 定义 pvc需要多大的存储
```
## gitlab PVC
gitlab-pvc.yaml
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: gitlab
  # namespace: kube-ops 
  annotations:
    volume.beta.kubernetes.io/storage-class: "ceph-rdb"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5000Gi 
```
```bash
# 创建
kubectl apply -f gitlab-redis-pvc.yaml -f gitlab-postgresql-pvc.yaml  -f gitlab-pvc.yaml -n kube-ops
```

# 创建gitlab及相关组件

- redis
- postgresql
- gitlab




## gitlab - redis 
gitlab-redis.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab-redis
  # namespace: kube-ops
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      name: redis
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        resources:
          limits:
            memory: "10240Mi"
            cpu: "2"
        image: sameersbn/redis
        imagePullPolicy: IfNotPresent
        ports:
        - name: redis
          containerPort: 6379
        volumeMounts:
        - name: gitlab-redis
          mountPath: /var/lib/redis
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 30
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          timeoutSeconds: 1
      volumes:
      - name: gitlab-redis
        persistentVolumeClaim:
          claimName: "gitlab-redis-pvc"

---
apiVersion: v1
kind: Service
metadata:
  name: gitlab-redis
  # namespace: kube-ops
  labels:
    app: redis
spec:
#  type: NodePort
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
#      nodePort: 30051
      protocol: TCP
  selector:
    app: redis
```


## gitlab - postgresql


gitlab-postgresql.yaml 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab-postgresql
  # namespace: kube-ops
  labels:
    app: postgresql
spec:
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      name: postgresql
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: sameersbn/postgresql:10
        imagePullPolicy: IfNotPresent
        env:
        - name: DB_USER
          value: gitlab
        - name: DB_PASS
          value: passw0rd
        - name: DB_NAME
          value: gitlab_production
        - name: DB_EXTENSION
          value: pg_trgm
        ports:
        - name: postgres
          containerPort: 5432
        resources:
          limits:
            memory: "4Gi"
            cpu: "2"
        volumeMounts:
        - name: gitlab-postgresql-data
          mountPath: /var/lib/postgresql
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -h
            - localhost
            - -U
            - postgres
          initialDelaySeconds: 30
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -h
            - localhost
            - -U
            - postgres
          initialDelaySeconds: 5
          timeoutSeconds: 1
      volumes:
      - name: gitlab-postgresql-data
        persistentVolumeClaim:
          claimName: "gitlab-postgresql-pvc"
          
          
---
apiVersion: v1
kind: Service
metadata:
  name: gitlab-postgresql
  namespace: kube-ops
  labels:
    app: postgresql
spec:
#  type: NodePort
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
#      nodePort: 30052
  selector:
    app: postgresql
```


## gitlab
gitlab.yaml


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab
  namespace: kube-ops
  labels:
    app: gitlab
spec:
  selector:
    matchLabels:
      app: gitlab
  template:
    metadata:
      name: gitlab
      labels:
        app: gitlab
    spec:
      containers:
      - name: gitlab
        image: sameersbn/gitlab:12.2.5
        imagePullPolicy: IfNotPresent
        env:
        - name: TZ
          value: Asia/Shanghai
        - name: GITLAB_TIMEZONE
          value: Beijing
        - name: GITLAB_SECRETS_DB_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_SECRETS_SECRET_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_SECRETS_OTP_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_ROOT_PASSWORD
          value: admin321
        - name: GITLAB_ROOT_EMAIL
          value: ylb@ylb.com
        - name: GITLAB_HOST
          value: git.ylb.com
        - name: GITLAB_PORT
          value: "80"
        - name: GITLAB_SSH_PORT
          value: "22"
        - name: GITLAB_NOTIFY_ON_BROKEN_BUILDS
          value: "true"
        - name: GITLAB_NOTIFY_PUSHER
          value: "false"
        - name: GITLAB_BACKUP_SCHEDULE
          value: daily
        - name: GITLAB_BACKUP_TIME
          value: 01:00
        - name: DB_TYPE
          value: postgres
        - name: DB_HOST
          value: gitlab-postgresql
        - name: DB_PORT
          value: "5432"
        - name: DB_USER
          value: gitlab
        - name: DB_PASS
          value: passw0rd
        - name: DB_NAME
          value: gitlab_production
        - name: REDIS_HOST
          value: gitlab-redis
        - name: REDIS_PORT
          value: "6379"
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        - name: ssh
          containerPort: 22
        volumeMounts:
        - mountPath: /home/git/data
          name: gitlab-data
#        livenessProbe:
#          httpGet:
#            path: /users/sign_in
#            port: 80
#          initialDelaySeconds: 180
#          timeoutSeconds: 5
#        readinessProbe:
#          httpGet:
#            path: /users/sign_in
#            port: 80
#          initialDelaySeconds: 5
#          timeoutSeconds: 1
        resources:
          limits:
            memory: "4Gi"
            cpu: "4"
      volumes:
      - name: gitlab-data
        persistentVolumeClaim:
              claimName: "gitlab"
              
---
apiVersion: v1
kind: Service
metadata:
  name: gitlab
  namespace: kube-ops
  labels:
    app: gitlab
spec:
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: https
      port: 443
      targetPort: 443
    - name: ssh
      port: 22
      targetPort: 22
  selector:
    app: gitlab
    
---
kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: gitlab
  #namespace: kube-ops
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/client-max-body-size: "0"
    ingress.kubernetes.io/proxy-body-size: "0"
    ingress.kubernetes.io/proxy_connect_timeout: 5s
    ingress.kubernetes.io/proxy_timeout: 300s
    ingress.kubernetes.io/ssl-redirect: "true"
    kubernetes.io/ingress.class: nginx
    meta.helm.sh/release-name: gitlab
#    meta.helm.sh/release-namespace: kube-ops
    nginx.ingress.kubernetes.io/client-max-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy_connect_timeout: 5s
    nginx.ingress.kubernetes.io/proxy_timeout: 300s
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

spec:
  rules:
  - host: git.ylb.com
    http:
      paths:
      - backend:
          serviceName: gitlab
          servicePort: 80
        path: /
  tls:
  - hosts:
    - git.ylb.com
    secretName: ylb.com  # ylb.com (将https证书制成secre, 本文已略)

```



```bash
kubectl apply -f gitlab-redis.yaml -f gitlab-postgresql.yaml -n kube-ops
kubectl apply -f gitlab.yaml -n kube-ops
```




# 访问
URL:   git.ylb.com
user/password:     root/admin321

# 常见问题


## 1. pvc 无法删除


> 始终处于“Terminating”状态，而且delete不掉。



解决方法:
```bash
kubectl patch  pvc -n kube-ops gitlab-postgresql-pvc -p '{"metadata":{"finalizers":null}}'
```

