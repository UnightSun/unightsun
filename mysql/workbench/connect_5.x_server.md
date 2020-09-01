# 使用MySQL Workbench 8.0连接mysql server 5.x

最近工作中需要维护老版本的mysql服务器，新下载的MySQL Workbench是8.0版本的，连接时候会Access denied。
尝试用MySQL Workbench 8.0自带的cli工具mysql，用命令行进行连接也会报一样的错误：
```
$ mysql --version
mysql  Ver 8.0.xx for macos10.15 on x86_64 (MySQL Community Server - GPL)
$ mysql -hxx.xx.xx.xx -Pxxxx -uxxxxx -pxxxxxxxx
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1045 (28000): Access denied for user 'xxxx'@'xx.xx.xx.xx:xxxx' (using password: Yes)
```
使用老版本的MySQL Workbench或cli客户端就不会出现这种问题
```
$ mysql --version
mysql  Ver 14.14 Distrib 5.6.48, for Linux (x86_64) using  EditLine wrapper
$ mysql -hxx.xx.xx.xx -Pxxxx -uxxxxx -pxxxxxxxx
Warning: Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is xxxxxxxx
Server version: 5.6.xx-xx.x-xxx-xxxxxx Source distribution

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

尝试使用Google解决问题，然而解决过程极其痛苦，各种错误的信息混杂在一起，最终激发了我写这篇文章的动力。

首先新版本连接不上、老版本可以，怀疑是登陆协议更新导致，依稀记得N久以前遇到过类似问题，在Advanced标签卡有个选项可以兼容老版本协议，但未找到。
翻了下[手册](https://dev.mysql.com/doc/workbench/en/wb-mysql-connections-secure-auth.html#idm46091649625968)发现已经在6.3.4之后的版本中移除
>实际上也不是这个mysql_old_password的问题

继续排查发现mysql 8.x升级了默认的auth_plugin，从mysql_native_password改为了caching_sha2_password，以这两个关键字搜索，得到了大量的文章。
但文章基本上都是指引如何升级server、或修改用户当前密码，看起来MySQL Workbench用的人不多？至少用新版本客户端链接老版本服务器的人不多。

在一个介绍升级mysql server的[文章](https://medium.com/@crmcmullen/how-to-run-mysql-8-0-with-native-password-authentication-502de5bac661)中得到启发，修改my.cnf文件：
查找my.cnf位置：
```
$ mysql --help | grep my.cnf                                                                                    
                      order of preference, my.cnf, $MYSQL_TCP_PORT,
/etc/my.cnf /etc/mysql/my.cnf /usr/local/mysql/etc/my.cnf ~/.my.cnf 
```
找到my.cnf的位置，我在这里修改~/.my.cnf文件
```
$ cat ~/.my.cnf                                                              
[mysqld]
default-authentication-plugin=mysql_native_password
[mysql]
default-auth=mysql_native_password
```
将mysql_native_password设置为默认的authentication-plugin，但这个方法只对cli的mysql命令生效，MySQL Workbench依然无法连接。

怀疑MySQL Workbench并未使用自带的命令行工具作为客户端，而是自己实现了连接逻辑，在连接的Advanced标签卡找到Others设置，尝试将上面的my.cnf配置填入
```
default-authentication-plugin=mysql_native_password
default-auth=mysql_native_password
```
***无效***

仔细看输入框的描述：
> Other options for Connector/C++ as option=value pairs, one per line.

是需要按Connector/C++的配置方式，简单了，翻出[文档](https://dev.mysql.com/doc/connector-cpp/1.1/en/connector-cpp-connect-options.html)，搜索auth，找到了：
> **defaultAuth**
> The name of the authentication plugin to use. This option corresponds to the MYSQL_DEFAULT_AUTH option for the mysql_options() C API function. The value is a string.
> This option was added in Connector/C++ 1.1.5.

在Others中填入：
```
defaultAuth=mysql_native_password
```
连接成功！不过出现了新的错误：
> Unknown character set: ''
这次的简单多了，看描述就是编码问题，简单加下配置。


## 最终方案：
在连接配置的 Connection > Advanced > Others 中填入
```
OPT_CHARSET_NAME=utf8
defaultAuth=mysql_native_password
```
解决问题
