# 背景


需要在 landset中显示日志.类似 hue 这样.在提交的时候,运行的时候,目的就是一站式的 IDE.



目前 yarn 的日志查看方式.
![7Iiqnv](https://gitee.com/xfly/imgbed/raw/master/img/post/7Iiqnv.png)

![fYf0h3](https://gitee.com/xfly/imgbed/raw/master/img/post/fYf0h3.png)
## 调研
一个另外的方案
- 在容器内通过各个服务的 log配置,把日志输出到文件
- 文件的位置挂载在一个统一的 PV 上.
- 让 firebeat去动态解析这个日志.
- 最终日志到 ES kibana上.

`log-pilot`这个工具需要在每个 node 上部署客户端.
而且我们的 pod 有的还是在 ECI 上,还是我们自己的方案能够落地.

现在需要用 presto 的这个方案.


我是为了让 spark 的日志可以让 firebeat 采集.
- 它能动态发现一个目录下的日志文件吗?






## 方案
![k8slog](https://gitee.com/xfly/imgbed/raw/master/img/post/k8slog.png)
## 过程
### 挂载持久层存储
nfs关在 spark flink 容器内.

`/tmp/log/{spark.app.name}.log `
`SparkApplication` 的 HOSTNAME环境变量自带 `appname.driver`

`/tmp/log/{flink.app.name}.log `
### 日志落盘
- 怎么把 appname hostname 打到不同配置文件里面.
- 怎么区分不同时间 APP
- 怎么区分不同名称的 APP

#### spark 任务日志落到文件
spark 默认使用 log4j
#### flink 任务日志落到文件
flink 默认支持 common log4j

切换到 logback 的方法.




- 最终到 es 里面取查询.
### 客户端开发获取日志的界面
- 这要跟 landset配合.

有的是初试,有的是上限.

Spark任务是否支持在 ECI 上,伸缩 executor,如果开启了动态分配资源的话?
OSS 是不支持 ECI 吗?
