杀端口重启的方法已经过时。
``` sh
ubuntu@cdh-master4:~$ ll   /tmp/ | grep hbase
drwxr-xr-x    3 akulaku   appuser         4096 Apr  9 02:09 hbase-akulaku/
drwxr-xr-x    3 hbase     hbase           4096 Apr  1 09:46 hbase-hbase/
drwxrwxr-x    3 hdfs      hdfs            4096 Jul  1 15:29 hbase-hdfs/
drwxrwxr-x    3 hive      hive            4096 Jun 25 06:07 hbase-hive/
drwxrwxr-x    3 liangchao liangchao       4096 Jun 20 06:20 hbase-liangchao/
drwxrwxr-x    3 ubuntu    ubuntu          4096 Apr  3 01:45 hbase-ubuntu/
drwxr-xr-x    2 hbase     hbase           4096 Jul  3 07:18 hsperfdata_hbase/

hbase@cdh-master4:/home/ubuntu$ jps
82907 ThriftServer
82905 HMaster
98693 Jps
hbase@cdh-master4:/home/ubuntu$ ll /tmp/hsperfdata_hbase/
total 2052
drwxr-xr-x    2 hbase hbase    4096 Jul  3 08:15 ./
drwxrwxrwt 4851 root  root  2027520 Jul  3 08:16 ../
-rw-------    1 hbase hbase   32768 Jul  3 08:16 82905
-rw-------    1 hbase hbase   32768 Jul  3 08:16 82907


sudo rm -f -r /tmp/hbase-hbase ;
sudo rm -f -r /tmp/hsperfdata_hbase  ;


lsof -i:60030 ;
netstat -ano | grep 60030 ;
```
