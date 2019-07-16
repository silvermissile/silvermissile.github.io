---
layout:     post
title:      presto配置ldap用于用户认证
subtitle:   HTTPS 证书
date:       2019-07-16
author:     xuefly
catalog: true
tags:
    - 大数据
    - presto
    - ldap
    - olap
    - https
---

# 1  说明
- ldap支持prosto cli,jdbc odbc这几种连接的时候使用用户名密码
- presto可以只跟coordinator相关，worker的配置不变。只验证客户端到coordinate，不验证各个节点
- 要求LDAPS，所以ldap server必须支持TLS。也就是说presto coordinator和ldap server 之间通信加密。
- ldap的认证是基于https提供的，所以必须哟啊配置客户端到presto coordinatorde https连接。



![pic](https://raw.githubusercontent.com/silvermissile/silvermissile.github.io/master/img/post/2019-07-15-prestoldap.jpg)

总数配置分两步独立进行
- presto coordinator 跟ldap server 之间的连接配置
- presto client 跟 coordinator之间连接配置


# 2  presto 服务端
## 2.1  连接ldap服务配置
### 导入ldapserver节点的证书
ldapserver证书：
>dc.cer的签名是：
>Owner: CN=al.al.com
>Issuer: CN=al-AL-CA-1, DC=al, DC=com
>所以在`/etc/hosts`中要配置：`ldapseverip  al.al.com`

- **导入ldap server证书，persto 服务要想通过ldaps与ldap server通信，必须要导入。**
``` sh
keytool -import -keystore /javahome/jre/lib/security/cacerts -trustcacerts -alias ldap -file /javahome/jre/lib/security/dc.cer
#password:changeit
```

- 验证导入成功
``` sh
keytool -list -keystore   /javahome/jre/lib/security/cacerts
keytool -list -keystore   /javahome/jre/lib/security/cacerts | grep ldap
```




### 修改配置文件
#### hosts
``` sh
ldapseverip  al.al.com
```
>坑太多，注释掉的都不能用。
参考：
- [How to fix javax.net.ssl.SSLHandshakeException: java.security.cert.CertificateException: No subject alternative names present](https://unix.stackexchange.com/questions/322883/how-to-correctly-set-hostname-and-domain-name)
- [how-to-fix-javax-net-ssl-sslhandshakeexception-java-security-cert-certificateexception-no-subject-alternative-names-present/](http://www.littlebigextra.com/how-to-fix-javax-net-ssl-sslhandshakeexception-java-security-cert-certificateexception-no-subject-alternative-names-present/)


#### password-authenticator
```  properties
password-authenticator.name=ldap

# 跟证书 hosts文件中都要一致
ldap.url=ldaps://al.al.com:636

ldap.user-bind-pattern=${USER}@al.com

#参考ldap服务器
ldap.group-auth-pattern=(&(objectClass=person)(sAMAccountName=${USER})(memberof=CN=presto,OU=group,DC=al,DC=com))

#必需
ldap.user-base-dn=DC=al, DC=com
```
正常启动，说明import 这步是完成正确的。
presto coordinator 跟ldap server 之间的连接配置完成。

## 为客户端配置HTTPS服务

### ~~使用自签名的证书（弃用）~~
也记录下过程，但是配置完，报下面的错误：
``` sh
/home/ubuntu/scripts/presto_cluster/presto  \
--server https://prestoserver:8443 \
--keystore-path /javahome/jre/lib/security/presto_keystore_new0326.jks  \
--keystore-password 1qaz2wsx \
--catalog hive \
--schema default \
--user username \
--password

172.31.17.54  prestoserver

presto:default> show sheamas;
Error running command: javax.net.ssl.SSLHandshakeException: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
presto:default>
```
#### 生成presto server 的keystore
``` sh
keytool -genkeypair -alias ubuntu -keyalg RSA -keystore keystore.jks
```
>注意：
>What is your first and last name?
>  [Unknown]:  prestoserver




#### 附加说明
>别名
>What is a keystore alias?
>Solution
>All keystore entries (key and trusted certificate entries) are accessed via unique aliases. Aliases are case-insensitive; the aliases Hugo and hugo would refer to the same keystore entry.
>
>An alias is specified when you add an entity to the keystore using the -genkey command to generate a key pair (public and private key) or the -import command to add a certificate or certificate chain to the list of trusted certificates. Subsequent keytool commands must use this same alias to refer to the entity.


命令汇总：
``` sh
# 生成jks
keytool -genkeypair -alias prestoservernew -keyalg RSA -keystore presto_keystore_new.jks

# 导出cer证书
keytool -keystore /javahome/jre/lib/security/presto_keystore_new.jks -export -alias prestoservernew -file /tmp/prestoservernew.cer

# 导入证书到truststore
keytool -import -keystore /javahome/jre/lib/security/cacerts -trustcacerts -alias prestoservernew -file /tmp/prestoservernew.cer

# 删除信任证书
keytool -delete -alias prestoservernew -keystore  /javahome/jre/lib/security/cacerts


#检查导入成功：
keytool  \
-keystore /javahome/jre/lib/security/cacerts  \
-storepass changeit \
-list

# 删除
keytool -delete -alias ldapservern2 -keystore  /javahome/jre/lib/security/cacerts
#changeit

#查看证书
ubuntu@prestoserver:~/data/presto-server-0.215/etc$ keytool -printcert -v -file   /javahome/jre/lib/security/dc.cer
Owner: CN=al.al.com
Issuer: CN=al-AL-CA-1, DC=al, DC=com
```
>这就是host文件为什么必须是`al.al.com`


### 使用购买的认证过的证书
#### 生成证书
需要把crt格式的证书转为jks格式。
发现上面的步骤不行，在运维出要了公司的证书文件：
- companyname_new.crt
- gd_bundle-g2-g1.crt
- server.key


- 方法一：在线转换
[SSL证书格式转换工具](https://www.chinassl.net/ssltools/convert-ssl.html)
![]()
- 方法二：命令行转换

  ``` sh
  openssl pkcs12 -export -export -chain -CAfile gd_bundle.crt  -in server.crt -inkey server.key -out server.p12 -name "server"
  keytool -rfc -list -keystore server.p12 -storetype pkcs12
  keytool -importkeystore -v -srckeystore  server.p12 -srcstoretype pkcs12 -srcstorepass changeit -destkeystore server.keystore -deststoretype jks -deststorepass changeit
  keytool -list -keystore server.keystore
  ```
  >参考：[Tomcat更换SSL证书方法-key和crt文件转换为jks](https://blog.csdn.net/blade2001/article/details/9787295?tdsourcetag=s_pctim_aiomsg)

####  配置证书
修改 config.properties
``` properties
httpserver.authentication.type=PASSWORD

httpserver.https.enabled=true
httpserver.https.port=8443

httpserver.https.keystore.path=/javahome/jre/lib/security/keystore.jks
httpserver.https.keystore.key=1qaz2wsx
```


#  3 presto 客户端

## 在各个客户端测试
### 浏览器
https://ip:8443/ui/

### 命令行

``` sh
# keystore-password keystore-password  不需要
# /javahome/jre/lib/security/presto_keystore_new0326.jks  体现在浏览器地址栏的标识

/home/ubuntu/scripts/presto_cluster/presto  \
--server https://prestoserver:8443 \
--catalog hive \
--schema default \
--user username \
--password
```




## 问题
### javax.net.ssl.SSLPeerUnverifiedException: Hostname prestoserver not verified:
``` sh
Error running command: javax.net.ssl.SSLPeerUnverifiedException: Hostname prestoserver not verified:
    certificate: sha256/R86RK3F8Lc5GZehK5c8HBweOdb4JUSx5WKyE7g77gOg=
    DN: CN=*.companyname.com, OU=Domain Control Validated
    subjectAltNames: [*.companyname.com, companyname.com]
presto:default>
```
> --server必须使用证书绑定的域名
