---
title: saltstack安装与使用
date: 2021-09-22 11:03:23
tags:
- saltstack
- devops
---



# 0. 环境准备

- os: centos7
- 机器说明:

|          | IP             | hostname  |
| -------- | -------------- | --------- |
| master01 | 192.168.90.10  | master-01 |
| slave-01 | 192.168.90.120 | slave-01  |

<!-- more -->



1. 修改各机器的hostname

   ```bash
   # 在master 上操作
   hostnamectl set-hostname  master-01
   
   # 在 slave-01 上操作
   hostnamectl set-hostname slave-01
   ```

   

2. 配置 /etc/hosts

   ```
   192.168.90.10   master-01
   192.168.90.120  slave-01
   ```

   

# 1. 安装



## 方式1: pip

### a. master 节点:

```bash
# 安装python3环境
yum install python3
# 安装salt
pip3 install salt

salt --version

nohup salt-master &> /dev/null & 			 # 启动服务

ss -tanl 	# 查看4505,4506 端口是否启动.
```

### b. slave 节点:

```bash
# 安装python3环境
yum install python3
# 安装salt
pip3 install salt
# 配置文件
mkdir -p /etc/salt/minion.d/

cat > /etc/salt/minion.d/master.conf << EOF
master: master-01
EOF
# 启动 salt-minion
nohup salt-minion &> /dev/null & 
```

### c. 查验: (在master 上操作)

```bash
salt-key 
# 结果如下:
    Accepted Keys:
    Denied Keys:
    Unaccepted Keys:
    slave-01

# 授权key
salt-key -a 'slave-01'

# 重新查看:
salt-key 
# 结果如下:
    Accepted Keys:
    slave-01
    Denied Keys:
    Unaccepted Keys:
    Rejected Keys:

# 测试
salt '*' test.ping
```



## 方式2: 官方安装脚本

