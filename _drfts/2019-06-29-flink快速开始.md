下载地址：
https://flink.apache.org/downloads.html
``` sh
#下载
wget http://mirror.bit.edu.cn/apache/flink/flink-1.8.0/flink-1.8.0-bin-scala_2.12.tgz

#解压

tar xzf flink-*.tgz   # Unpack the downloaded archive
cd flink-1.8.0
#启动本地模式:
./bin/start-cluster.sh  # Start Flink

```

浏览器中打开；http://localhost:8081/
rf@RFdeMacBook-Air ~> nc -l -p 9000
hello
world
flink
flink
flink


rf@RFdeMacBook-Air ~> netstat -na | grep 9000                                                                            1
tcp4       0      0  *.9000                 *.*                    LISTEN


rf@RFdeMacBook-Air ~/m/flink-1.8.0> tail -f log/flink-rf-taskexecutor-0-RFdeMacBook-Air.local.out                      130
hello : 1
world : 1
flink : 2
flink : 1


rf@RFdeMacBook-Air ~/m/flink-1.8.0> bin/flink run examples/streaming/SocketWindowWordCount.jar --port 9000
Starting execution of program


bin/stop-cluster.sh


https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/projectsetup/java_api_quickstart.html#maven
``` sh
f@RFdeMacBook-Air ~/myworkspacee> tree
.
└── quick-flink\
    ├── pom.xml
    ├── quick-flink.iml
    └── src
        └── main
            ├── java
            │   └── com
            │       └── tede
            │           ├── BatchJob.java
            │           └── StreamingJob.java
            └── resources
                └── log4j.properties

7 directories, 5 files

```
