---
title: nginx ldap 认证
date: 2021-09-15 13:33:03
tags:
- ldap
- nginx
---
# 0. 环境说明
- centos 7.3
- nginx 1.20.1

<!-- more -->

# 1. 编译nginx-auth-ldap 模块
```bash
# 安装依赖
yum install -y openldap-devel

# 下载nginx-auth-ldap 模块
cd /usr/local/src
git clone https://github.com/kvspb/nginx-auth-ldap.git

# 重新编译nginx
# 查看原nginx的编译参数
nginx -V

# 下面要加上你原来的编译参数
./configure --add-module=/usr/local/src/nginx-auth-ldap
make

# 备份
cp /usr/local/nginx/sbin/nginx{,.bak}
# 替换
cp objs/nginx  /usr/local/nginx/sbin/nginx

```

# 2. 配置nginx-ldap认证

```bash
    ldap_server mytest {
        url ldap://10.2.20.92:389/DC=ylb,DC=com?cn?sub?(objectClass=person);
        # 配置你的ldap RootDN 信息
        binddn "cn=root,dc=ylb,dc=com";
        # 密码
        binddn_passwd root@pw;
        #group_attribute People;
        #group_attribute_is_dn on;
        require valid_user;
    }
    server {
        listen       8080;
        server_name  _;
        root         /usr/local/nginx/html;
        location / {
            auth_ldap "Forbidden";
            auth_ldap_servers mytest;
        }
    }
```
