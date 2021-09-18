---
title: ldap安装与配置
date: 2021-09-13 17:13:43
tags:
- ldap
- devops
- phpldapadmin
---

# 0. ldap 说明

目录规则:

|--域
|--|---公司
|--|----|----分公司
|--|----|-----|----部门
|--|----|-----|-----|-----用户

对应的Distinguished Name显示如下：
```
CN=user1,OU=部门,OU=分公司,OU=公司,DC=xxx,DC=com
```

<!-- more -->

# 1. 安装LDAP

```bash
yum install openldap openldap-clients openldap-servers

cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
# start
systemctl start slapd

```



# 2. 配置



## 2.1 导入schema
> 不然下边在ldapadd或ldapmodify时会报错.
```bash
find  /etc/openldap/schema/ -type f -name "*.ldif" -exec ldapadd -Y EXTERNAL -H ldapi:/// -f {} \;

```
## 2.2 修改RootDN,RootPW
```bash
# 默认安装后RootDN为cn=Manager,dc=my-domain,dc=com，RootPW为空，这俩分别相当于用户名和密码。
# 查看
ldapsearch -H ldapi:// -LLL -Q -Y EXTERNAL -b "cn=config" "(olcRootDN=*)" dn olcRootDN olcRootPW

# 准备LDIF(LDAP Data Interchange Format)文件
PASSWORD="root@pw"
LDAP_Root_DN='cn=root,dc=ylb,dc=com'
LDAP_Root_PW=`slappasswd -s ${PASSWORD}`

cat > /root/ldap/rootpw.ldif << EOF
dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: ${LDAP_Root_PW}
-
replace: olcRootDN
olcRootDN: ${LDAP_Root_DN}
EOF

# 执行,使配置生效
ldapadd -Y EXTERNAL -H ldapi:/// -f /root/ldap/rootpw.ldif

# 查验是否生效
ldapsearch -H ldapi:// -LLL -Q -Y EXTERNAL -b "cn=config" "(olcRootDN=*)" dn olcRootDN olcRootPW
```

## 2.3 修改Base DN
```bash
# 将默认dc=my-domain,dc=com替换成自己的DN。
LDAP_Root_DN='cn=root,dc=ylb,dc=com'

# Base DN
LDAP_BASE_DN='dc=ylb,dc=com'
cat > /root/ldap/base.ldif << EOF
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external
 ,cn=auth" read by dn.base="${LDAP_Root_DN}" read by * none


dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: ${LDAP_BASE_DN}
EOF

# 执行,使配置生效
ldapadd -Y EXTERNAL -H ldapi:/// -f /root/ldap/base.ldif

```

## 2.4 权限控制
- 只允许自己和管理员修改密码，禁止匿名bind
- 自己只能查看自己的信息，管理员有所有权限

```bash
LDAP_BASE_DN='dc=ylb,dc=com'
cat > /root/ldap/acl.ldif << EOF
dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: to attrs=userPassword,shadowLastChange by self write by dn="cn=root,${LDAP_BASE_DN}" write by anonymous auth by * none
olcAccess: to * by self read by dn="cn=root,${LDAP_BASE_DN}" write by * none
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f /root/ldap/acl.ldif
```

## 2.5 启用memberOf overlay
```bash
cat > /root/ldap/memberof.ldif << EOF
# Load memberof module
dn: cn=module{0},cn=config
objectClass: olcModuleList
objectclass: top
olcModuleLoad: memberof

# Backend memberOf overlay
dn: olcOverlay={0}memberof,olcDatabase={2}hdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcMemberOf
olcOverlay: {0}memberof
olcMemberOfDangling: ignore
olcMemberOfRefInt: TRUE
olcMemberOfGroupOC: groupOfNames
olcMemberOfMemberAD: member
#olcMemberOfGroupOC: groupOfUniqueNames
#olcMemberOfMemberAD: uniqueMember
olcMemberOfMemberOfAD: memberOf
EOF

# 执行,使配置生效
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f /root/ldap/memberof.ldif

# 验证memberof模块已经加载
slapcat -n 0 | grep olcModuleLoad
```

## 2.6 创建组织结构

```bash
#Base DN
LDAP_BASE_DN='dc=ylb,dc=com'

cat > /root/ldap/organization.ldif << EOF
dn: ${LDAP_BASE_DN}
objectClass: top
objectClass: dcObject
objectClass: organization
o: Xnile

dn: ou=people,${LDAP_BASE_DN}
objectClass: organizationalUnit
ou: People

dn: ou=group,${LDAP_BASE_DN}
objectClass: organizationalUnit
ou: Group
EOF

# 执行
ldapadd -x -D cn=root,dc=ylb,dc=com -W -f /root/ldap/organization.ldif

```


## 2.7 成员管理-memberof

### 2.7.1 添加用户(user1)
```bash
#密码
LDAP_USER_PW=`slappasswd -s 123456`
#Base DN
LDAP_BASE_DN='dc=ylb,dc=com'

cat > /root/ldap/user1.ldif << EOF
dn: cn=user1,ou=people,${LDAP_BASE_DN}
cn: user1
givenName: user1
sn: user1
uid: user1
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/user1
mail: user1@ylb.com
objectClass: top
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
loginShell: /bin/bash
userPassword: ${LDAP_USER_PW}
EOF

# 执行
ldapadd -x -D cn=root,dc=ylb,dc=com -W -f /root/ldap/user1.ldif
```

