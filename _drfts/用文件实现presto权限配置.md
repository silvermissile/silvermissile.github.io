> by zhaoxf on 2019-2-15

# 修改配置文件

## 1. presto

配置rule.json
``` json
{
  "catalog": "system",
  "allow": true
}
```
>system必须配置为true，否则DBeaver连接到presto不能得到元数据，报错
## 2. hive
### hive connector
``` properties
hive.security=file
security.config-file=/opt/cloudera/parcels/presto/etc/hive_access_control.json

# 每10s更新一次 修改之后不用重启
security.refresh-period=10s
```
### hive_access_control
#### schema规则
- 可以在tmp库创建表，owner为true，也不能删表，也不能查询
``` json
{
  "schema": "tmp",
  "owner": true
}
```
- 测试如下：
``` sql
presto:default> CREATE TABLE tmp.orders_lich (
             ->   orderkey bigint,
             ->   orderstatus varchar,
             ->   totalprice double,
             ->   orderdate date
             -> )
             -> WITH (format = 'ORC');
presto:default> drop table tmp.orders_lich;
Query 20190214_070257_00036_hseu7 failed: Access Denied: Cannot drop table tmp.orders_lich
presto:default> select * from tmp.orders_lich;
Query 20190214_070134_00032_hseu7 failed: Access Denied: Cannot select from table tmp.orders_lich
```
#### table规则
- 没有满足table规则，即便是能够创建表也不能删除，也不能查询表
- 该规则放到最后，一旦被提前匹配到，就没有权限。
``` json
{
  "privileges": []
}
```

- 给guokai整个userinfotmp库的查询权限
``` json
{
  "schema": "userinfotmp",
    "user": "guokai",
  "privileges": ["SELECT"]
},
```


- 给所有人整个shoeesy的查询权限
``` json
{
  "schema": "shoeesy",
    "table": ".*",
  "privileges": ["SELECT"]
}
```
- **presto的tabel配置liwei*、useractionlog*这种通配符不支持。**

### jvm.config
```  sh
-server
-Xmx10G
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+UseGCOverheadLimit
-XX:+ExplicitGCInvokesConcurrent
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError
-DHADOOP_USER_NAME=hive
```

>DHADOOP_USER_NAME=hive：必须配置为hive，跟hdfs数据仓库的用户对应，才能insert。
>不管连接presto的是哪个用户，执行insert语句的时候都是DHADOOP_USER_NAME的运行用户。
>在集群中presto 驻守进程的用户是ubuntu，是启动的时候使用的登录用户

#### 问题
``` sh
access deny :can not insert into  table
#这个是新问题， presto拒绝了。 但是又不是每次都会拒绝，时而可以时而不行。
```
发现是用户问题：`DHADOOP_USER_NAME=hive`有的配置正确，有的不正确。

# 部署配置文件
更新配置文件时，上传修改文件到`/home/ubuntu/data/xfapp/presto_config/`,然后用命令部署到各个节点即可。
>配置了`security.refresh-period=10s`后，修改rules.json或者hive_access_control.json 后，不用重启presto。


``` sh
ansible presto -m shell -a "ls -l  /opt/cloudera/parcels/presto/etc" ;
ansible presto --become  -mshell -a "chown -r  ubuntu:ubuntu   /opt/cloudera/parcels/presto/etc";

ansible presto -m copy  -a "src=/home/ubuntu/data/xfapp/presto_config/access-control.properties  dest=/opt/cloudera/parcels/presto/etc/";
ansible presto -m copy  -a "src=/home/ubuntu/data/xfapp/presto_config/rules.json  dest=/opt/cloudera/parcels/presto/etc/";
ansible presto -m copy  -a "src=/home/ubuntu/data/xfapp/presto_config/hive_access_control.json  dest=/opt/cloudera/parcels/presto/etc/";
ansible presto -m copy  -a "src=/home/ubuntu/data/xfapp/presto_config/hive.properties   dest=/opt/cloudera/parcels/presto/etc/catalog/"
ansible presto -m copy  -a "src=/home/ubuntu/data/xfapp/presto_config/mysql_akuloan.properties    dest=/opt/cloudera/parcels/presto/etc/catalog/"
ansible presto -m copy  -a "src=/home/ubuntu/data/xfapp/presto_config/jvm.config    dest=/opt/cloudera/parcels/presto/etc/"
ansible presto -m shell  -a "rm -f /opt/cloudera/parcels/presto/etc/catalog/mysql_akuloan.properties   "

```
