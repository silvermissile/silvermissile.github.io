

git课程筹备：
举个例子：
有的人这样操作：
https://community.hortonworks.com/questions/139411/timeout-job-pending-on-yarn.html
有的人直接在开源
https://issues.apache.org/jira/browse/YARN-3813


windows
linux/macOS

1.检查SSH是否失效

       在git命令行中进行git操作的时候，发现原来设置过的SSH key已经失效；登陆到github网站上查看，图标呈现灰色；好了，发现问题所在；

2.设置用户名和邮箱

      在git命令行中对git进行全局设置， git config --global user.name "用户名"， git config --global user.email "邮箱地址"；

3.生成SSH key

      在git命令行中，输入命令： cd ~/.ssh，来检测是否生成过key,没有生成过key，会有相关信息提示；然后输入命令： ssh-keygen -t rsa -C “邮箱地址”，按下回车键（一路回车）；然后根据返回的信息，找到.ssh目录下的两个文件；

4.在github上添加SSH key

      在github上点击“setting”，找到添加SSH key的菜单，然后新增SSH key；把文件id_rsa.pub  里面的内容全部复制到 key编辑框中，保存完毕；

5.检查SSH key是否有效

      在git命令行输入：ssh  -T git@github.com；这里会要求你输入SSH key密码，如果刚才生成SSH key时未输入密码，密码就为空；然后看到信息：

ERROR: Hi 用户名! You’ve successfully authenticated；说明配置成功；

6.再次查看github密钥

      登陆到github上查看刚刚输入的SSH key，现在图标的颜色变为绿色，说明密钥配置有效；现在可以在git命令行上进行git操作了；



注意（来自知乎：https://www.zhihu.com/question/21402411）：

如果  github提示Permission denied (publickey)
首先，清除所有的key-pair
ssh-add -D
rm -r ~/.ssh
删除你在github中的public-key

重新生成ssh密钥对
ssh-keygen -t rsa -C "xxx@xxx.com"
chmod 0700 ~/.ssh
chmod 0600 ~/.ssh/id_rsa*

接下来正常操作
在github上添加公钥public-key:
1、首先在你的终端运行 xclip -sel c ~/.ssh/id_rsa.pub将公钥内容复制到剪切板
2、在github上添加公钥时，直接复制即可
3、保存

测试：
在终端 ssh -T git@github.com
---------------------
作者：GoodMaxi
来源：CSDN
原文：https://blog.csdn.net/sun_73hd/article/details/79419561
版权声明：本文为博主原创文章，转载请附上博文链接！


测试:
``` sh
rf@RFdeMacBook-Air ~/.ssh> ssh -T git@github.com
The authenticity of host 'github.com (192.30.253.113)' can't be established.
RSA key fingerprint is SHA256:nThbg6kXUpJWGl7E1IGOCspRomTxdCARLviKw6E5SY8.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'github.com,192.30.253.113' (RSA) to the list of known hosts.
Hi silvermissile! You've successfully authenticated, but GitHub does not provide shell access.
```

upic
我的仓库名写错了。
