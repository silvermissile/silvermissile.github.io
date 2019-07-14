[toc]

# 1  说明
- ldap支持prosto cli,jdbc odbc，使用用户名密码
- presto只用配置coordinator，不用管worker
- 只验证了客户端到coordinate，没有验证各个节点
- 要求LDAPS，所以ldap server必须支持TLS。也就是说presto coordinator和ldap server 之间通信加密。
- client 和presto server之间应该加密
- ldap的认证是通过https提供的。连接成功了



# 2 presto 服务端
## 2.1 导入ldap服务端证书
**导入ldap server证书，persto 服务要想通过ldaps与ldap server通信，必须要导入。**
``` sh
keytool -import -keystore /usr/share/java/jdk1.8.0_181/jre/lib/security/cacerts -trustcacerts -alias ldap -file /usr/share/java/jdk1.8.0_181/jre/lib/security/dc.cer

changeit
```
>dc.cer的签名是：
>Owner: CN=al.al.com
>Issuer: CN=al-AL-CA-1, DC=al, DC=com
>所以在`/etc/hosts`中要配置：`164.52.75.136   al.al.com`


## 准备key和证书
### ~~使用自签名的证书~~
也记录下过程，但是配置完，报下面的错误：
``` sh
/home/ubuntu/scripts/presto_cluster/presto  \
--server https://testpresto.akulaku.com:8443 \
--keystore-path /home/ubuntu/data/jdk1.8.0_152/jre/lib/security/presto_keystore_new0326.jks  \
--keystore-password 1qaz2wsx \
--catalog hive \
--schema default \
--user zhaoxf \
--password

172.31.17.54  testpresto.akulaku.com

presto:default> show sheamas;
Error running command: javax.net.ssl.SSLHandshakeException: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
presto:default>
```
#### 生成presto server 的keystor
``` sh
keytool -genkeypair -alias ubuntu -keyalg RSA -keystore keystore.jks
```
>注意：
>What is your first and last name?
>  [Unknown]:  ip-172-31-17-54




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
keytool -keystore /usr/share/java/jdk1.8.0_181/jre/lib/security/presto_keystore_new.jks -export -alias prestoservernew -file /tmp/prestoservernew.cer

# 导入证书到truststore
keytool -import -keystore /usr/share/java/jdk1.8.0_181/jre/lib/security/cacerts -trustcacerts -alias prestoservernew -file /tmp/prestoservernew.cer

# 删除信任证书
keytool -delete -alias prestoservernew -keystore  /usr/share/java/jdk1.8.0_181/jre/lib/security/cacerts


#检查导入成功：
keytool  \
-keystore /usr/share/java/jdk1.8.0_181/jre/lib/security/cacerts  \
-storepass changeit \
-list

# 删除
keytool -delete -alias ldapservern2 -keystore  /usr/share/java/jdk1.8.0_181/jre/lib/security/cacerts
#changeit

#查看证书
ubuntu@ip-172-31-17-54:~/data/presto-server-0.215/etc$ keytool -printcert -v -file   /usr/share/java/jdk1.8.0_181/jre/lib/security/dc.cer
Owner: CN=al.al.com
Issuer: CN=al-AL-CA-1, DC=al, DC=com
```
>这就是host文件为什么必须是`al.al.com`


### 使用购买的认证过的证书
发现上面的步骤不行，在运维出要了公司的证书文件：
- akulaku_new.crt
- gd_bundle-g2-g1.crt
- server.key

需要把crt格式的证书转为jks格式。
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

## 修改配置文件
### 修改host文件
``` sh
#164.52.75.136   ldapserver
#164.52.75.136 al-AL-CA-1
#164.52.75.136 ldapserver.al-AL-CA-1
#164.52.75.136 ldapserver.al.al.com

164.52.75.136   al.al.com
```
>坑太多，注释掉的都不能用。
参考：
- [How to fix javax.net.ssl.SSLHandshakeException: java.security.cert.CertificateException: No subject alternative names present](https://unix.stackexchange.com/questions/322883/how-to-correctly-set-hostname-and-domain-name)
- [how-to-fix-javax-net-ssl-sslhandshakeexception-java-security-cert-certificateexception-no-subject-alternative-names-present/](http://www.littlebigextra.com/how-to-fix-javax-net-ssl-sslhandshakeexception-java-security-cert-certificateexception-no-subject-alternative-names-present/)

### config.properties
``` properties
http-server.authentication.type=PASSWORD

http-server.https.enabled=true
http-server.https.port=8443

http-server.https.keystore.path=/usr/share/java/jdk1.8.0_181/jre/lib/security/keystore.jks
http-server.https.keystore.key=1qaz2wsx
```
### password-authenticator
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







#  3 presto 客户端

## 在各个客户端测试
### 浏览器
https://13.250.59.8:8443/ui/
https://testpresto.akulaku.com:8443/ui/

https://www.al.al.com

https://54.179.144.205:8443/ui/

presto.akulaku.com
### 命令行

``` sh
# 在测试cdh 54测试

ubuntu@ip-172-31-17-54:/usr/share/java/jdk1.8.0_181/jre/lib/security$ /home/ubuntu/data/presto/presto-cli-0.217-executable.jar \
> --server https://testpresto.akulaku.com:8443 \
> --keystore-path /usr/share/java/jdk1.8.0_181/jre/lib/security/presto_keystore_new0326.jks  \
> --keystore-password 1qaz2wsx \
> --truststore-path /usr/share/java/jdk1.8.0_181/jre/lib/security/cacerts \
> --truststore-password changeit \
> --catalog hive \
> --schema default \
> --user zhaoxf \
> --password
Password:
presto:default> show catalogs;
   Catalog
--------------
 hive
 ignite
 mysql_offset
(3 rows)

Query 20190326_065819_00001_sgq3r, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0:02 [0 rows, 0B] [0 rows/s, 0B/s]

```

``` sh
# keystore-password keystore-password  不需要
# /usr/share/java/jdk1.8.0_181/jre/lib/security/presto_keystore_new0326.jks  体现在浏览器地址栏的标识

/home/ubuntu/scripts/presto_cluster/presto  \
--server https://testpresto.akulaku.com:8443 \
--catalog hive \
--schema default \
--user zhaoxf \
--password
```



### 通过driver
```
SSL
SSLKeyStorePath
SSLKeyStorePassword
```
## 问题
### javax.net.ssl.SSLPeerUnverifiedException: Hostname ip-172-31-17-54 not verified:
``` sh
/usr/share/java/jdk1.8.0_181/jre/lib/security$ /home/ubuntu/data/presto/presto-cli-0.217-executable.jar \
> --server https://ip-172-31-17-54:8443 \
> --keystore-path /usr/share/java/jdk1.8.0_181/jre/lib/security/presto_keystore_new0326.jks  \
> --keystore-password 1qaz2wsx \
> --truststore-path /usr/share/java/jdk1.8.0_181/jre/lib/security/cacerts \
> --truststore-password changeit \
> --catalog hive \
> --schema default \
> --user zhaoxf \
> --password
Password:
presto:default> show catalogs;
Error running command: javax.net.ssl.SSLPeerUnverifiedException: Hostname ip-172-31-17-54 not verified:
    certificate: sha256/R86RK3F8Lc5GZehK5c8HBweOdb4JUSx5WKyE7g77gOg=
    DN: CN=*.akulaku.com, OU=Domain Control Validated
    subjectAltNames: [*.akulaku.com, akulaku.com]
presto:default>
```
> --server必须使用证书绑定的域名
