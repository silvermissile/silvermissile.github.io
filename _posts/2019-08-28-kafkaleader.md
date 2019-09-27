---
layout:     post
title:      kafka集群没有主节点
subtitle:   不能查看元数据，不能生产
date:       2019-08-28
author:     xuefly
catalog: true
tags:
    - 大数据
    - kafkacat
    - kafka

---

### 问题
现象1
``` sh
kafkacat -L -b hk-test-cdh109:9092
% ERROR: Failed to acquire metadata: Local: Broker transport failure
```
>一般是网络问题，确实主机、端口是通的话，还出现这个问题。

现象2
``` sh
kafka-console-producer  --broker-list hk-test-cdh109:9092 --topic syslog


clients.NetworkClient: [Producer clientId=console-producer] Error while fetching metadata with correlation id 55 : {syslog=LEADER_NOT_AVAILABLE}
clients.NetworkClient: [Producer clientId=console-producer] Error while fetching metadata with correlation id 56 : {syslog=LEADER_NOT_AVAILABLE}
```
对比正常的 kafka 集群，发现有问题的kafka服务没有带`Active`标志的实例：`Kafka Broker (Active Controller)`

### 解决方法
强制重新选举
``` sh
zookeeper-client
ls  /
get /controller


delete /controller
```

### 验证
``` sh
kafkacat  -L -b hk-test-cdh109:9092
Metadata for all topics (from broker 193: hk-test-cdh109:9092/193):
 4 brokers:
  broker 1 at 192.168.1.105:9092
  broker 193 at hk-test-cdh109:9092 (controller)
  broker 195 at hk-test-cdh110:9092
  broker 194 at hk-test-cdh111:9092
 2 topics:
  topic "my-topic" with 13 partitions:
    partition 0, leader 194, replicas: 194, isrs: 194
    partition 1, leader 195, replicas: 195, isrs: 195
    partition 2, leader 1, replicas: 1, isrs: 1
    partition 3, leader 193, replicas: 193, isrs: 193
    partition 4, leader 194, replicas: 194, isrs: 194
```
参考[How many Kafka controllers are there in a cluster and what is the purpose of a controller?](https://stackoverflow.com/questions/49525141/how-many-kafka-controllers-are-there-in-a-cluster-and-what-is-the-purpose-of-a-c)
