来源：[Add documentation for db backed ResourceGroupConfigurationManager](https://github.com/prestodb/presto/commit/e9b820e0ad748e348a0feaad9799f1e19c50c4c8?diff=split)

http://149.129.233.220:9980/ui/

```  properties
resource-groups.configuration-manager=db
resource-groups.config-db-url=jdbc:mysql://rm-d9jz6061q58mo1u65.mysql.ap-southeast-5.rds.aliyuncs.com:3306/presto_resource_group?user=presto_resource_group&password=presto_resource_group_password
```


``` sql
create database presto_resource_group DEFAULT CHARACTER SET  utf8mb4;

grant all on presto_resource_group.* TO 'presto_resource_group'@'%' IDENTIFIED BY 'presto_resource_group_password';
```

如果数据库里面没有表，会自动创建。
``` sql
insert into resource_groups (resource_group_id, name, soft_memory_limit, max_queued,hard_concurrency_limit )
values (1, 'global', '95%', 20, 20);


insert into resource_groups (resource_group_id, name, soft_memory_limit, max_queued,hard_concurrency_limit ,parent)
values (2, '${USER}', '80%', 10, 10,1);

insert into selectors (resource_group_id, user_regex)
values (2, 'luoyh|zhaoxf', );
```

``` sql
nsert into resource_groups_global_properties (name, value) values ('cpu_quota_period', '1h');
```

``` sql
insert into resource_groups (resource_group_id, name, soft_memory_limit, max_queued, max_running,``
``soft_cpu_limit, hard_cpu_limit) values (1, 'global', '95%', 100, 100, '5m', '10m');



| resource_groups | CREATE TABLE `resource_groups` (
  `resource_group_id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(250) NOT NULL,
  `soft_memory_limit` varchar(128) NOT NULL,
  `max_queued` int(11) NOT NULL,
  `soft_concurrency_limit` int(11) DEFAULT NULL,
  `hard_concurrency_limit` int(11) NOT NULL,
  `scheduling_policy` varchar(128) DEFAULT NULL,
  `scheduling_weight` int(11) DEFAULT NULL,
  `jmx_export` tinyint(1) DEFAULT NULL,
  `soft_cpu_limit` varchar(128) DEFAULT NULL,
  `hard_cpu_limit` varchar(128) DEFAULT NULL,
  `parent` bigint(20) DEFAULT NULL,
  `environment` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`resource_group_id`),
  KEY `parent` (`parent`),
  CONSTRAINT `resource_groups_ibfk_1` FOREIGN KEY (`parent`) REFERENCES `resource_groups` (`resource_group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 |



| selectors | CREATE TABLE `selectors` (
  `resource_group_id` bigint(20) NOT NULL,
  `priority` bigint(20) NOT NULL,
  `user_regex` varchar(512) DEFAULT NULL,
  `source_regex` varchar(512) DEFAULT NULL,
  `query_type` varchar(512) DEFAULT NULL,
  `client_tags` varchar(512) DEFAULT NULL,
  `selector_resource_estimate` varchar(1024) DEFAULT NULL,
  KEY `resource_group_id` (`resource_group_id`),
  CONSTRAINT `selectors_ibfk_1` FOREIGN KEY (`resource_group_id`) REFERENCES `resource_groups` (`resource_group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 |
```

不管怎么配这个selector都没效果。
``` bash
ubuntu@id-cdh-manager:~$ presto  --server id-data-presto-51:9980 --user guojx
presto:default> show catalogs;
Query 20190830_073819_00005_jr6tu failed: No selectors are configured
```

现在能想到大的方法就是看源码里面，这个selector是怎么查到的。

发现创建了个库

Google 和文档解决问题，已经进入了死胡同。
要从源码解决问题。
- 两种搜索。double shift没找到的东西，用find in path  找到了。
- 代码导航
  - 返回 cmd+[

如果用json文件是可以的，但是每次需要重启。

发现不是通过SQL来查的，而是用过代码，这样的话就需要把测试环境搭建起来，
所以现在我要搞的是
- 在测试环境安装MySQL，我的最差的笔记本。还是直接用香港测试服务器的吧
- 搭建presto服务
- 这次要视频，写文档了。
