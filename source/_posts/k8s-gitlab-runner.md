---
title: k8s中安装使用gitlab-runner
date: 2021-10-12 15:23:44
tags:
- k8s
- gitlab-runner
- devops
---


# 0. 先决条件：

- k8s  : v1.19.4
  - namespace: `kube-ops`
- helm:  v3.2.1
- ceph
  - 创建 bucket : `runner`

<!-- more -->


# 1. 获取 helm 安装包 


```bash
# 1. helm 中搜索gitlab-runner
helm search hub gitlab-runner

URL                                               	CHART VERSION	APP VERSION	DESCRIPTION                
https://hub.helm.sh/charts/pnnl-miscscripts/git...	0.1.5        	0.1.3-1    	A Helm chart for Kubernetes
https://hub.helm.sh/charts/gitlab/gitlab-runner   	0.33.0-rc1   	14.3.0-rc1 	GitLab Runner              
https://hub.helm.sh/charts/choerodon/gitlab-runner	0.2.4        	0.2.4      	gitlab-runner for Choerodon
https://hub.helm.sh/charts/psu-swe/gitlab-runner  	0.7.0        	13.7.0     	GitLab Runner              
https://hub.helm.sh/charts/wenerme/gitlab-runner  	0.32.0       	14.2.0     	GitLab Runner              
https://hub.helm.sh/charts/camptocamp/gitlab-ru...	0.12.6       	12.6.0     	GitLab Runner              


# 2. 将地址 https://hub.helm.sh/charts/gitlab/gitlab-runner  在浏览器中打开,找到 repo 地址: http://charts.gitlab.io/

# 3. 添加 repo
helm repo add gitlab http://charts.gitlab.io/

# 4. 拉取gitlab-runner helm 包
helm pull gitlab/gitlab-runner
tar -xvf gitlab-runner-0.32.0.tgz
```





# 2. 安装配置 gitlab-runner

## 2.1 获取 runner 与 gitlab 的连接信息 

查看gitlab 上的runner 连接信息: https://git.ylb.com/admin/runners , 下面的 配置文件:`gitlab-runner.yaml` 要用。

- url   ：https://git.ylb.com
- registration token  : `Wh_irngrmTLEqn-CYmQL`



## 2.2 准备runner的启动配置

gitlab-runner 的helm 配置文件: `gitlab-runner.yaml`

> 主要修改配置:
- gitlabUrl
- runnerRegistrationToken
- cache

```yaml
imagePullPolicy: IfNotPresent
gitlabUrl: https://git.ylb.com

runnerRegistrationToken: "Wh_irngrmTLEqn-CYmQL"

terminationGracePeriodSeconds: 3600
concurrent: 10
checkInterval: 30
rbac:
  create: true
  clusterWideAccess: true

metrics:
  enabled: true
runners:
# 这里面配置一些 runner 运行 相关的配置
# 配置示例: https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-runnerskubernetes-section
  config: |
    [[runners]]
      [runners.kubernetes]
        image = "ubuntu:16.04"

        [runners.kubernetes.node_selector]
          kubernetes.io/os = linux


# s3 缓存配置
  cache:
    cacheType: s3
    cachePath: "gitlab_runner"
    cacheShared: true

    s3ServerAddress: "10.2.6.63:7480"
    s3BucketName: "runner"
    s3BucketLocation:
    s3CacheInsecure: false
    secretName: s3access

  tags: "k8s"

# 配置gitlab-runner 运行的节点
# nodeSelector:
#   os: "linux"

securityContext:
  fsGroup: 65533
  runAsUser: 100

```



## 2.3 配置 yaml 中用到的`secret`

```yaml
kubectl create secret generic s3access \
    --from-literal=access_key=7X1X28GZNB9XGIFN2CLS  \
    --from-literal=secret_key=ywOkr9BYSHRwzGEgB5mCJpb8QpTn4w19QNseKSAj  \
    -n kube-ops
```



## 2.4 针对一些问题对 template 的修改

### 1. s3CacheInsecure 配置无效问题



`s3CacheInsecure:  false` 表示对`ceph` 的请求为http (**如果是true就是https**)，但实际证明，当前版本的chart中该配置是无效的，等到运行时还是会以https协议访问，解决此问题的方法是修改`templates/__cache.tpl` 文件.



```bash
# 第18-21行: 删除原先的if判断和对应的end这两行，直接给CACHE_S3_INSECURE赋值
 18 {{-       if .Values.runners.cache.s3CacheInsecure }}
 19 - name: CACHE_S3_INSECURE
 20   value: "true"
 21 {{-       end }}

# 修改为如下:
 18 - name: CACHE_S3_INSECURE
 19   value: {{ default "" .Values.runners.cache.s3CacheInsecure | quote }}

```

### 2. docker 执行的问题

当前版本的chart中 是没有将宿主机的docker的sock映射给runner executor，当job中执行docker命令就会报错: `ERROR: Cannot connect to the Docker daemon at tcp://localhost:2375. Is the docker daemon running? ` , 我们需要将宿主机的docker socket 挂载上去，docker 相关的操作由宿主机来执行。

