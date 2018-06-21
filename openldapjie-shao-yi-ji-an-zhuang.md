# 本章不会深入去讲解openldap协议

## 介绍

OpenLDAP是[轻型目录访问协议](https://baike.baidu.com/item/轻型目录访问协议)（Light weight Directory Access Protocol，LDAP）的自由和[开源](https://baike.baidu.com/item/开源)的实现，在其OpenLDAP许可证下发行，并已经被包含在众多流行的Linux发行版中。相对应的是微软的AD（Windows Active Directories）。实现的也是AD相对简化的功能。虽然相对AD简化，轻量，但并不意味着其功能弱。目前市场上很多商业产品都是通过内置ldap，从而实现统一的用户管理。

openLDAP可以与VPN结合实现统一接入认证，可以与堡垒机结合实现账号管理与审计，可以与系统结合从而保障系统安全。本书的多处章节的实例都会使用到ldap。

## 安装

1.准备CentOS7.4系统，关闭firewall，关闭selinux。

```
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
setenforce 0
yum remove -y firewall
```

2.安装以下软件

```
yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel
```

3.启动LDAP服务，并设置开机启动

```
systemctl start slapd.service
systemctl enable slapd.service
```

4.验证LDAP服务是否启动

```
# ss -antup | grep -i 389
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      1520/slapd          
tcp6       0      0 :::389                  :::*                    LISTEN      1520/slapd
```

### 设置LDAP root密码 {#设置ldap-root密码}

1.运行以下命令生成root密码。这个密码后面配置需要用到，请保存好。

```
# slappasswd
New password: 
Re-enter new password: 
{SSHA}d/thexcQUuSfe3rx3gRaEhHpNJ52N8D3
```

### 配置OpenLDAP服务 {#配置openldap服务}

OpenLDAP服务的配置文件放在`/etc/openldap/slapd.d/`目录下。我们首先需要更新`olcSuffix`和`olcRootDN`变量。

**olcSuffix**- 数据库后缀，通常它是你服务的域名。

**olcRootDN**- Root Distinguished Name \(DN\)。root用户的DN。

**olcRootPW**- root 密码。

创建一个.ldif文件并添加以下内容

```
# vim db.ldif

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=example,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=root,dc=example,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}S41s5yYWezC3Cd+B+7nEGQlkCk0LQrv2
```

保存并退出，让该配置在LDAP服务器上生效。

```
# ldapmodify -Y EXTERNAL -H ldapi:/// -f db.ldif
SASL/EXTERNAL authentication started
SASL username:gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={2}hdb,cn=config"
modifying entry "olcDatabase={2}hdb,cn=config"
modifying entry "olcDatabase={2}hdb,cn=config"
```

配置root用户的monitor

```
# vim monitor.ldif

dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external, cn=auth" read by dn.base="cn=root,dc=example,dc=com" read by * none
```

完成编辑后使之生效

```
ldapmodify -Y EXTERNAL  -H ldapi:/// -f monitor.ldif
```

### 创建LDAP证书 {#创建ldap证书}

接下来生成LDAP服务器的自签名证书，输入以下命令将在`/etc/openldap/certs/`目录下生成证书和私钥。

```
# openssl req -new -x509 -nodes -out /etc/openldap/certs/ldap.cert -keyout /etc/openldap/certs/ldap.key -days 365
Generating a 2048 bit RSA private key
...............................................................................................................................+++
....................................+++
writing new private key to '/etc/openldap/certs/ldap.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Zhejiang
Locality Name (eg, city) [Default City]:Hangzhou
Organization Name (eg, company) [Default Company Ltd]:Company
Organizational Unit Name (eg, section) []:IT
Common Name (eg, your name or your server's hostname) []:example.com
Email Address []:admin@example.com
```

修改用户组

```
chown -R ldap:ldap /etc/openldap/certs/ldap.*
```

新建certs.ldif用来配置LDAP使用自签名证书来进行安全会话

```
# vi certs.ldif

dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/ldap.cert

dn: cn=config
changetype: modify
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/ldap.key
```

将配置导入服务器

```
ldapmodify -Y EXTERNAL  -H ldapi:/// -f certs.ldif
```

验证配置

```
slaptest -u
```

正常情况下会显示如下结果表明验证完成

```
config file testing succeeded
```

### 设置LDAP数据库 {#设置ldap数据库}

将示例数据库配置文件复制到`/var/lib/ldap`下，并更改用户组。

```
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown ldap:ldap /var/lib/ldap/*
```

添加cosine和nis模式

```
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif 
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

给你的域生成一个base.ldif

```
# vim base.ldif

dn: dc=example,dc=com
dc: example
objectClass: top
objectClass: domain

dn: cn=root,dc=example,dc=com
objectClass: organizationalRole
cn: root
description: LDAP Manager

dn: ou=People,dc=example,dc=com
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=example,dc=com
objectClass: organizationalUnit
ou: Group
```

添加这个base结构

```
ldapadd -x -W -D "cn=root,dc=example,dc=com" -f base.ldif
```

到此，LDAP的基础配置完成，并且添加了root用户。

### 添加LDAP服务到防火墙 {#添加ldap服务到防火墙}

把LDAP加到firewalld的服务里。

```
# firewall-cmd --add-service=ldap --permanent
success
# firewall-cmd --reload
success
```

服务端的配置已经完成。接下来你就可以使用任意的ldap客户端连接。端口为389，使用root DN`cn=root,dc=example,dc=com`进行登录。

