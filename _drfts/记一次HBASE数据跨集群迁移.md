大纲
- 背景
- 调研
- 测试
- 实战
  - 数据导出






# 背景
# 调研
各个方案说明

# 测试
值不一致的情况
can 'USERACTIONLOG_TMP', {'LIMIT' => 5}
mac要现在互联网账号的位置输入正确的授权码，


HBASE查看值
用Pythonhbase中查看被编码的中文：
print '\x7F\xFF\xFF\xFF\xFF\x15'.decode('utf-8')

查看被编码的主键的值：
``` bash
hbase(main):005:0* scan 'USERACTIONLOG_NEW', {'LIMIT' => 5}
ROW                                COLUMN+CELL
 \x00\x80\x00\x00\x00\x00\x00\x00e column=0:\x00\x00\x00\x00, timestamp=1504153335068, value=x
 \x80\x00\x01^6\x86u\x1C
 \x00\x80\x00\x00\x00\x00\x00\x00e column=0:\x80\x0B, timestamp=1504153335068, value=00000000-4873-4a79-0000-000000c1924f
 \x80\x00\x01^6\x86u\x1C
 \x00\x80\x00\x00\x00\x00\x00\x00e column=0:\x80\x0C, timestamp=1504153335068, value=\x80\x00\x00\x04
 \x80\x00\x01^6\x86u\x1C
 \x00\x80\x00\x00\x00\x00\x00\x00e column=0:\x80\x0D, timestamp=1504153335068, value=2.0.5
hbase(main):009:0>
PLong::INSTANCE.getCodec.decodeLong("\x7F\xFF\xFF\xFF\xFE\xF8\xD0\x1E".to_java_bytes, 0, SortOrder::ASC)
```

# 实战
  - 数据导出


  解决：大文件从HDFS上s3的问题。
  指定Hadoop运行的缓存目录到数据盘
  大文件问题：怎么拆分
