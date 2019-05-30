---
title: Centos7安装并配置mysql5.6完美教程
date: 2016-12-09 11:26:48
tags: ['Linux', 'MySQL', '安装教程']
categories: ['安装教程']
---

Centos7将默认数据库mysql替换成了Mariadb，对于我们这些还想使用mysql的开发人员来说并不是一个好消息。然而，网上关于Linux安装mysql数据库的教程数不胜数，但是，大多教程都是漏洞百出。今天，为了帮助各位需要的人，从而写下Centos7安装配置mysql5.6的教程。好了，话不多说，上教程：

在接下来的mysql安装过程中，请一定保证自己当前所在目录是正确的! 

e g: [root@localhost ~]# 表示当前目录为~

[root@localhost mysql]# 表示当前目录为mysql
<!-- more -->
# **一、安装MySQL**

1、下载安装包mysql-5.6.34-linux-glibc2.5-x86_64.tar（可前往官网自行下载：http://dev.mysql.com/downloads/mysql/）

2、卸载系统自带的Mariadb


打开Terminal终端：

```
[root@localhost ~]# rpm -qa|grep mariadb  // 查询出来已安装的mariadb
[root@localhost ~]# rpm -e --nodeps 文件名  // 卸载mariadb，文件名为上述命令查询出来的文件
```


3、删除etc目录下的my.cnf

```
[root@localhost ~]# rm /etc/my.cnf
```

4、执行以下命令来创建mysql用户组

```
[root@localhost ~]# groupadd mysql
```

5、执行以下命令来创建一个用户名为mysql的用户并加入mysql用户组

```
[root@localhost ~]# useradd -g mysql mysql
```

6、将下载的压缩包放到 /usr/local/ 目录下（通过mv 要移动的文件  /usr/local/）

7、解压安装包

```
[root@localhost ~]# tar -xvf mysql-5.6.34-linux-glibc2.5-x86_64.tar
```

注：如果压缩包为：mysql-5.6.34-linux-glibc2.5-x86_64.tar.gz，



则解压命令为: tar -zxvf mysql-5.6.34-linux-glibc2.5-x86_64.tar.gz


8、将解压好的文件夹重命名为mysql

```
[root@localhost local]# mv 解压出来的文件夹名 mysql
```

9、在 etc 下新建配置文件my.cnf，并在该文件中添加一下代码：



当然，也有简单方式：直接copy一份my.cnf文件到/etc下，然后再修改即可。

e g：copy一份/usr/local/mysql/support-files/下的my-default.cnf文件到/etc下

```
命令为：[root@localhost support-files]# cp my-default.cnf /etc/my.cnf
```

然后，配置/etc目录下的my.cnf文件

```
[root@localhost support-files]# vim /etc/my.cnf
```

通过vim编辑器编辑my.cnf代码如下：


```
[mysql]
# 设置mysql客户端默认字符集
default-character-set=utf8
socket=/var/lib/mysql/mysql.sock

[mysqld]
skip-name-resolve
#设置3306端口
port = 3306
socket=/var/lib/mysql/mysql.sock
# 设置mysql的安装目录
basedir=/usr/local/mysql
# 设置mysql数据库的数据的存放目录
datadir=/usr/local/mysql/data
# 允许最大连接数
max_connections=200
# 服务端使用的字符集默认为8比特编码的latin1字符集
character-set-server=utf8
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB
lower_case_table_name=1
max_allowed_packet=16M
```

10、进入安装mysql软件目录


```
[root@localhost ~]# cd /usr/local/mysql
[root@localhost mysql]# chown -R mysql:mysql ./       修改当前目录拥有着为mysql用户
[root@localhost mysql]# ./scripts/mysql_install_db --user=mysql --basedir=/usr/local/mysql/ --datadir=/usr/local/mysql/data/ 安装数据库
```

注：若执行以上最后一个命令出现以下问题：

```
FATAL ERROR: please install the following Perl modules before executing
./scripts/mysql_install_db:Data::Dumper
```

解决方法 ：安装autoconf库


命令: `yum -y install autoconf   //此包安装时会安装Data:Dumper模块`
安装完成重新执行上述最后一个命令

重新回到上述第三个命令继续操作：

```
[root@localhost mysql]# chown -R mysql:mysql data 修改当前data目录的拥有者为mysql用户
```

到此数据库安装完毕！


# **二、配置MySQL**

1、授予my.cnf最大权限

```
[root@localhost ~]# chmod 777 /etc/my.cnf
```

设置开机自启动服务控制脚本：

2、复制启动脚本到资源目录

```
[root@localhost mysql]# cp ./support-files/mysql.server /etc/rc.d/init.d/mysqld
```

3、增加mysqld服务控制脚本执行权限

```
[root@localhost mysql]# chmod +x /etc/rc.d/init.d/mysqld
```

4、将mysqld服务加入到系统服务

```
[root@localhost mysql]# chkconfig --add mysqld
```

5、检查mysqld服务是否已经生效

```
[root@localhost mysql]# chkconfig --list mysqld
```

命令输出类似下面的结果:

```
mysqld 0:off 1:off 2:on 3:on 4:on 5:on 6:off
```

表明mysqld服务已经生效，在2、3、4、5运行级别随系统启动而自动启动，以后可以使用service命令控制mysql的启动和停止

命令为:`service mysqld start`和`service mysqld stop`

6、启动mysqld

```
[root@localhost mysql]# service mysqld start
```

7、将mysql的bin目录加入PATH环境变量，编辑 ~/.bash_profile文件

```
[root@localhost mysql]# vim ~/.bash_profile
```

在文件最后添加如下信息:

```
export PATH=$PATH:/usr/local/mysql/bin
```

然后按ESC键

继续 shift键加冒号打出来=>  ：

接下来输入wq回车即可

执行下面的命令是修改的内容立即生效:

```
[root@localhost mysql]# source ~/.bash_profile
```

8、以root账户登录mysql,默认是没有密码的

```
[root@localhost mysql]# mysql -uroot -p
```

要输入密码的时候直接回车即可。

9、设置root账户密码为root（也可以修改成你要的密码）

```
mysql>use mysql;
mysql>update user set password=password('root') where user='root' and host='localhost';
mysql>flush privileges;
```

10、设置远程主机登录，注意下面的your username 和 your password改成你需要设置的用户和密码

```
mysql>GRANT ALL PRIVILEGES ON *.* TO 'your username'@'%' IDENTIFIED BY 'your password' WITH
GRANT OPTION;
```

好了，到此，在Centos 7上安装mysql5.6就完成了。当然，centos 6上安装mysql也可按照如此操作。接下来，快去用Centos 7安装好的mysql去写写sql command吧！