打开`templates/configmap.yaml` 做如下修改。

```bash
# 在第57行加如下内容：
    cat >>/home/gitlab-runner/.gitlab-runner/config.toml <<EOF
            [[runners.kubernetes.volumes.host_path]]
              name = "docker"
              mount_path = "/var/run/docker.sock"
              read_only = true
              host_path = "/var/run/docker.sock"
    EOF 
    
  
```



### 3. gitlab-runner 缓存功能无法使用的问题

当我测试缓存功能时，发现缓存一直没有生效。 查看日志发现没有upload cache.zip 到 s3 存储，而是报如下错误： `No URL provided, cache will be not uploaded to shared cache server. Cache will be stored only locally. `

通过搜索，找到个帖子：[Gitlab Cache Server Configuration Bug](https://blog.csdn.net/xichenguan/article/details/101436883)  定位到了问题，是由于gitlab-runner 启动后没有获取到 `secret: s3access` 的认证信息导致的。 



下面是问题排查过程与解决方法。 （解决方法直接看 4.2) 

```bash
# 1. 通过下面命令生成 gitlab-runner 启动的配置文件
helm install gitlab-runner  gitlab-runner -f gitlab-runner.yaml -n kube-ops --dry-run --debug > gitlab-tmp.yaml

# 2. 在第135，136 行发现， CACHE_S3_ACCESS_KEY 读取的文件为: /secrets/accesskey, CACHE_S3_SECRET_KEY 读取的文件为: /secrets/secretkey

134     if [[ -f /secrets/accesskey && -f /secrets/secretkey ]]; then
135       export CACHE_S3_ACCESS_KEY=$(cat /secrets/accesskey)
136       export CACHE_S3_SECRET_KEY=$(cat /secrets/secretkey)
137     fi

# 3. 下面在启动的gitlab-runner 中查看里面存在的文件为: access_key,secret_key.  没有上面所说的 accesskey,secretkey  
kubectl exec -it -n ylb-tmp gitlab-runner-gitlab-runner-67749879bd-hzvzt -- ls /secrets

access_key                 runner-token
runner-registration-token  secret_key

# 4. 我们已经定位到了问题，是由于文件名不一致导致。下面我们修改对应的配置。
# 4.1 查看 所在的配置文件
find templates -type f | xargs grep "cat /secrets/accesskey"
templates/configmap.yaml:      export CACHE_S3_ACCESS_KEY=$(cat /secrets/accesskey)

find templates -type f | xargs grep "cat /secrets/secretkey"
templates/configmap.yaml:      export CACHE_S3_ACCESS_KEY=$(cat /secrets/secretkey)

# 4.2 解决： 修改 templates/configmap.yaml 中对应的内容
cat /secrets/accesskey -->  cat /secrets/access_key 
cat /secrets/secretkey -->  cat /secrets/secret_key 
```





下面附一个测试 yaml : `.gitlab-ci.yml`

```yaml
image: busybox:latest

stages:
- build
- test


cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
  - hello.txt

before_script:
  - echo "Before script section"

after_script:
  - echo "After script section"

my_build1:
  stage: build
  tags:
    - k8s
  script:
   
    - echo "将内容写入缓存"
    - echo "build stage ..." > hello.txt

my_test1:
  stage: test
  tags:
    - k8s
    
  script:
    - echo "从缓存读取内容"
    - cat hello.txt

```



## 2.5 install  gitlab-runner
`helm install gitlab-runner gitlab-runner -f gitlab-runner.yaml -n kube-ops`



# 3. 常见问题

## 问题1

```
Uploading artifacts...
./target/*.jar: found 1 matching files and directories 
WARNING: Uploading artifacts as "archive" to coordinator... failed  id=77 responseStatus=308 Permanent Redirect status=308 token=9SFqw4T8
WARNING: Retrying...                                context=artifacts-uploader error=invalid argument
FATAL: invalid argument                            
```



`gitlab-runner register` 注册时，url 没有配置为htts导致的.



## 问题2

```
ERROR: Cannot connect to the Docker daemon at tcp://localhost:2375. Is the docker daemon running? 
```

没有 docker socket 导致。 解决方法见上面的： `docker 执行问题`



## 问题3

```
Creating cache test00001...
target/: found 2 matching files and directories    
No URL provided, cache will be not uploaded to shared cache server. Cache will be stored only locally. 
Created cache
```

是由于gitlab-runner 启动后没有获取到 `secret: s3access` 的认证信息导致的。  详见上面的：`gitlab-runner 缓存功能无法使用的问题`



# 4. 参考

- [GitLab Runner部署(kubernetes环境)](https://blog.csdn.net/boling_cavalry/article/details/106991576)
- [Gitlab Cache Server Configuration Bug](https://blog.csdn.net/xichenguan/article/details/101436883)
- [gitlab-ci配置详解](https://segmentfault.com/a/1190000011890710)
