---
layout:     post
title:      利用hive的Create_timeAccesstime做数据质量管理
subtitle:   技术调研
date:       2020-07-07
author:     xuefly
catalog: true
tags:
    - 数据治理
    - hive
    - metadata


---
<!-- TOC -->

- [1. 需求：](#1-需求)
- [2. 现状](#2-现状)
  - [2.1 用HSQL 查](#21-用hsql-查)
    - [2.1.1](#211)
    - [2.1.2](#212)
  - [2.2 查 HIVE 的 metadata 数据库](#22-查-hive-的-metadata-数据库)
  - [2.3 在 hue 看。](#23-在-hue-看)
- [3 方案](#3-方案)
  - [3.1 修改一](#31-修改一)
  - [3.2 尝试二](#32-尝试二)
  - [3.3 遇到 bug](#33-遇到-bug)
- [4 原理 hook](#4-原理-hook)
- [5 测试](#5-测试)
  - [5.1 hivesql](#51-hivesql)
  - [5.2 presto](#52-presto)
  - [5.3 sparksql](#53-sparksql)
- [附](#附)
  - [解释 `hive.security.authorization.sqlstd.confwhitelist.append`](#解释-hivesecurityauthorizationsqlstdconfwhitelistappend)

<!-- /TOC -->
## 1. 需求：

当前的nna是作为 HDFS 的分析工具，不涉及 hive 的表。作为补充，hive 的元数据可以作为 hive 的表的分析来源。
场景：
- 根据创建时间、访问时间来实现数据的生命周期管理。例如，给归档提供参考.可以在元信息上根据这些进行数据生命周期管理。开发定时任务扫描出过期的数据表，邮件发出来。
- 提取每个 table 的 ower，可以在技术上找到指定的OWNER，可以配合在 landset 上指定表的负责人，在日常指定人来维护。
- 也许可以使用一些算法，来分析热点。

## 2. 现状

### 2.1 用HSQL 查
#### 2.1.1
**访问时间`transient_lastDdlTime`默认不会更新**
``` sql
-- You need to run the following command:

describe formatted <your_table_name>;
-- Or if you need this information about a particular partition:

describe formatted <your_table_name> partition (<partition_field>=<value>);
```



``` sql
show create table  someyewu_application_feature_sub_order;

createtab_stmt
CREATE TABLE ` someyewu_application_feature_sub_order`(
  `order_id` bigint COMMENT '订单id',
  `parent_order_id` bigint COMMENT '父订单id',
  `uid` bigint COMMENT '客户id',
  `create_time` bigint COMMENT '订单创建时间',
  `price_amt` decimal(20,2) COMMENT '消费分期本金',
  `monthly_installment_payment_amt` decimal(20,2) COMMENT '月供',
  `periods` int COMMENT '分期期数',
  `pay_offs` bigint COMMENT '还清次数',
  `province` string COMMENT '下单时GPS城市',
  `city` string COMMENT '下单时GPS地区',
  `login_ip` string COMMENT '下单IP',
  `device_id` string COMMENT '登录设备号',
  `etl_time` string COMMENT '数据仓库创建时间')
ROW FORMAT SERDE
  'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
STORED AS INPUTFORMAT
  'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
LOCATION
  'hdfs://nameservice1/user/hive/warehouse/ads.db/ someyewu_application_feature_sub_order'
TBLPROPERTIES (
  'COLUMN_STATS_ACCURATE'='true',
  'numFiles'='265',
  'numRows'='104786514',
  'rawDataSize'='1362224682',
  'totalSize'='9395667259',
  'transient_lastDdlTime'='1593554048') -- Your time zone: Wednesday, July 1, 2020 5:54:08 AM GMT+08:00


```
#### 2.1.2
**依赖于 HDFS，只有真正访问了数据，访问时间才会更新**
Hive数据仓库中随着越来越多业务方的使用，会产生非常多的库表。如何对数据生命周期进行管理成了很重要的工作。

- 经过验证发现，通过对Hive表执行以下语句会详细展现该表的很多统计信息，其中就有lastAccessTime。 (前提是该执行用户必须有对应表所在hdfs文件目录的读权限)
- 如果表只有元数据，在 HDFS 没有对应实际存在的数据，则会显示：`unknown`。
- 虽然hive可以查询到这些信息，但发现多访问几下居然没更新那个最后访问时间了。
- 原来是由于hdfs有个默认配置会只记录1小时精度的最后访问时间，参数为dfs.namenode.accesstime.precision（默认小时粒度）
- 就算客户端是通过spark、impala，一样可以反映到最后访问时间。原因是他们都会直接访问文件读取数据而被hdfs记录到。
``` sql
show table extended in ads like  someyewu_application_feature_sub_order;

tableName: someyewu_application_feature_sub_order
owner:hive
location:hdfs://nameservice1/user/hive/warehouse/ads.db/ someyewu_application_feature_sub_order
inputformat:org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat
outputformat:org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat
columns:struct columns { i64 order_id, i64 parent_order_id, i64 uid, i64 create_time, decimal(20,2) price_amt, decimal(20,2) monthly_installment_payment_amt, i32 periods, i64 pay_offs, string province, string city, string login_ip, string device_id, string etl_time}
partitioned:false
partitionColumns:
totalNumberFiles:265
totalFileSize:9395667259
maxFileSize:1727216880
minFileSize:21091082
lastAccessTime:1593553923503  -- Wednesday, July 1, 2020 5:52:03.503 AM GMT+08:00
lastUpdateTime:1593903269506  --Your time zone: Sunday, July 5, 2020 6:54:29.506 AM GMT+08:00


-- 这个不能获得时间
 show table extended like  "f_tedcom_installmentdb_a_20190825" ;
tab_name
tableName:f_tedcom_installmentdb_a_20190825
owner:hive
location:hdfs://nameservice1/user/hive/warehouse/ods.db/f_tedcom_installmentdb_a_20190825
inputformat:org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat
outputformat:org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat
columns:struct columns { i64 item_id, i64 sku_id, i64 count, string etl_time}
partitioned:false
partitionColumns:
totalNumberFiles:unknown
totalFileSize:unknown
maxFileSize:unknown
minFileSize:unknown
lastAccessTime:unknown
lastUpdateTime:unknown
```



参考：
- [dfs.namenode.accesstime.precision详解](https://blog.csdn.net/qq_35440040/article/details/105370755)
- [Hive表生命周期管理](https://blog.csdn.net/huanggang028/article/details/79032070)



###  2.2 查 HIVE 的 metadata 数据库
**默认情况下`LAST_ACCESS_TIME`一直为0**
[MySQL时间戳和时间的获取/相互转换/格式化](https://blog.csdn.net/lilongsy/article/details/79061639)

``` sql

  SELECT     t.TBL_NAME,     d.NAME ,     from_unixtime(t.CREATE_TIME) ,     from_unixtime(LAST_ACCESS_TIME),     t.OWNER FROM     hive.TBLS t,hive.DBS d     where     t.DB_ID=d.DB_ID and   t.TBL_NAME =" someyewu_application_feature_sub_order";
  +-----------------------------------+------+------------------------------+---------------------------------+-------+
  | TBL_NAME                          | NAME | from_unixtime(t.CREATE_TIME) | from_unixtime(LAST_ACCESS_TIME) | OWNER |
  +-----------------------------------+------+------------------------------+---------------------------------+-------+
  |  someyewu_application_feature_sub_order | ads  | 2020-01-09 11:39:52          | 1970-01-01 00:00:00             | hive  |
  +-----------------------------------+------+------------------------------+---------------------------------+-------+
  1 row in set (0.00 sec)

-- 查看所有表
SELECT
t.TBL_NAME,
d.NAME ,
from_unixtime(t.CREATE_TIME) ,
from_unixtime(LAST_ACCESS_TIME),
t.OWNER
FROM
hive.TBLS t,hive.DBS d
where 	t.DB_ID=d.DB_ID   ;



-- 查看f_tedcom_installmentdb_a_20190825
SELECT
from_unixtime(CREATE_TIME ),
from_unixtime(LAST_ACCESS_TIME),
t.OWNER,
t.TBL_NAME,
TBL_ID ,
d.NAME
FROM
hive.TBLS t,hive.DBS d
where 	t.DB_ID=d.DB_ID  and t.TBL_NAME ="f_tedcom_installmentdb_a_20190825" ;


SELECT  DISTINCT(t.OWNER) from hive.TBLS t  ;
```


### 2.3 在 hue 看。
可以在 detail 中看到`OWNER` `CreateTime`
![hive-meta-hue](https://gitee.com/xfly/imgbed/raw/master/img/post/hive-meta-hue.png)


## 3 方案
2.1.1中的方法。

### 3.1 修改一

1.hive-site.xml 的 Hive 服务高级配置代码段（安全阀）中添加2个配置`Hive Service Advanced Configuration Snippet (Safety Valve) for hive-site.xml`
``` properties
hive.security.authorization.sqlstd.confwhitelist=hive.exec.pre.hooks
hive.exec.pre.hooks=org.apache.hadoop.hive.ql.hooks.UpdateInputAccessTimeHook$PreExec
```
2.重启hive便可
我也遇到了这个问题：https://stackoverflow.com/questions/46149610/hive-update-lastaccesstime
怎么解决的呢？
### 3.2 尝试二
用 hsql 调试。
``` sql
set hive.security.authorization.sqlstd.confwhitelist.append;
set hive.security.authorization.sqlstd.confwhitelist;

set hive.exec.pre.hooks;
set hive.security.authorization.sqlstd.confwhitelist;
set hive.security.authorization.sqlstd.confwhitelist.append;
set hive.security.authorization.sqlstd.confwhitelist=hive.exec.pre.hooks;
set hive.exec.pre.hooks=org.apache.hadoop.hive.ql.hooks.UpdateInputAccessTimeHook$PreExec;
 show table extended like  "f_tedcom_installmentdb_a_20190825" ;
DESCRIBE formatted   f_tedcom_installmentdb_a_20190825 ;
 SELECT * FROM ods.f_tedcom_installmentdb_a_20190825 LIMIT 100;
 show create table f_tedcom_installmentdb_a_20190825 ;

show table extended in dwd like tedcom_hardware_wifi_new_details ;
show create table tedcom_hardware_wifi_new_details ;
SELECT * from  tedcom_hardware_wifi_new_details  limit 100 ;

set hive.security.authorization.sqlstd.confwhitelist.append=hive\.exec\.pre\.hooks
use recommend;



-- 设置属性
set hive.security.authorization.sqlstd.confwhitelist=hive.exec.pre.hooks;

set hive.exec.pre.hooks=org.apache.hadoop.hive.ql.hooks.UpdateInputAccessTimeHook$PreExec;
--这个属性不能再运行时修改。

-- 查看属性值
set hive.security.authorization.sqlstd.confwhitelist;
hive.security.authorization.sqlstd.confwhitelist=hive\.auto\..*|hive\.cbo\..*|hive\.convert\..*|hive\.exec\.dynamic\.partition.*|hive\.exec\..*\.dynamic\.partitions\..*|hive\.exec\.compress\..*|hive\.exec\.infer\..*|hive\.exec\.mode.local\..*|hive\.exec\.orc\..*|hive\.fetch.task\..*|hive\.hbase\..*|hive\.index\..*|hive\.index\..*|hive\.intermediate\..*|hive\.join\..*|hive\.limit\..*|hive\.mapjoin\..*|hive\.merge\..*|hive\.optimize\..*|hive\.orc\..*|hive\.outerjoin\..*|hive\.ppd\..*|hive\.prewarm\..*|hive\.skewjoin\..*|hive\.smbjoin\..*|hive\.stats\..*|hive\.tez\..*|hive\.vectorized\..*|mapred\.map\..*|mapred\.reduce\..*|mapred\.output\.compression\.codec|mapreduce\.job\.reduce\.slowstart\.completedmaps|mapreduce\.job\.queuename|mapreduce\.input\.fileinputformat\.split\.minsize|mapreduce\.map\..*|mapreduce\.reduce\..*



show table extended in ads like  someyewu_application_feature_sub_order;
show create table  someyewu_application_feature_sub_order;

```

> 可以看到有很多正则表达式用 `|` 分割，如果`set hive.security.authorization.sqlstd.confwhitelist=hive.exec.pre.hooks` 会把原来的覆盖,需要使用`hive.security.authorization.sqlstd.confwhitelist.append`






``` xml
<property>
   <name>hive.security.authorization.sqlstd.confwhitelist.append</name>
   <value>hive\.exec\.pre\.hooks</value>
</property>
<property>
   <name>hive.exec.pre.hooks</name>
   <value>org.apache.hadoop.hive.ql.hooks.UpdateInputAccessTimeHook$PreExec</value>
</property>
```
>需要注意的地方：
- 转义字符
- appended

参考：
- https://community.cloudera.com/t5/Support-Questions/how-to-set-more-than-one-parameters-in-to-quot-hive-security/td-p/142267
- http://www.hadoopadmin.co.in/hive/last-access-time-of-a-table-is-showing-zero/

### 3.3 遇到 bug
![hive_accee_time](https://gitee.com/xfly/imgbed/raw/master/img/post/hive_accee_time.png)
如果还遇到找不到表的问题,很可能是遇到了这个 bug [HIVE-18060UpdateInputAccessTimeHook fails for non-current database](https://issues.apache.org/jira/browse/HIVE-18060).
CDH 中直到这个版本才解决.[CDH 5.15.0 Release Notes](http://archive.cloudera.com/cdh5/cdh/5/hive-1.1.0-cdh5.15.0.releasenotes.html)
## 4 原理 hook
![](https://cwiki.apache.org/confluence/download/attachments/27362072/system_architecture.png)
//TD


## 5 测试


###  5.1 hivesql
满足预期，**不管是访问元数据还是数据，访问时间都会更新**
测试过程
``` sql



DESCRIBE formatted   f_tedcom_installmentdb_a_20190825 ;  --执行这个语句会更新LastAccessTime: 同时反映到数据库
col_name	data_type	comment
# col_name            	data_type           	comment
	NULL	NULL
item_id	bigint
sku_id	bigint
count	bigint
etl_time	string
	NULL	NULL
# Detailed Table Information	NULL	NULL
Database:           	ods                 	NULL
Owner:              	hive                	NULL
CreateTime:         	Mon Aug 26 07:49:51 UTC 2019	NULL
LastAccessTime:     	Tue Jul 07 02:50:16 UTC 2020	NULL
Protect Mode:       	None                	NULL
Retention:          	0                   	NULL
Location:           	hdfs://nameservice1/user/hive/warehouse/ods.db/f_tedcom_installmentdb_a_20190825	NULL
Table Type:         	MANAGED_TABLE       	NULL
Table Parameters:	NULL	NULL
	COLUMN_STATS_ACCURATE	false
	numFiles            	0
	numRows             	-1
	rawDataSize         	-1
	totalSize           	0
	transient_lastDdlTime	1594090216
	NULL	NULL
# Storage Information	NULL	NULL
SerDe Library:      	org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe	NULL
InputFormat:        	org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat	NULL
OutputFormat:       	org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat	NULL
Compressed:         	No                  	NULL
Num Buckets:        	-1                  	NULL
Bucket Columns:     	[]                  	NULL
Sort Columns:       	[]                  	NULL
Storage Desc Params:	NULL	NULL
	serialization.format	1



 show create table f_tedcom_installmentdb_a_20190825 ; --  会更新transient_lastDdlTime  同时更新数据库LAST_ACCESS_TIME
createtab_stmt
CREATE TABLE `f_tedcom_installmentdb_a_20190825`(
  `item_id` bigint,
  `sku_id` bigint,
  `count` bigint,
  `etl_time` string)
ROW FORMAT SERDE
  'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
STORED AS INPUTFORMAT
  'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
LOCATION
  'hdfs://nameservice1/user/hive/warehouse/ods.db/f_tedcom_installmentdb_a_20190825'
TBLPROPERTIES (
  'COLUMN_STATS_ACCURATE'='false',
  'numFiles'='0',
  'numRows'='-1',
  'rawDataSize'='-1',
  'totalSize'='0',
  'transient_lastDdlTime'='1594090281')  --2020/7/7 上午10:51:21

 SELECT * FROM ods.f_tedcom_installmentdb_a_20190825 LIMIT 100; -- 会更新数据库LAST_ACCESS_TIME

```


检查`LAST_ACCESS_TIME` 的变化。
``` sql
SELECT
  from_unixtime(CREATE_TIME),
  from_unixtime(LAST_ACCESS_TIME),
  t.OWNER,
  t.TBL_NAME,
  TBL_ID ,
  d.NAME
FROM
  hive.TBLS t,
  hive.DBS d
where
  t.DB_ID = d.DB_ID
  and t.TBL_NAME = "f_tedcom_installmentdb_a_20190825" ;
| from_unixtime(CREATE_TIME ) | from_unixtime(LAST_ACCESS_TIME) | OWNER | TBL_NAME                           | TBL_ID | NAME |
|:----------------------------|:--------------------------------|:------|:-----------------------------------|:-------|:-----|
| 2019-08-26 07:49:51.0       | 2020-07-07 03:06:03.0           | hive  | f_tedcom_installmentdb_a_20190825 | 8267   | ods  |

```


### 5.2 presto
//TD
### 5.3 sparksql
//TD




## 附
### 解释 `hive.security.authorization.sqlstd.confwhitelist.append`
[解释 hive.security.authorization.sqlstd.confwhitelist.append ](https://mapr.com/docs/61/Hive/UsingWhiteListwithFallbackAuthorizer.html)
> You can add an exception to Fallback Hive Authorizer restrictions using the hive.security.authorization.sqlstd.confwhitelist.append property.

>The hive.security.authorization.sqlstd.confwhitelist property is list of comma-separated Java regexes that you can append to. Appending to this list instead of updating the original list means that you can append to the default set by SQL-standard authorization instead of replacing it entirely.
>You can modify the configurations parameters that match these regexes when SQL-standard authorization is enabled.

>To get the default value, use the set <param> command. The hive.conf.restricted.list checks are still enforced after the white-list check.

>An example of a white-list configuration is as follows:
``` xml
<property>
  <name>hive.security.authorization.sqlstd.confwhitelist.append</name>
  <value>hive.reloadable.aux.jars.path</value>
</property>
```
>After adding this configuration to the hive-site.xml file, execute the following command:
set hive.reloadable.aux.jars.path=/path/to/jar
