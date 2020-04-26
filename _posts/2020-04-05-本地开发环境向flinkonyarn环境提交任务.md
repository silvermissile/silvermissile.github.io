--- 
layout:     post
title:      本地开发环境向flinkonyarn环境提交任务
subtitle:   流畅开发的技巧
date:       2020-04-05
author:     xuefly
catalog: true
tags:
    - 大数据
    - flink
    - 开发
---


在本地，向flink on yarn环境提交任务。
- 优点：每次开发完可以直接测试。
- 缺点：每次提交任务要上传jar包，以为网络传输文件，还以为是卡住了。但是不管怎么样，都要向传输。

因为网络问题：以后尽量还是交互式在测试节点上运行。

# 准备环境：
- 添加主机名映射
修改 `/etc/hosts `添加相关的 `hostname `
- 添加kerberos 配置文件
`/etc/krb5.conf`
- 添加Hadoop相关配置
	 - 可以从cm下载
	 - 或者从集群的/etc/目录拷贝。
``` sh
tree yarn-conf
yarn-conf
├── core-site.xml
├── hadoop-env.sh
├── hdfs-site.xml
├── mapred-site.xml
├── ssl-client.xml
├── topology.map
├── topology.py
└── yarn-site.xml
```

设置环境变量
`export HADOOP_CONF_DIR=/Users/rf/workspace/flink-1.10.0/testhkconfig/yarn-conf`
- 下载Hadoop的jar
本地不用安装Hadoop，下载jar到 flink的lib目录：

`wget -p ./lib  https://repo.maven.apache.org/maven2/org/apache/flink/flink-shaded-hadoop-2-uber/2.6.5-10.0/flink-shaded-hadoop-2-uber-2.6.5-10.0.jar`

- 获得凭证
``` sh
klist
Credentials cache: API:4B33830A-581E-4C6E-AC6B-FCC94EBF42AE
        Principal: testuser@TED.COM

  Issued                Expires               Principal
Apr  5 11:33:36 2020  Apr  6 11:33:33 2020  krbtgt/TED.COM@TED.COM
Apr  5 11:45:12 2020  Apr  6 11:33:33 2020  HTTP/cdh230.TED.com@TED.COM
```

# 提交任务
```  sh
~/x/w/flink-1.10.0> ./bin/flink run -m yarn-cluster \
                       ./examples/batch/WordCount.jar \
                       --input hdfs:///tmp/README.txt   --output hdfs:///tmp/flink_people_outlocao999

```