### 2.7.2 添加组(devops)
```bash

#Base DN
LDAP_BASE_DN='dc=ylb,dc=com'
cat > /root/ldap/group.ldif <<EOF
dn: cn=devops,ou=group,${LDAP_BASE_DN}
objectClass: groupOfnames
cn: devops
description: All users
member: cn=user1,ou=people,${LDAP_BASE_DN}
EOF

# 执行
ldapadd -x -D cn=root,dc=ylb,dc=com -W -f /root/ldap/group.ldif


# 验证
PASSWORD="root@pw"
ldapsearch -x -LLL -D cn=root,dc=ylb,dc=com -w $PASSWORD -b cn=user1,ou=people,dc=ylb,dc=com dn memberof

```

### 2.7.3 删除
```bash
ldapdelete -x -D "cn=root,dc=ylb,dc=com" -W  "cn=user1,ou=people,dc=ylb,dc=com"

```

### 2.7.4 创建一个只读用户
> 其他系统要想接入LDAP需要有一个账号来连接LDAP库获取库的信息，为了安全考虑一般不太推荐直接使用RootDN和RootPW而应该单独创建一个只读的用户专门用来做三方系统接入。

```bash
# 1. 添加用户
#密码
LDAP_READONLY_USER_PW=`slappasswd -s readonly@pw`

#Base DN
LDAP_BASE_DN='dc=ylb,dc=com'
cat > /root/ldap/readOnly.ldif << EOF
dn: cn=readonly,${LDAP_BASE_DN}
cn: readonly
objectClass: simpleSecurityObject
objectClass: organizationalRole
description: LDAP read only user
userPassword: ${LDAP_READONLY_USER_PW}
EOF

# 执行
ldapadd -x -D cn=root,dc=ylb,dc=com -w root@pw -f /root/ldap/readOnly.ldif

# 2. 设置权限

LDAP_BASE_DN='dc=ylb,dc=com'
cat > /root/ldap/readonly-user-acl.ldif << EOF
dn: olcDatabase={2}hdb,cn=config
changetype: modify
delete: olcAccess
-
add: olcAccess
olcAccess: to attrs=userPassword,shadowLastChange by self write by dn="cn=admin,${LDAP_BASE_DN}" write by anonymous auth by * none
olcAccess: to * by self read by dn="cn=admin,${LDAP_BASE_DN}" write by dn="cn=readonly,${LDAP_BASE_DN}" read by * none
EOF

# 执行
ldapmodify -Y EXTERNAL -H ldapi:/// -f /root/ldap/readonly-user-acl.ldif

```

## 2.8 调试

`ldapmodify -vnf <NAME.ldif> `用这个命令可以效验ldif文件是否有错误。



## 2.9 ldap ssl

```bash
# 生成ca证书
cd /etc/openldap/certs/
openssl genrsa -out rootCA.key 2048
openssl req -x509 -new -nodes -subj "/C=CN/ST=ShangHai/L=ShangHai/O=ldap/OU=ylb/CN=ldap-ca"  -key rootCA.key -sha256 -days 1024 -out rootCA.pem
COPY

# 生成ldap证书请求
openssl genrsa -out ldap.key 2048
openssl req -new -subj "/C=CN/ST=ShangHai/L=ShangHai/O=ldap/OU=ylb/CN=ldap-server.ylb.com" -key ldap.key -out ldap.csr
COPY
# 签发ldap证书
openssl x509 -req -in ldap.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out ldap.crt -days 3650 -sha256



cat > certs.ldif << EOF
dn: cn=config
changetype: modify
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/openldap/certs/rootCA.pem

dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/ldap.crt

dn: cn=config
changetype: modify
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/ldap.key

EOF


ldapmodify -Y EXTERNAL  -H ldapi:/// -f certs.ldif

# 修改配置
vi /etc/sysconfig/slapd
SLAPD_URLS="ldapi:/// ldap:/// ldaps:///"

# 重启
systemctl restart slapd
```




# 3. 安装phpldapadmin

```bash
yum install -y phpldapadmin

```
## 3.1 修改配置
修改 `/etc/httpd/conf.d/phpldapadmin.conf` 

```html
#
#  Web-based tool for managing LDAP servers
#

Alias /phpldapadmin /usr/share/phpldapadmin/htdocs
Alias /ldapadmin /usr/share/phpldapadmin/htdocs

<Directory /usr/share/phpldapadmin/htdocs>
  <IfModule mod_authz_core.c>
    # Apache 2.4
    # Require local
    Require all granted // 我们用的是apache 2.4,修改这行.
  </IfModule>
  <IfModule !mod_authz_core.c>
    # Apache 2.2
    Order Deny,Allow
    Deny from all
    Allow from 127.0.0.1
    Allow from ::1
  </IfModule>
</Directory>

```
## 3.2 修改配置,用dn登录

修改 `/etc/phpldapadmin/config.php` 



```bash
// 使用dn 登录.   398行
$servers->setValue('login','attr','dn');

// 关闭匿名登录. 460行
$servers->setValue('login','anon_bind',true);

// 设置用户的唯一性. 519 行
$servers->setValue('unique','attrs',array('mail','uid','uidNumber','cn','sn'));


# 访问
# http://YOUR_IP/ldapadmin/
# user: cn=user1,ou=people,dc=ylb,dc=com
# password: root@pw
```



# 4. java api

- https://nightlies.apache.org/directory/api/2.1.0/apidocs/



# 参考
- [Centos7.3 phpldapadmin 安装和使用](https://blog.csdn.net/u011026329/article/details/79160809)
- [CentOS 7搭建OpenLDAP Server](https://blog.dianduidian.com/post/openldap%E6%90%AD%E5%BB%BA/)

