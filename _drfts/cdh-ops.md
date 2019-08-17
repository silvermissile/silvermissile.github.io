### 删表数据仍在
现象：
- 用户在hue使用impala，删表，再次创建的时候会，提示目录已经已存在。因为数据仍在
- 用户使用spark overwrite写数据到表的时候，没有权限。

原因：
/user/hive/warehouse目录默认权限上加了粘滞位。
每个用户只能删除owner是自己的路径。

之所以会这样，是因为迁移数据的时候，路径的权限没有传递过来，只过来了数据。

解决方法：`hdfs dfs -chmod -R -t  /user/hive/warehouse/tmp.db/`
去掉粘滞位后，放宽了用户权限，这样存在隐患，需要进一步限制。

参考：[官方文档](https://www.cloudera.com/documentation/enterprise/5-12-x/topics/cdh_ig_filesystem_perm.html)
