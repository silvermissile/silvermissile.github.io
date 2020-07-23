---
layout:     post
title:      远程调试CDH的 NameNode
subtitle:   演示
date:       2020-07-23
author:     xuefly
catalog: true
tags:
    - 大数据
    -  CDH
    -  调试

---



## 服务端 NameNode
####  修改配置
在`cm`上,点击`NameNode-->confiration-->Java Configuration Options for NameNode`
添加以下内容:
`-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5006 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled `.


![S7Lx8Z](https://gitee.com/xfly/imgbed/raw/master/img/post/S7Lx8Z.png)
>不能使用`*:5006`,直接`address=5006`,否则在启动 NameNodede 的时候报错,在 stdout/stderr 中可以看到:[Java JDB: ERROR: transport error 202: gethostbyname: unknown host](https://stackoverflow.com/questions/50344957/java-jdb-error-transport-error-202-gethostbyname-unknown-host)

####  重启`NameNode`
效果
![uaaGxw](https://gitee.com/xfly/imgbed/raw/master/img/post/uaaGxw.png)
>`Recent Log Entries`提供大量进程启动信息.其中`tout`可以看到当前进程的目录,`/opt/cloudera/cm-5.14.0/run/cloudera-scm-agent/process/5963-hdfs-NAMENODE`,里面有服务进程的配置文件.

## 本地 IDE
获得 CDH 源码,可参考[怎么获得CDH中指定版本组件的源码？](https://silvermissile.github.io/2020/07/11/%E6%80%8E%E4%B9%88%E8%8E%B7%E5%BE%97CDH%E4%B8%AD%E6%8C%87%E5%AE%9A%E7%89%88%E6%9C%AC%E7%BB%84%E4%BB%B6%E7%9A%84%E6%BA%90%E7%A0%81/).


服务端 `NameNode `启动后,在`intellij`中配置远程调试启动即可:
![fbboKu](https://gitee.com/xfly/imgbed/raw/master/img/post/fbboKu.png)
