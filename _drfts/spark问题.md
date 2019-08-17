``` python
import os
import sys
os.environ['SPARK_HOME'] = "/opt/cloudera/parcels/SPARK2/lib/spark2"
os.environ['PYSPARK_SUBMIT_ARGS'] = "--master yarn pyspark-shell"
sys.path.append(os.path.join(os.environ['SPARK_HOME'], "python"))
sys.path.append(os.path.join(os.environ['SPARK_HOME'], "python/lib/py4j-0.10.6-src.zip"))
from pyspark.sql import SparkSession
conn = SparkSession.builder.master("yarn").appName("test") \
       .config("spark.submit.deployMode", "client") \
       .config("hive.metastore.uris", "thrift://id-cdh-master-86:9083") \
       .config("spark.sql.hive.convertMetastoreParquet", "false") \
       .config("spark.num.executors", "20") \
       .config("spark.executor.cores", "2") \
       .config("spark.executor.memory", "16g") \
       .config("spark.driver.memory", "4g") \
       .config("spark.executor.memoryOverhead", "2048") \
       .config("spark.sql.shuffle.partitions", "100") \
       .enableHiveSupport().getOrCreate()
sql = """ create table temp.wangsongddswe as
  select count(1) as rn from
 snap.akulaku_installmentdb_t_bill t limit 1
"""
hive_df = conn.sql(sql)
```

``` sh
hive@id-cdh-master-89:/usr/local/datadisk$ python  sparktest.py
WARNING: User-defined SPARK_HOME (/opt/cloudera/parcels/SPARK2-2.3.0.cloudera2-1.cdh5.13.3.p0.316101/lib/spark2) overrides detected (/opt/cloudera/parcels/SPARK2/lib/spark2).
WARNING: Running spark-class from user-defined location.
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/SPARK2-2.3.0.cloudera2-1.cdh5.13.3.p0.316101/lib/spark2/jars/phoenix-4.13.2-cdh5.11.2-client.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-5.14.0-1.cdh5.14.0.p0.24/jars/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
Warning: Ignoring non-spark config property: hive.metastore.uris=thrift://id-cdh-master-86:9083
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Traceback (most recent call last):
  File "sparktest.py", line 23, in <module>
    hive_df = conn.sql(sql)
  File "/opt/cloudera/parcels/SPARK2/lib/spark2/python/pyspark/sql/session.py", line 708, in sql
    return DataFrame(self._jsparkSession.sql(sqlQuery), self._wrapped)
  File "/opt/cloudera/parcels/SPARK2/lib/spark2/python/lib/py4j-0.10.6-src.zip/py4j/java_gateway.py", line 1160, in __call__
  File "/opt/cloudera/parcels/SPARK2/lib/spark2/python/pyspark/sql/utils.py", line 69, in deco
    raise AnalysisException(s.split(': ', 1)[1], stackTrace)
pyspark.sql.utils.AnalysisException: u'java.lang.NoSuchMethodError: org.apache.hadoop.hdfs.client.HdfsAdmin.getKeyProvider()Lorg/apache/hadoop/crypto/key/KeyProvider;;'
```



Google的时候发现版本不兼容所致。
我检查了正常集群和出问题集群所有的版本都是一致的。

spark的问题，掩盖了
该引入的jar包的没有引入。

我在检查jar包的时候，发现了多出来的jar包。

本来不需要，剔除掉多的干扰的jar包，就解决问题了。我猜测的原因是：
两个不同的jar包中包含了相同了类。两个类的版本不一致。

这次打脸了。



对比两个集群
hdfs yarn hive spark2的服务配置，没发现问题
hadoop classpath
hadoop version
spark2-submit --version

more  /etc/spark2/spark2-env.sh

实在是找不到问题的出处。

直接对比：/opt/cloudera/parcels/SPARK2/lib/spark2/jars/两个目录的jar包。
诸葛
解决方法：
1.



``` sh
./scproot.sh  host_cdh_plus_client   /home/ubuntu/phoenix-4.13.2-cdh5.11.2-client.jar  /opt/cloudera/parcels/CDH/lib/hbase/lib/
./scproot.sh  host_cdh_plus_client   /home/ubuntu/phoenix-4.13.2-cdh5.11.2-server.jar  /opt/cloudera/parcels/CDH/lib/hbase/lib/
#./scproot.sh  host_cdh_plus_client   /home/ubuntu/phoenix-queryserver-client-4.13.2-cdh5.11.2.jar  /opt/cloudera/parcels/CDH/lib/hbase/lib/
```

清理旧jar
``` sh
./sshroot.sh  host_cdh_plus_client  "rm -f   /opt/cloudera/parcels/SPARK2/lib/spark2/jars/phoenix-*"
./sshroot.sh  host_cdh_plus_client  "rm -f   /opt/cloudera/parcels/CDH/lib/hbase/lib/phoenix-*"
```

分发新版本：
``` sh
./scproot.sh  host_cdh_plus_client   /usr/local/datadisk/apache-phoenix-4.14.0-cdh5.14.2-bin/phoenix-4.14.0-cdh5.14.2-client.jar /opt/cloudera/parcels/SPARK2/lib/spark2/jars/
./scproot.sh  host_cdh_plus_client   /usr/local/datadisk/apache-phoenix-4.14.0-cdh5.14.2-bin/phoenix-4.14.0-cdh5.14.2-server.jar /opt/cloudera/parcels/CDH/lib/hbase/lib/
```
回滚
``` sh
./sshroot.sh  host_cdh_plus_client  "rm -f   /opt/cloudera/parcels/SPARK2/lib/spark2/jars/phoenix-*"
./sshroot.sh  host_cdh_plus_client  "rm -f   /opt/cloudera/parcels/CDH/lib/hbase/lib/phoenix-*"

./scproot.sh  host_cdh_plus_client   /home/ubuntu/phoenix-4.13.2-cdh5.11.2-client.jar  /opt/cloudera/parcels/SPARK2/lib/spark2/jars/
./scproot.sh  host_cdh_plus_client   /home/ubuntu/phoenix-4.13.2-cdh5.11.2-server.jar  /opt/cloudera/parcels/CDH/lib/hbase/
```

1. 配置好phoenix 5.14 三台备用。
2. 停掉写HBASE的任务
3. 删除jspark HBASE classpath中phoenix 的旧jar
4. 分发 新jar
5. 重启HBASE
5. 停掉旧三台
5. 启动新三台phoenix 5.14
