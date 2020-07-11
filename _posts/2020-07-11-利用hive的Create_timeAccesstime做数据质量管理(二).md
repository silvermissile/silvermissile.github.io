---
layout:     post
title:      利用hive的Create_timeAccesstime做数据质量管理(二)
subtitle:   修改 CDH 的 hive 源码
date:       2020-07-11
author:     xuefly
catalog: true
tags:
    - hive
    - 源码


---


在[利用hive的Create_timeAccesstime做数据质量管理](https://silvermissile.github.io/2020/07/07/%E5%88%A9%E7%94%A8hive%E7%9A%84Create_timeAccesstime%E5%81%9A%E6%95%B0%E6%8D%AE%E8%B4%A8%E9%87%8F/)提到,为获取hive 元数据中的访问时间,经过一番折腾,因为使用的 hive 的版本存在 bug,功败垂成.

本文记录解决这个 bug 的过程.


 - 在[CDH 5.15.0 Release Notes](http://archive.cloudera.com/cdh5/cdh/5/hive-1.1.0-cdh5.15.0.releasenotes.html)可以发现这次发布有哪些变动.其中 遇到的 bug 的issue 是`[HIVE-18060]`
- 在[ hive-1.1.0-cdh5.15.0.CHANGES.txt](http://archive.cloudera.com/cdh5/cdh/5/hive-1.1.0-cdh5.15.0.CHANGES.txt)可以发现每次提交的记录.其中解决这个不够的提交是:`012c5513a293fd45d5966851b72e96e786ca00b8`
- 只要把这次提交,合并到正在使用的`dh5-1.1.0_5.14.0`hive 中,再重新编译部署即可.


### 合并提交
``` sh
git clone https://github.com/cloudera/hive.git
cd hive
#新建 cdh5-1.1.0_5.15.0 分支 与远程仓库的 cdh5-1.1.0_5.15.0 分支对应
git checkout -b   cdh5-1.1.0_5.15.0  origin/cdh5-1.1.0_5.15.0
git branch --contains 012c5513a293fd45d5966851b72e96e786ca00b8
git show 012c5513a293fd45d5966851b72e96e786ca00b8

#新建 cdh5-1.1.0_5.14.0  分支 与远程仓库的  origin/cdh5-1.1.0_5.14.0 分支对应
git checkout -b   cdh5-1.1.0_5.14.0  origin/cdh5-1.1.0_5.14.0
git cherry-pick 012c5513a293fd45d5966851b72e96e786ca00b8
```
### 编译源码
本来打算只打包这个组件`hive/ql`,报找不到类的错误.
在 源码的根目录把,编译所有模块解决问题,并编译成功.
` mvn clean package -Phadoop-2,dist -DskipTests -Dmaven.javadoc.skip=true`
参考:[从源码编译 hive](https://cwiki.apache.org/confluence/display/Hive/GettingStarted#GettingStarted-BuildingHivefromSource)
### 定位目标 jar
在`git show 012c5513a293fd45d5966851b72e96e786ca00b8`可以看到:
`a/ql/src/java/org/apache/hadoop/hive/ql/hooks/UpdateInputAccessTimeHook.java`
``` java
package org.apache.hadoop.hive.ql.hooks;

import java.util.Set;

import org.apache.hadoop.hive.ql.session.SessionState;
import org.apache.hadoop.security.UserGroupInformation;
import org.apache.hadoop.hive.ql.metadata.Hive;
import org.apache.hadoop.hive.ql.metadata.HiveException;
import org.apache.hadoop.hive.ql.metadata.Partition;
import org.apache.hadoop.hive.ql.metadata.Table;

/**
 * Implementation of a pre execute hook that updates the access
 * times for all the inputs.
 */
public class UpdateInputAccessTimeHook {
```
所以涉及到的类是:`org.apache.hadoop.hive.ql.hooks.UpdateInputAccessTimeHook`
由[此](https://jar-download.com/artifacts/org.apache.hive/hive-exec/3.1.0/source-code/org/apache/hadoop/hive/ql/hooks/UpdateInputAccessTimeHook.java)知
涉及到的 jar 包是:`hive-exec-1.1.0-cdh5.14.0.jar`


``` sh
#找到需要替换的目标 jar 包位置
root@cdh109:/opt/cloudera/parcels/CDH/lib/hive/lib# ll |grep exec
lrwxrwxrwx  1 root root    43 Jan  6  2018 hive-exec-1.1.0-cdh5.14.0.jar -> ../../../jars/hive-exec-1.1.0-cdh5.14.0.jar
lrwxrwxrwx  1 root root    29 Jan  6  2018 hive-exec.jar -> hive-exec-1.1.0-cdh5.14.0.jar
root@cdh109:/opt/cloudera/parcels/CDH/lib/hive/lib# ll   ../../../jars/hive-exec-1.1.0-cdh5.14.0.jar
-rw-r--r-- 1 root root 19598157 Jan  6  2018 ../../../jars/hive-exec-1.1.0-cdh5.14.0.jar
root@cdh109:/opt/cloudera/parcels/CDH/lib/hive/lib# cd  ../../../jars/
root@cdh109:/opt/cloudera/parcels/CDH/jars# pwd
/opt/cloudera/parcels/CDH/jars
root@cdh109:/opt/cloudera/parcels/CDH/jars# ll |grep  hive-exec-1.1.0-cdh5.14.0.jar
-rw-r--r-- 1 root root  19598157 Jan  6  2018 hive-exec-1.1.0-cdh5.14.0.jar
root@cdh109:/opt/cloudera/parcels/CDH/jars# pwd
/opt/cloudera/parcels/CDH/jars

#解压核实
cd hive-exec-1.1.0-cdh5.14.0/org/apache/hadoop/hive/ql/hooks

rf@xf-macpro ~/D/h/o/a/h/h/q/hooks> ll |grep UpdateInputAccessTimeHook.
-rwxr-xr-x@ 1 rf  staff   946B  1  6  2018 UpdateInputAccessTimeHook$1.class
-rwxr-xr-x@ 1 rf  staff   4.1K  1  6  2018 UpdateInputAccessTimeHook$PreExec.class
-rwxr-xr-x@ 1 rf  staff   645B  1  6  2018 UpdateInputAccessTimeHook.class
```



``` sh
#备份
ansible cdh -m shell   -a "ls -l /opt/cloudera/parcels/CDH/jars/hive-exec-1.1.0-cdh5.14.0.jar  "
ansible cdh -m shell   -a "ls -l  /opt/cloudera/parcels/CDH/jars/hive-exec-1.1.0-cdh5.14.0.jar.old"
```

### 部署

上传`/Users/rf/xuefeng/workspace/hive/packaging/target/apache-hive-1.1.0-cdh5.14.0-jdbc.jar`到集群.

``` sh
# 替换 jar 包
ansible cdh -m copy    -a "src=/tmp/hive-exec-1.1.0-cdh5.14.0.jar  dest=/opt/cloudera/parcels/CDH/jars/hive-exec-1.1.0-cdh5.14.0.jar"
```
### 重启 HIVE 验证生效
