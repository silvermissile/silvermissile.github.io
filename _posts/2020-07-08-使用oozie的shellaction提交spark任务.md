---
layout:     post
title:      使用oozie的shellaction提交spark任务
subtitle:   采坑记录
date:       2020-07-08
author:     xuefly
catalog: true
tags:
    - 大数据
    - oozie
    - shell
    - spark

---

<!-- TOC -->

- [选择脚本](#选择脚本)
- [环境变量](#环境变量)
- [指定 hive 配置](#指定-hive-配置)
- [最终效果](#最终效果)

<!-- /TOC -->

使用的提交 spark 任务的脚本:
``` sh

/opt/cloudera/parcels/SPARK2/bin/spark2-submit \
                --master yarn --deploy-mode client \
                --files /etc/hive/conf/hive-site.xml \
                --num-executors 35 \
                --executor-cores 2\
                --executor-memory 6g \
                --driver-memory 4g \
                --class com.xxx.xxx hdfs:///xxxxx.jar
```
shell 任务会随机分配在一个 yarn 的 nodemanager 上运行,脚本中引用的文件要在各个节点都存在,否则在`file`选项中指定为 HDFS 路径.

## 选择脚本

在选择了要执行的 shell 脚本后,很容易因为编辑的原因在执行的时候报找不到脚本的 IO 问题.
```  sh
java.io.IOException: Cannot run program "testshll.sh" (in directory "/usr/local/datadisk/yarn/nm/usercache/username/appcache/application_1592467334765_238133/container_e97_1592467334765_238133_01_000002"): error=2, No such file or directory
```
>注意:
- 也要在`file`选项里面指定.

``` xml
<exec>/user/username/testshll.sh</exec>
<file>/user/username/testshll.sh#testshll.sh</file>
```
- 脚本文件的内容建议先在终端验证后,再粘贴.
- 脚本文件中不要有`#!/usr/bin/bash`这行.


## 环境变量
在代码中调用 shell 脚本,相比直接在命令行下执行,很多环境变量会丢失.
例如:在创建临时文件`/user/yarn/.staging`会使用 yarn 用户,造成权限问题.

yarn 路径的权限问题.`Oozie shell action - Permission denied: user=xyz, access=EXECUTE, inode="/user/yarn/.staging":yarn:supergroup:drwx------`
解决方法:
- 配置任务时添加环境变量`<env-var>HADOOP_USER_NAME=username</env-var>`
- 或者在脚本中`export HADOOP_USER_NAME=xyz`



>>>>>>> 54c29b7f3086e792feaa8e938681f9bf3b422e80
## 指定 hive 配置
想要在 shell 任务类型中使用 hive。
比如：
- `shell`中使用`hive -e `
- `shell`中提交的`spark`任务中访问了`hive`


这时需要指定 `hive-site.xml`

``` xml
<action name="someaction">
  <shell xmlns="uri:oozie:shell-action:0.2">
    <job-tracker>${jobTracker}</job-tracker>
    <name-node>${nameNode}</name-node>
    <exec>somescript.sh</exec>
    <env-var>SOME_VARIABLE=1</env-var>
    <file>${someactionScriptPathName}#somescript.sh</file>
    <capture-output/>
  </shell>
  <ok to="nextaction"/>
  <error to="Kill"/>
</action>
```


这时需要指定 `hive-site.xml`.否则:
``` sh

aused by: java.sql.SQLException: Unable to open a test connection to the given database. JDBC url = jdbc:derby:;databaseName=/var/lib/hive/metastore/metastore_db;create=true, username = APP. Terminating connection pool (set lazyInit to true if you expect to start your database after your app). Original Exception: ------
java.sql.SQLException: Failed to create database '/var/lib/hive/metastore/metastore_db', see the next exception for details.
```

指定方法:
``` xml
<file>/user/zhaoxf/hive-site.xml#hive-site.xml</file>
```
然后在命令行添加选项:
`--files hive-site.xml \`


当引用了 hive 的 jar 包,还要指定 hive 依赖
``` xml
<configuration>
    <property>
        <name>oozie.action.sharelib.for.shell</name>
        <value>hive</value>
    </property>
</configuration>
```


## 最终效果

![oozie-shell-action-1](https://gitee.com/xfly/imgbed/raw/master/img/post/oozie-shell-action-1.png)
![oozie-shell-action-2](https://gitee.com/xfly/imgbed/raw/master/img/post/oozie-shell-action-2.png)

``` xml
<workflow-app name="AfiIndicator_shell" xmlns="uri:oozie:workflow:0.5">
    <start to="shell-758d"/>
    <kill name="Kill">
        <message>Action failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
    </kill>
    <action name="shell-758d">
        <shell xmlns="uri:oozie:shell-action:0.1">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <configuration>
                <property>
                    <name>oozie.action.sharelib.for.shell</name>
                    <value>hive</value>
                </property>
            </configuration>
            <exec>/user/username/testshll.sh</exec>
              <env-var>HADOOP_USER_NAME=username</env-var>
            <file>/user/username/testshll.sh#testshll.sh</file>
            <file>/user/username/hive-site.xml#hive-site.xml</file>
              <capture-output/>
        </shell>
        <ok to="End"/>
        <error to="Kill"/>
    </action>
    <end name="End"/>
</workflow-app>
```


参考:
[如何使用Hue创建Spark2的Oozie工作流补充](https://cloud.tencent.com/developer/article/1078105)
