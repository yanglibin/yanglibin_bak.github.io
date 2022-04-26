---
title: gitlab-runner的安装与注册
date: 2021-10-11 14:03:18
tags:
- gitlab-runner
- devops
---

本文将介绍 `gitlab runner` 的安装，详见[官网](https://docs.gitlab.com/runner/install/)

[Runner 相关的介绍](https://docs.gitlab.com/runner/)



<!-- more -->



# 1. 安装


```bash
# For RHEL/CentOS/Fedora
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash

sudo yum install gitlab-runner


# For Debian/Ubuntu/Mint
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash

sudo apt-get install gitlab-runner


# macos
brew install gitlab-runner

sudo brew services start gitlab-runner
sudo gitlab-runner register

```



# 2. 注册

runner 的 `executor ` 有3种:

- ssh 远程连接方式
- docker
- k8s



## 2.1 手动交互注册


```bash
# mac
~$ sudo gitlab-runner register

Runtime platform                                    arch=amd64 os=darwin pid=5860 revision=c1edb478 version=14.0.1
Running in system-mode.                            
                                                   
Enter the GitLab instance URL (for example, https://gitlab.com/):
https://git.zznode.com
Enter the registration token:
Wh_irngrmTLEqn-CYmQL
Enter a description for the runner:
[admindeMacBook-Pro.local]: macos-runner
Enter tags for the runner (comma-separated):
macos
Registering runner... succeeded                     runner=Wh_irngr
Enter an executor: custom, ssh, kubernetes, shell, virtualbox, docker+machine, docker-ssh+machine, docker, docker-ssh, parallels:
shell
Runner registered successfully. Feel free to start it, but if its running already the config should be automatically reloaded! 


# linux
gitlab-runner register


Runtime platform                                    arch=amd64 os=linux pid=27613 revision=b37d3da9 version=14.3.0
Running in system-mode.                            
                                                   
Enter the GitLab instance URL (for example, https://gitlab.com/):
https://git.zznode.com
Enter the registration token:
Wh_irngrmTLEqn-CYmQL
Enter a description for the runner:
[localhost.localdomain]: centos7-runner
Enter tags for the runner (comma-separated):
linux,centos
Registering runner... succeeded                     runner=Wh_irngr
Enter an executor: parallels, shell, ssh, docker+machine, docker-ssh+machine, docker, docker-ssh, kubernetes, custom, virtualbox:
shell
Runner registered successfully. Feel free to start it, but if its running already the config should be automatically reloaded! 

```



## 2.2 免交互注册



```bash
gitlab-runner register \
     --non-interactive \
     --executor "docker" \
     --docker-image "myharbor.zznode.com/public/maven" \
     --url "https://git.zznode.com" \
     --registration-token "Wh_irngrmTLEqn-CYmQL" \
     --description "maven runner" \
     --tag-list "shared-runner" \
     --run-untagged \
     --locked="false" \
     --docker-privileged="false" \
     --docker-extra-hosts "git.zznode.com:10.2.6.103"  \
     --docker-volumes /var/run/docker.sock:/var/run/docker.sock


各行参数说明：
--non-interactive 免交互
--execturor runner 类型，这里只能是 docker
--docker-image 默认 docker 镜像. 当这里与.gitlab-ci.yaml 中同时指定了默认镜像时, .gitlab-ci.yaml 优先。
--url GitLab 的地址
--registration 上面找到的 GitLab token
--description runner 名称 (注册后可以在 GitLab ui 修改)
--tag-list 定义标签列表，后面 GitLab Job 会用到 (注册后可以在 GitLab ui 修改)
--run-untagged 是否允许未指定 tag 的 job 用这个 runner，默认 true
--locked 是否锁定，也就是一次只能一个 job 运行
--docker-extra-hosts  docker 中的 hosts 解析
--docker-volumes  挂载卷


# 显示 runner 列表:
gitlab-runner list

# 取消 注册:
gitlab-runner unregister --name="maven runner"

gitlab-runner verify --delete


# 如果runner采用k8s 方式,在运行时会根据你yaml中指定的image 生成临时的`runner-xxx` 来运行对应的job stage
```



