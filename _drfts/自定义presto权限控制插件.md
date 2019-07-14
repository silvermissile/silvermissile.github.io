presto支持以plugin形式来自定义权限控制类。
## 自定义
```
/home/ubuntu/scripts/presto_cluster/presto --server 172.31.17.54:9080 --catalog hive --schema default

AkAccessControl.filterCatalogs  identity:Identity{user='ubuntu'}
AkAccessControl.filterCatalogs  catalogs:[hive]
AkAccessControl.filterTables  identity:Identity{user='ubuntu'}
AkAccessControl.filterTables  catalogName:hive
AkAccessControl.filterTables  schemaNames:[default.apachelog]

AkAccessControl.filterCatalogs  identity:Identity{user='ubuntu'}
AkAccessControl.filterCatalogs  catalogs:[hive]
AkAccessControl.filterTables  identity:Identity{user='ubuntu'}
AkAccessControl.filterTables  catalogName:hive
AkAccessControl.filterTables  schemaNames:[]

AkAccessControl.filterCatalogs  identity:Identity{user='ubuntu'}
AkAccessControl.filterCatalogs  catalogs:[hive]
AkAccessControl.filterTables  identity:Identity{user='ubuntu'}
AkAccessControl.filterTables  catalogName:hive
AkAccessControl.filterTables  schemaNames:[default.dim_role]

AkAccessControl.filterCatalogs  identity:Identity{user='ubuntu'}
AkAccessControl.filterCatalogs  catalogs:[hive]
AkAccessControl.filterTables  identity:Identity{user='ubuntu'}
AkAccessControl.filterTables  catalogName:hive
AkAccessControl.filterTables  schemaNames:[]
```
>这说明判断是否有权限的逻辑都已经写好了，只用判断有没有对应资源的权限就好了

### 开发实现

- 实现  SystemAccessControl  所有方法
3. We just override checkCanSetUser with custom code since we only need to check the principal and user. There's other functions that control different part of Presto.
- 实现 SystemAccessControlFactory  接口 - override getName() to return the name you use to activate the plugin and override create(Map<String, String> config) to return the SystemAccessControl implementation
-  实现 plugin 接口
presto
- Implement Plugin - only need to override getSystemAccessControlFactories() to return the SystemAccessControlFactory() implementation




### 部署
- 把jar包(以及依赖)放入presto server的plugin目录
- 创建 etc/access-control.properties文件写入： ` access-control.name=*name_of_access_control*`
- 重启presto


# 参考
https://prestodb.io/docs/current/develop/spi-overview.html
https://prestodb.io/docs/current/develop/system-access-control.html
