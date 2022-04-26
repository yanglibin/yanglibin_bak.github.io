---
title: 在kubernetes中部署OpenLDAP
date: 2021-09-16 17:21:03
tags:
- k8s
- ldap
- ldap
- phpldapadmin
- devops
---

# 1. 环境说明

在进行本文操作前, 你已经搭建好了`ceph`存储, 并且已创建了`storageClass` :  `ceph-rdb`.



- os: 	 centos7
- k8s: 	 v1.19.4
- namespace:  ldap
- 存储: ceph



**创建namespace**:  `kubectl create ns ldap`

<!-- more -->
# 2. 创建secret 
> 设置`organizatation`,`domain`,`password`,存储在`secret` 中。

`ldap-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openldap-secrets
  # namespace: openldap
type: Opaque
data:
  organizatation: "ZGV2b3BzCg==" # devops
  domain: "eWxiLmNvbQo=" # ylb.com
  password: "bXlwYXNzd29yZA==" #mypassword
```

执行: `kubectl apply -f ldap-secret.yaml -n ldap` 

# 3. 创建pvc
> 用于存储ldap的数据与配置

## 3.1 ceph-secret

`ldap-ceph-secret.yaml`

```yaml
apiVersion: v1
data:
  # 下面的key 改为你自己的
  key: QVFDZS9oUmdZSlVESVJBQW80Y3RnN2RKc0lKdHQ1SkU1QjUwc0E9PQ==
kind: Secret
metadata:
  name: ceph-user-secret
#  namespace: ldap
type: kubernetes.io/rbd

```

## 3.2 ldap-pvc

`ldap-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ldap-data
  # namespace: ldap
  labels:
    app: ldap
spec:
  storageClassName: ceph-rdb
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: "5Gi"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ldap-config
  # namespace: ldap
  labels:
    app: ldap
spec:
  storageClassName: ceph-rdb
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: "1Gi"
```

## 3.3 执行

```bash
kubectl apply -n ldap -f ldap-ceph-secret.yaml
kubectl apply -n ldap -f ldap-pvc.yaml
```


# 4. 创建deployment

## 4.1 openldap
- deployment
- service

`ldap.yaml`

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openldap-deployment
  # namespace: openldap
  labels:
    app: openldap
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openldap
  template:
    metadata:
      labels:
        app: openldap
    spec:
      containers:
        - name: openldap
          image: osixia/openldap:1.3.0
          resources:
            limits:
              memory: "512Mi"
              cpu: "500m"
          ports:
            - containerPort: 389
            - containerPort: 636
          env:
            - name: LDAP_ORGANISATION
              valueFrom:
                secretKeyRef:
                  name: openldap-secrets
                  key: organizatation
            - name: LDAP_DOMAIN
              valueFrom:
                secretKeyRef:
                  name: openldap-secrets
                  key: domain
            - name: LDAP_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: openldap-secrets
                  key: password
          volumeMounts:
            - name: ldap-data
              mountPath: "/var/lib/ldap"
              subPath: database
            - name: ldap-config
              mountPath: "/etc/ldap/slapd.d"
              subPath: config

      volumes:
        - name: ldap-data
          persistentVolumeClaim:
            claimName: ldap-data

        - name: ldap-config
          persistentVolumeClaim:
            claimName: ldap-config
---
apiVersion: v1
kind: Service
metadata:
  name: openldap-service
  # namespace: openldap
spec:
  selector:
    app: openldap
  type: NodePort
  ports:
  - name: openldap1
    protocol: TCP
    port: 389
    targetPort: 389
  - name: openldap2
    protocol: TCP
    port: 636
    targetPort: 636

```


## 4.2 phpldapadmin
- deployment
- service



`phpldapadmin.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpldapadmin-deployment
  # namespace: openldap
  labels:
    app: phpldapadmin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phpldapadmin
  template:
    metadata:
      labels:
        app: phpldapadmin
    spec:
      containers:
        - name: phpldapadmin
          image: osixia/phpldapadmin:0.9.0
          resources:
            limits:
              memory: "512Mi"
              cpu: "500m"
          ports:
            - containerPort: 443
          env:
            - name: PHPLDAPADMIN_LDAP_HOSTS
              value: openldap-service            # 为上面 ldap service 的值.
---
apiVersion: v1
kind: Service
metadata:
  name: phpldapadmin-service
  # namespace: openldap
spec:
  type: LoadBalancer
  selector:
    app: phpldapadmin
  ports:
  - protocol: TCP
    port: 9943
    targetPort: 443
```



## 4.3 执行

```bash
kubectl apply -f ldap.yaml -n ldap
kubectl apply -f phpldapadmin.yaml -n ldap
```



# 5. 访问

```bash
# 查看pod 是否全部启动
kubectl get pod -n ldap

NAME                                          READY   STATUS    RESTARTS   AGE
openldap-deployment-66f689f798-h755j          1/1     Running   0          72m
phpldapadmin-deployment-7969dc4fdd-2587d      1/1     Running   0          96m

# 查看svc 中对应的外部访问端口
kubectl get svc -n ldap

NAME                         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)               
openldap-service             NodePort       10.102.75.135    <none>        389:48105/TCP,636:19285/TCP   89m
phpldapadmin-service         LoadBalancer   10.97.99.148     <pending>     9943:36828/TCP                89m

# 浏览器访问:
# https://YOUR_K8S_MASTER_IP:36828
# user: cn=admin,dc=ylb,dc=com    # 在前面 secret 已设置
# password:  mypassword           # 在前面 secret 已设置
```

# 6. 参考文档

- [Como desplegar un LDAP en Kubernetes](https://piensoluegoinstalo.com/como-desplegar-un-ldap-en-kubernetes/)