- [文档](https://repo.saltproject.io/)

### a. master 节点

```bash
curl -fsSL https://bootstrap.saltproject.io -o install_salt.sh
sudo sh install_salt.sh -P -M -x python3

salt --version

# 启动服务
systemctl start salt-master

ss -tanl 	# 查看4505,4506 端口是否启动.
```



### b. slave节点

```bash
curl -fsSL https://bootstrap.saltproject.io -o install_salt.sh
sudo sh install_salt.sh -P -x python3

# 配置文件
cat > /etc/salt/minion.d/master.conf << EOF
master: master-01
EOF
# 启动 salt-minion
systemctl start salt-minion
```



### c. 查验: ( 在master上操作 )

```bash
salt-key 
# 结果如下:
    Accepted Keys:
    Denied Keys:
    Unaccepted Keys:
    slave-01

# 授权key
salt-key -a 'slave-01'

# 重新查看:
salt-key 
# 结果如下:
    Accepted Keys:
    slave-01
    Denied Keys:
    Unaccepted Keys:
    Rejected Keys:

# 测试
salt '*' test.ping
```



## 方式3: yum   ( 本文采用 )

- [文档](https://docs.saltproject.io/en/3003/topics/installation/rhel.html)

### a. yum 仓库配置

```bash
vim /etc/yum.repos.d/salt.repo

[saltstack-repo]
name=Salt repo for Red Hat Enterprise Linux $releasever
baseurl=https://repo.saltproject.io/py3/redhat/$releasever/$basearch/latest
enabled=1
gpgcheck=1
gpgkey=https://repo.saltproject.io/py3/redhat/$releasever/$basearch/latest/SALTSTACK-GPG-KEY.pub
       https://repo.saltproject.io/py3/redhat/$releasever/$basearch/latest/base/RPM-GPG-KEY-CentOS-7
     
```

### b. master 节点

```bash
yum install -y salt-master

# 启动服务
systemctl start salt-master

ss -tanl 	# 查看4505,4506 端口是否启动.
```

### c. slave 节点

```bash
yum install -y salt-minion

# 配置文件
cat > /etc/salt/minion.d/master.conf << EOF
master: master-01
EOF
# 启动 salt-minion
systemctl start salt-minion
```



### d. 查验: ( 在master上操作 )

```bash
salt-key 
# 结果如下:
    Accepted Keys:
    Denied Keys:
    Unaccepted Keys:
    slave-01

# 授权key
salt-key -a 'slave-01'

# 重新查看:
salt-key 
# 结果如下:
    Accepted Keys:
    slave-01
    Denied Keys:
    Unaccepted Keys:
    Rejected Keys:

# 测试
salt '*' test.ping
```





## 配置文件介绍

**安装好salt之后开始配置，salt-master默认监听两个端口：**

 默认情况下，Salt主服务器侦听所有接口（0.0.0.0）上的端口4505和4506 



| 4505 | publish_port 提供远程命令发送功能           |
| ---- | ------------------------------------------- |
| 4506 | ret_port 提供认证，文件服务，结果收集等功能 |

**通信协议**: ZeroMQ协议通信 



**确保客户端可以通信服务器的此2个端口，保证防火墙允许端口通过。**

```bash
# 防火墙配置
-A INPUT -p tcp -m multiport --dports 4505,4506 -m state --state NEW -j ACCEPT
```



```bash
# salt-master的配置文件	
/etc/salt/master
/etc/salt/master.d/*.conf 

# salt-minion的配置文件	
/etc/salt/minion
/etc/salt/minion.d/*.conf  # master.conf

# salt-minion id name 配置
/etc/salt/minion_id
```



# 2. CLI 操作

模块

- [salt.modules.group](https://docs.saltproject.io/en/latest/ref/modules/all/salt.modules.group.html)
- [salt.modules.pkg](https://docs.saltproject.io/en/latest/ref/modules/all/salt.modules.pkg.html)
- [salt.modules.service](https://docs.saltproject.io/en/latest/ref/modules/all/salt.modules.service.html)
- [salt.modules.sysctl](https://docs.saltproject.io/en/latest/ref/modules/all/salt.modules.sysctl.html)
- [salt.modules.user](https://docs.saltproject.io/en/latest/ref/modules/all/salt.modules.user.html)



### 连通性测试

```bash
salt '*' test.ping
# * 指所有slave 节点, 支持使用统配符, 如:slave-*
# 也可以使用 -E 来指定正则表达式
```

### [远程执行命令](https://docs.saltproject.io/en/latest/topics/execution/remote_execution.html)

```bash
salt 'slave-01'  cmd.run "ls /tmp"

# 执行多个指令
salt '*' cmd.run,test.ping,test.echo   'cat /proc/cpuinfo',,foo
```



### [user / group](https://docs.saltproject.io/en/latest/ref/modules/all/index.html#all-salt-modules)

```bash
# 创建group
salt '*' group.add mytest gid=1100

# 创建user
salt '*' user.add  mytest uid=1100 gid=1100 

# 删除user
salt '*' user.delete mytest

```



### pkg / service

```bash
# 下载软件包 httpd
salt '*' pkg.download httpd

# 安装软件 httpd
salt '*' pkg.install httpd


# 启动 httpd
salt '*' service.start httpd

# 开机启动
salt '*' service.enable httpd

# 停止服务
salt '*' service.stop httpd

# 卸载 httpd
salt '*' pkg.remove httpd

```



### sysctl

```bash
# get net.ipv4.ip_forwart
salt '*' sysctl.get net.ipv4.ip_forward
# 设置 net.ipv4.ip_forwart=1
salt '*' sysctl.persist net.ipv4.ip_forward 1
# 显示所有
salt '*' sysctl.show
```



### [file](https://docs.saltproject.io/en/latest/ref/modules/all/salt.modules.file.html#module-salt.modules.file)

```bash
# 向文件中添加一行
salt '*' file.append /etc/motd 'hello salt 


# 文件copy:  将 node* 节点上的 /tmp/a.txt copy 到 /tmp/a/1.txt 上
salt 'node*' file.copy /tmp/a.txt /tmp/a/1.txt


# 将 master 中 http/httpd.conf  分发到 minion 的 /tmp/httpd.conf

salt '*' file.manage_file /tmp/httpd.conf '' '{}' salt://http/httpd.conf 'salt://http/httpd.conf '{hash_type: 'md5', 'hsum': <md5sum>}' root root '755' '' base ''


```



### 文件推送

```bash
# salt-cp  : 将 master 上的 /etc/hosts 文件推送到 minion 节点上的 /tmp/a.hosts
salt-cp 'slave*'  /etc/hosts /tmp/a.hosts

```

### 文件拉取

> 默认是禁止从minion拉取文件到master，需要修改 master 上的配置(/etc/salt/master.d/roots.conf): file_recv: True 才能拉取文件

修改master 配置

```bash
cat >>  /etc/salt/master.d/roots.conf << EOF
# 开启拉取文件配置
file_recv: True 
EOF

# 重启服务
systemctl restart salt-master
```



```bash
salt '*' cp.push /etc/issue
# 拉取的文件默认保存在: /var/cache/salt/master/minions/
```



### 其他

自己在 https://docs.saltproject.io/en/latest/ref/modules/all/index.html#all-salt-modules 中查看



# 3. 模块的使用

> [文档地址](https://docs.saltproject.io/en/latest/ref/states/all/index.html)

- file
- pkg
- service
- cmd
- 



### master 新加配置

```bash
cat > /etc/salt/master.d/roots.conf  << EOF
# 指定base目录
file_roots:
  base:
    - /srv/salt/_modules

# 指定多个sls的入口文件
state_top: top.sls
# 分组
nodegroups:
  group1: slave-01
  group2: master-01
  group3: slave-01, master-01
EOF

mkdir -p  /srv/salt/_modules  # base 目录

# 重启 master
systemctl restart salt-master

# top.sls
cat > /srv/salt/_modules << EOF
base:         # 指定环境为base,所以这里的top.sls 在 /srv/salt/_modules 下
  "*":        # 指定主机
# 指定调用的sls 文件
#    - PATH.FILE   # .sls 后缀不要写
EOF

```



### 目录结构

```bash
tree /srv/salt/_modules

/srv/salt/_modules
├── sys
│   ├── files
│   │   └── hosts
│   ├── file.sls
│   ├── pkg.sls
│   └── service.sls
└── top.sls

2 directories, 5 files
```





### file 模块

> 使用file 模块管理文件与目录
>
> [相关官方文档](https://docs.saltproject.io/en/latest/ref/states/all/salt.states.file.html#module-salt.states.file)



#### top.sls 文件

```bash
cat > /srv/salt/_modules/top.sls << EOF
base: 
  "*": 
    - sys.file 
EOF
```



####  file.sls 文件

```yaml
cat  > /srv/salt/_modules/sys/file.sls  << EOF
分发hosts 文件:
  file.managed:
    - name: /tmp/hosts
    - source: salt://sys/files/hosts
    - mode:
    - user: root
    - group: root

minion节点上向文本中插入内容:
  file.append:
    - name: /tmp/hosts
    - makedirs: True
    - text: 
      - 这是插入的第一行
      - 这是插入的第二行

minion节点上创建目录:
  file.directory:
    - name: /tmp/minion/test/a
    - user: test
    - group: test
    - dir_mode: 755
    - makedirs: True

minion节点上的文件copy操作:
  file.copy:
    - name: /tmp/host.copy
    - source: /etc/hosts
    - force: True
    - makedirs: True
    - user: test
    - group: root
    - mode: 644

minion节点创建软连接:
  file.symlink:
    - name: /tmp/myetc
    - target: /etc
    - makedirs: True

minion节点 rename 操作:
  file.rename:
    - name: /tmp/etc_rename
    - source: /tmp/myetc
    - makedirs: True
    
EOF
```



### pkg 模块

> 安装软件包

#### top.sls 文件

```bash
cat > /srv/salt/_modules/top.sls << EOF
base: 
  "*": 
    - sys.pkg
EOF
```



#### pkg.sls

```bash
cat > /srv/salt/_modules/sys/pkg.sls << EOF
下载软件包 vim-enhanced:
  pkg.downloaded:
    - name: vim-enhanced
    - version: 2:7.4.629-8.el7_9

更新软件包 vim-enhanced:
  pkg.uptodate:
    - name: vim-enhanced


安装软件包httpd:
  pkg.installed:
    - name: httpd
    - version: 2.4.6-97.el7.centos


卸载软件包 vim-enhanced:
  pkg.removed:
    - name: vim-enhanced

EOF

```



### service 模块

> 服务管理



#### top.sls 文件

```bash
cat > /srv/salt/_modules/top.sls << EOF
base: 
  "*": 
    - sys.service
EOF
```



#### service.sls

```bash
cat > /srv/salt/_modules/sys/service.sls << EOF
停止httpd服务,并关闭自启动:
  service.dead:
    - name: httpd
    - enable: False
    - init_delay: 20  # 延时20s

启动httpd服务,并开启自启动:
  service.running:
    - name: httpd
    - enable: True
EOF
```



### cmd 模块

#### cmd.sls

```yaml
cmd 模块运行命令测试:
  cmd.run:
    - name: sh /tmp/a.sh

cmd touch 创建文件测试:
  cmd.run:
    - name: touch /tmp/fooo
    - creates: /tmp/fooo

```



### 运行

```bash
salt '*' state.highstate
```



# 4.  Pillars & Grains 

https://blog.51cto.com/nginxs/1909012

## pillar 

>  Pillars: 用来自定义一些变量在minion中



master 配置`pillars` :

```bash
cat >> /etc/salt/master.d/roots.conf << EOF
# 开启pillar
pillar_opts: True
# pillar root 目录配置
pillar_roots:
  base:
    - /srv/pillar

EOF

# 重启master
systemctl restart salt-master


# 创建 pillar_roots base 目录:
mkdir -p /srv/pillar

```

### 目录结构说明

```bash
tree /srv/pillar/

/srv/pillar/
├── server.sls
└── top.sls

0 directories, 2 files

```



### top.sls

```yaml
base:
  'slave*':
    - server        # server.sls
```

### server.sls

```yaml
server_host: 192.168.0.10
server_name: omg
server_password: password123
server_port: 9527

```



### 运行查看

```bash
salt 'slave-01' pillar.data  server_host server_name server_password server_port
# 或
# salt 'slave-01' pillar.items  server_host server_name server_password server_port

# 结果如下:
slave-01:
    ----------
    server_host:
        192.168.0.10
    server_name:
        omg
    server_password:
        password123
    server_port:
        9527


```



## Grains

> 采集系统值(操作系统、域名、IP地址、内核、操作系统类型、内存和许多其他系统属性) 
>
> 在minion中`/etc/salt/minion.d/xxx.conf`  定义值,供服务端采集.
>
> 

 可以简单地在state定义文件中通过这种方式引用grains数据`{{ grains['key'] }}`  



```bash
# 自定义值供服务端采集
cat > /etc/salt/minion.d/grains.conf  << EOF
grains:
  name: wukong.sun
  age: 100
  addr: SH

EOF

# 重启minion
systemctl restart salt-minion

# 列出可用的grains
salt '*' grains.ls

# 查看
salt '*' grains.items
```



##  pillar与grains的使用示例

> 在模板文件中使用pillar与grains中的变量



目录文件说明:

```
tree /srv/salt/_modules/

/srv/salt/_modules/
├── sys
│   ├── files
│   │   ├── grains.template
│   ├── grains.sls
└── top.sls

```



### top.sls

```yaml
base: 
  "*": 
    - sys.grains
```



### grains.sls

```yaml
模板文件中使用pillar与grains中的变量:
  file.managed:
    - name: /tmp/grains.txt
    - source: salt://sys/files/grains.template
    - mode: 644
    - user: root
    - group: root
    - template: jinja

```

### grains.template

```
files/grains.template 
the name is {{ grains['name'] }}
the age  is {{ grains['age'] }}
the server_name is {{ pillar['server_name'] }}
the server_port is {{ pillar['server_port'] }}
--- ---
the OS is {{ grains['os'] }}
```

### 运行

```bash
salt '*' state.highstate
```



# 5. 一个综合使用案例

https://docs.saltproject.io/en/latest/ref/states/all/salt.states.file.html#module-salt.states.file

- 需要 开启pillar

## install JDK8

> 1. 从master上copy源文件(jdk包) 到 slave节点
> 2. 在slave节点执行解压操作
> 3. 在slave节点的 `/etc/profile.d/` 目录下新加 `java8.sh` 
> 4. 向 `java8.sh` 中增加环境变量

### 目录结构说明

```
tree /srv/salt/_modules
/srv/salt/_modules
└── jdk
    ├── install_jdk.sls
    └── src
        └── jdk-8u271-linux-x64.tar.gz

2 directories, 2 files

```

### install_jdk.sls

```yaml
1. copy jdk package :
  file.managed:
    - name: /usr/local/src/jdk-8u271-linux-x64.tar.gz
    - source: salt://jdk/src/jdk-8u271-linux-x64.tar.gz

2. 解压:
  cmd.run:
    - name: tar -xvf /usr/local/src/jdk-8u271-linux-x64.tar.gz  -C /usr/local/

3. 软连接设置:
  file.symlink:
    - name: /usr/local/jdk8
    - target: /usr/local/jdk1.8.0_271

4. 创建环境变量文件java8.sh:
  cmd.run:
    - name: touch /etc/profile.d/java8.sh
    - creates: /etc/profile.d/java8.sh

5. 配置jdk环境变量:
  file.append:
    - name: /etc/profile.d/java8.sh
    - makedirs: True
    - text:
      - export JAVA_HOME=/usr/local/jdk8
      - export PATH=${JAVA_HOME}/bin/:${PATH}
```



### :执行

```bash
salt 'slave-*' state.sls jdk.install_jdk
```



# 6. SaltStack api 使用

> 参考: https://zhuanlan.zhihu.com/p/343762278

## 安装 salt-api

```bash
# 安装 salt-api
yum install -y salt-api
```



## 配置

> 在master 上配置

`/etc/salt/master.d/api.conf` 

```yaml
rest_cherrypy:
  port: 8181
  host: 0.0.0.0
  disable_ssl: True
  #ssl_crt: /etc/pki/tls/certs/localhost.crt
  #ssl_key: /etc/pki/tls/private/localhost_nopass.key


external_auth:
  pam:
    salttest:
      - .*
      - '@runner'
      - '@wheel'
      - '@jobs'

```

创建登录账号:

```bash
useradd -M -s /sbin/nologin salttest
echo salttest | passwd salttest --stdin
```



## 启动

```bash
systemctl restart salt-master

systemctl start salt-api

# 查看监听端口
ss -tanl
```



## 验证

```bash
# 验证login登录，获取token字符串
curl -sS http://localhost:8181/login \
-H 'Accept: application/x-yaml' \
-d username=salttest \
-d password=salttest  \
-d eauth=pam


# 通过api 执行 test Ping 测试
curl -sSk http://localhost:8181 \
-H 'Accept: application/x-yaml' \
-H 'X-Auth-Token: e1169f7b360a8676b8a7c5c3b59fbd1c0004f7bf'  \
-d client=local \
-d tgt='slave*' \
-d fun=test.ping


# 通过api执行 cmd.run
curl -sSk http://localhost:8181 \
-H 'Accept: application/x-yaml' \
-H 'X-Auth-Token: e1169f7b360a8676b8a7c5c3b59fbd1c0004f7bf'  \
-d client=local \
-d tgt='slave-02' \
-d fun=cmd.run \
-d arg='ls -l /tmp'

```







```bash
# deploy.sls
1. copy artifact :
  file.managed:
    - name: /root/deploy_salt/{{ salt['pillar.get']('name') }}
    - source: salt://deploy/src/{{ salt['pillar.get']('name') }}



# 执行
salt '*'  state.highstate pillar='{name: aaa.jar}'
```

