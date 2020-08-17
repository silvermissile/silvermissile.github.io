---
layout:     post
title:      利用hive的Create_timeAccesstime做数据质量管理(三)
subtitle:   解决新问题`select expression`报错找不到`_dummy_database _dummy_table`
date:       2020-07-24
author:     xuefly
catalog: true
tags:
    - hive
    - 源码

---
在[利用hive的Create_timeAccesstime做数据质量管理(二)](https://silvermissile.github.io/2020/07/11/%E5%88%A9%E7%94%A8hive%E7%9A%84Create_timeAccesstime%E5%81%9A%E6%95%B0%E6%8D%AE%E8%B4%A8%E9%87%8F%E7%AE%A1%E7%90%86(%E4%BA%8C)/)之后,使用过程中又发现一个 bug:`not found _dummy_tabl`
``` sql
SELECT current_date();
Error while processing statement: FAILED: Hive Internal Error: org.apache.hadoop.hive.ql.metadata.InvalidTableException(Table not found _dummy_table)


-- 加个  from tableName 正常
SELECT current_date() FROM test_a;
```
解决过程如下:
## 服务端调式运行
修改 hiveserver 的配置 `
Java Configuration Options for HiveServer2 `添加
`-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=15005`

启动的时候遇到错误:`Unable to determine Hadoop version information. 'hadoop version' returned:`

添加环境变量配置:
`HiveServer2 Environment Advanced Configuration Snippet (Safety Valve)`:`HADOOP_VERSION=2.6.0-cdh5.14.0`

>参考:[Unable to determine Hadoop version information](https://stackoverflow.com/questions/18559555/unable-to-determine-hadoop-version-information)
修改两项配置后如下:
![TEddLH](https://gitee.com/xfly/imgbed/raw/master/img/post/TEddLH.png)

重启 `hiveserver2`,可以看到:
![S850Uk](https://gitee.com/xfly/imgbed/raw/master/img/post/S850Uk.png)



## 本地 IDE 新建远程调试
![9Xp4JQ](https://gitee.com/xfly/imgbed/raw/master/img/post/9Xp4JQ.png)

## 复现
用`beeline`测试运行:`select 1`,可以看到:
![baSBX2](https://gitee.com/xfly/imgbed/raw/master/img/post/baSBX2.png)


>修改源码参考:[HIVE Authorization v2 should not check permission for dummy entity](https://issues.apache.org/jira/browse/HIVE-11498)

## 解决方法
添加判断`re.isDummy()`,如下:
``` java
public static class PreExec implements PreExecute {
//略
//
//
      int lastAccessTime = (int) (System.currentTimeMillis()/1000);

      for(ReadEntity re: inputs) {
        //添加判断
        if (re.isDummy() || re.isPathType()) {
          continue;
        }
        // Set the last query time
        ReadEntity.Type typ = re.getType();
        switch(typ) {
        // It is possible that read and write entities contain a old version
        // of the object, before it was modified by StatsTask.
        // Get the latest versions of the object
        case TABLE: {
          String dbName = re.getTable().getDbName();
          String tblName = re.getTable().getTableName();
          Table t = db.getTable(dbName, tblName);
          t.setLastAccessTime(lastAccessTime);
          db.alterTable(dbName + "." + tblName, t);
          break;
        }
        case PARTITION: {
          String dbName = re.getTable().getDbName();
          String tblName = re.getTable().getTableName();
          Partition p = re.getPartition();
          Table t = db.getTable(dbName, tblName);
          p = db.getPartition(t, p.getSpec(), false);
          p.setLastAccessTime(lastAccessTime);
          db.alterPartition(dbName, tblName, p);
          t.setLastAccessTime(lastAccessTime);
          db.alterTable(dbName + "." + tblName, t);
          break;
        }
        default:
          // ignore dummy inputs
          break;
        }
      }
    }
```
重新编译部署,测试通过.参考:[利用hive的Create_timeAccesstime做数据质量管理(二)](https://silvermissile.github.io/2020/07/11/%E5%88%A9%E7%94%A8hive%E7%9A%84Create_timeAccesstime%E5%81%9A%E6%95%B0%E6%8D%AE%E8%B4%A8%E9%87%8F%E7%AE%A1%E7%90%86(%E4%BA%8C)/)
