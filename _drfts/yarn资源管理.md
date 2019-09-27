``` sh
..............................
..............................
2019-08-22 21:12:51,471 INFO  feature.flinktask.CouponAntiStat                              - Start onTimer......
2019-08-22 21:12:51,472 INFO  feature.flinktask.CouponAntiStat                              - COUPON KEY:7049-2,Result:2
2019-08-22 21:12:57,882 INFO  org.apache.flink.yarn.YarnTaskExecutorRunner                  - RECEIVED SIGNAL 15: SIGTERM. Shutting down as requested.
2019-08-22 21:12:57,883 INFO  org.apache.flink.runtime.blob.PermanentBlobCache              - Shutting down BLOB cache
2019-08-22 21:12:57,883 INFO  org.apache.flink.runtime.state.TaskExecutorLocalStateStoresManager  - Shutting down TaskExecutorLocalStateStoresManager.
2019-08-22 21:12:57,883 INFO  org.apache.flink.runtime.blob.TransientBlobCache              - Shutting down BLOB cache
2019-08-22 21:12:57,893 INFO  org.apache.flink.runtime.io.disk.iomanager.IOManager          - I/O manager removed spill file directory /usr/local/datadisk/yarn/nm/usercache/feature_flink/appcache/application_1566179077250_11463/flink-io-bc40c31f-0645-4756-9014-22e254ce0
116
```


能用多少，还剩多少，每个队列有个这样的标志。


When enabled, if a pool's minimum share is not met for some period of time, the Fair Scheduler preempts applications in other pools. Preemption guarantees that production applications are not starved while also allowing the cluster to be used for experimental and research applications. To minimize wasted computation, the Fair Scheduler preempts the most recently launched applications.

应用： 申请的小于
最小资源数：
启用后，如果在一段时间内未满足池的最小共享，则Fair Scheduler将抢占其他池中的应用程序。 Preemption保证生产应用程序不会缺资源，同时也允许集群用于实验和研究应用程序。为了最大限度地减少计算浪费，Fair Scheduler抢占了最近提交的应用程序。

应该
share那个呢？
会冲突：没有高于steady share 就不会被抢。

最大值最小值因为是绝对值，也要设置成内存和CPU的比例一致。
而百分比是相对值的。mem cpu占总体的比例应该一致。

抢占是按照比例来的。

>If fairSharePreemptionTimeout is not set for a given queue or one of its ancestor queues, and the defaultFairSharePreemptionTimeout is not set, pre-emption by this queue will never occur, even if pre-emption is enabled.

按照这个说法，抢占从来就没有发生。
- flink自己的kill。
- 容器使用了过多的资源，

所以不能开启。
