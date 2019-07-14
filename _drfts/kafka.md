kafkacat：https://www.cnblogs.com/xdao/p/10674848.html
``` sh
apt install kafkacat
kafkacat -C  -b 172.22.1.147  -t  riskAkuloanActionlog  -p 0 -o 100 -e
```


为了充分使用集群性能，临时任务只要很快跑完是不会限制资源，长时间的任务要报备

不然其他重要常规任务会失败


不知道的长时间跑又占用大量资源的发现了会被人工kill掉
