在vi 中执行命令
经常遇到一个情况，修改了文件之后，想马上执行验证效果。
1):!command


不退出vim，并执行shell命令command，将命令输出显示在vim的命令区域，不会改变当前编辑的文件的内容

例如

:!ls -l

特别的可以运行:!bash来启动一个bash shell并执行命令，不需要退出vim

2):r !command


将shell命令command的结果插入到当前行的下一行


例如


:r !date，读取时间并插入到当前行的下一行。

spark.sql("create table recommend.testtest " )
