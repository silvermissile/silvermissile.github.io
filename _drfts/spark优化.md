How-to: Tune Your Apache Spark Jobs
# 1
>翻译说明：第二部分主要是spark的配置调优。
#2
## 优化资源分配
在Spark的用户反馈中经常提到一类问题如
>“我有一个500个节点的集群，但是在我运行一个应用时，我看到只有两个Task在执行。求帮助！”。

考虑到Spark资源控制参数的数量，这种问题不是不可能存在的。在这一节里，我们将学习如何压榨出集群的最后一点资源。根据资群资源管理器的不同（Yarn、Mesos和Spark Standalone），相应的建议和配置也有所不同。不过我们将主要集中在Yarn上——这也是Cloudera推荐使用的资源管理器。



参考：别人的翻译：https://www.zhyea.com/2018/07/21/how-to-tune-your-apache-spark-jobs-part-2.html
