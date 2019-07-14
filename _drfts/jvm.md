这完全可以搞一个教程出来。
jvm：实操。怎么看，怎么设置。

原理
查看
修改
监控
G1垃圾回收参数设置
https://www.cnblogs.com/smile361/p/7927700.html


演示修改后的效果。

译文-调整G1收集器窍门
  https://segmentfault.com/a/1190000007815623?utm_source=tag-newest



有的是有只管的效果。就跟堆硬件这种这么
Java -XX 所以不用记住。

查看GC日志
图形工具
profile
- 直接网上看下别人怎么配的，
- 配置参数
- 监控工具用起来


HADOOP_OPTS,  we needed to set HADOOP_CLIENT_OPTS fix this error. This was needed because all the hadoop commands run as a client.  HADOOP_OPTS needs to be setup for modifying actual Hadoop run time, and HADOOP_CLIENT_OPTS is needed to be setup for modifying run time for Hadoop command line client.



Garbage collection (memory cleanup) by the JVM can cause HBase clients to experience excessive latency. See Tuning Java Garbage Collection for HBase for a discussion of various garbage collection settings and their impacts on performance.
To tune the garbage collection settings, you pass the relevant parameters to the JVM.

Example configuration values are not recommendations and should not be considered as such. This is not the complete list of configuration options related to garbage collection. See the documentation for your JVM for details on these settings.

-XX:+UseG1GC
Use the 'G1' garbage collection algorithm. Cloudera considers this setting to be experimental.
-XX:MaxGCPauseMillis=value
The garbage collection pause time. Set this to the maximum amount of latency your cluster can tolerate while allowing as much garbage collection as possible.
-XX:+ParallelRefProcEnabled
Enable or disable parallel reference processing by using a + or - symbol before the parameter name.
-XX:-ResizePLAB
Enable or disable resizing of Promotion Local Allocation Buffers (PLABs) by using a + or - symbol before the parameter name.
-XX:ParallelGCThreads=value
The number of parallel garbage collection threads to run concurrently.
-XX:G1NewSizePercent=value
The percent of the heap to be used for garbage collection. If the value is too low, garbage collection is ineffective. If the value is too high, not enough heap is available for other uses by HBase.

https://blog.cloudera.com/blog/2014/12/tuning-java-garbage-collection-for-hbase/


Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
export HADOOP_OPTS="-XX:-UseGCOverheadLimit"
https://stackoverflow.com/questions/1393486/error-java-lang-outofmemoryerror-gc-overhead-limit-exceeded

Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
export HADOOP_CLIENT_OPTS="-XX:-UseGCOverheadLimit -Xmx4096m"
