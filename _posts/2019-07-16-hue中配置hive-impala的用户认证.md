---
layout:     post
title:      hue中配置hive-impala的用户认证
subtitle:   使用Hive或Impala和Impersonation进行LDAP或PAM传递身份验证
date:       2019-07-16
author:     xuefly
catalog: true
tags:
    - 大数据
    - hive
    - hue
    - impala
---

>来自：[LDAP or PAM pass-through authentication with Hive or Impala and Impersonation](http://gethue.com/ldap-or-pam-pass-through-authentication-with-hive-or-impala/)  翻译 by xuefly

## 冒充 Impersonation
Hue是在浏览器上登录hue的用户，与hue上一系列Hadoop服务之间的中介服务。 因此，Hue被其他大数据服务视为单个“hue”用户。使用Impersonation是为了让hue上的登录用户，仍然应用真实登录用户的权限，而不是所有登录者都是hue的权限。 例如，当用户'bob'提交查询时，Hue还会发送该用户的用户名，而HiveServer2将使用'bob'而不是'hue'作为查询的所有者。
![](http://gethue.com/wp-content/uploads/2015/09/hue-auth-client.png)

## 身份验证
Hue支持多种方式与其他服务器进行身份验证：常见的Kerberos和LDAP，以及PAM。

hue可以区分用于Hive或Impala的身份验证（hive 、impala曾经只能一起用同一配置）。 例如，您可以将Hue配置为一边使用LDAP连接HiveServer2，另一边使用Kerberos 连接 Impala。

用于pam和LDAP的用户名和密码，可在主配置部分（[desktop]）中进行配置，也可以分别在每个相应的应用程序（hive 、impala）中进行覆盖。

为了提供更好的安全性，还可以配置密码文件的路径，（而不是将密码直接写入hue.ini中）。
### 举例
#### 示例 1
如果未设置明文密码，则将使用该密码文件。例如，以下是演示如何在文件中为所有应用配置“hue”用户和密码。
``` sh
[desktop]
auth_username=hue
#auth_password=
auth_password_script=/path/to/ldap_password
```
#### 示例 2
如果Hue需要使用单独的用户名和密码对HiveServer2进行身份验证：[beeswax]优先级高于[desktop]，将覆盖[desktop]
``` sh
[beeswax]
auth_username=hue_hive
auth_password=hue_hive_pwd
#auth_password_script=
```
#### 示例 3
如果hive使用LDAP身份验证，但impala没有，我们不在[impala]中指定任何内容来让impala继承[desktop]配置。
``` ini
[desktop]
auth_username=hue
#auth_password=
#auth_password_script=

[beeswax]
auth_username=hue_hive
auth_password=hue_hive_pwd

[impala]
#auth_username=
#auth_password=hue_impala
#auth_password_script=/
```

### 注意
>不设置任何密码将使LDAP / PAM身份验证处于非激活状态。

>还支持Hue和其他Hadoop服务之间的SSL加密


>在CM的“HiveServer2高级配置代码段（安全阀）for hive-site.xml”（即：Hue Service Advanced Configuration Snippet (Safety Valve) for hue_safety_valve.ini）中添加的配置，将覆盖hue的hive-site.xml中相应配置项。重启hive hue 后，配置生效。
