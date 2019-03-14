---
title: centos6 安装zabbix4.2
tags: zabbix,监控,运维
grammar_cjkRuby: true
---


######  1.关闭selinux
``` 
sed -i "s#SELINUX=enforcing#SELINUX=disabled#g" /etc/selinux/config  #重启生效
setenforce 0   #临时关闭
```
######  2. zabbix需要mysql5.6以上版本，删除旧的版本
```
rpm -ivh http://dev.mysql.com/get/mysql-community-release-el6-5.noarch.rpm
yum -y install mysql-server
yum list installed | grep mysql
```
###### 3.修改mysql配置文件/etc/my.cnf,在[mysqld]中添加innodb_file_per_table=1
```
vim /etc/my.cnf
innodb_file_per_table=1
/etc/init.d/mysqld start
```
###### 4.登陆数据库
```
[root@localhost /]# mysql
#创建zabbix库，指定字符集
mysql> CREATE DATABASE zabbix CHARACTER SET utf8 COLLATE utf8_bin;
Query OK, 1 row affected (0.06 sec)
#创建zabbix用户密码：zabbix  授权拥有访问zabbix库的所有权限
mysql> GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost IDENTIFIED BY 'zabbix'; 
Query OK, 0 rows affected (0.02 sec)

#查看数据库是否创建成功
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| zabbix             |
+--------------------+
4 rows in set (0.03 sec)
```
###### 5.Zabbix 3需要PHP是至少5.4或更高版本。我们的CentOS 6.5库跟php 5.3.3因此我们需要安装一个新的
```
rpm -ivh http://repo.webtatic.com/yum/el6/latest.rpm
```
* 5.1安装所需要的包

```
yum -y install httpd php56w php56w-gd php56w-mysql php56w-bcmath php56w-mbstring php56w-xml php56w-ldap
```
######  6.修改php配置
```
[root@localhost /]# vim /etc/php.ini 
post_max_size = 16M
max_execution_time = 300
max_input_time = 300
date.timezone = Asia/Shanghai
always_populate_raw_post_data = -1
```
###### 7. 修改apache配置
```
vim /etc/httpd/conf/httpd.conf
ServerName 127.0.0.1
DirectoryIndex index.html index.html.var index.php

启动httpd服务
/etc/init.d/httpd start
```
###### 8.创建zabbix用户
```
groupadd zabbix
useradd -g zabbix zabbix
```
######  9.依赖包安装
```
yum -y install gcc mysql-community-devel libxml2-devel unixODBC-devel net-snmp-devel libcurl-devel libssh2-devel OpenIPMI-devel openssl-devel openldap-devel libevent-devel pcre-devel wget
```
##### 10.下载zabbix安装包、解压、导入sql
```
mkdir -p /usr/src/zabbix && cd /usr/src/zabbix
# 官方源码包下载地址：
wget https://excellmedia.dl.sourceforge.net/project/zabbix/ZABBIX%20Latest%20Development/4.2.0beta2/zabbix-4.2.0beta2.tar.gz
tar -zxvf zabbix-4.2.0beta2.tar.gz
# 导入zabbix数据库
cd /usr/src/zabbix/zabbix-4.2.0beta2/database/mysql/
mysql zabbix < schema.sql 
mysql zabbix < images.sql 
mysql zabbix < data.sql 
```
##### 11.安装zabbix
```
cd /usr/src/zabbix/zabbix-4.2.0beta2
# 编译
./configure --enable-server --enable-agent --with-mysql --enable-ipv6 --with-net-snmp --with-libcurl --with-libxml2 --with-unixodbc --with-ssh2 --with-openipmi --with-openssl --prefix=/usr/local/zabbix
# 安装
make install
echo $?
```
##### 12 .修改zabbix_server的配置
```
[root@localhost etc]# vim /usr/local/zabbix/etc/zabbix_server.conf
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix
````
##### 13.创建一个新的web前端文件
```
mkdir /var/www/html/zabbix
# 移动源到刚才创建目录下
cd /usr/src/zabbix/zabbix-4.2.0beta2/frontends/php/
cp -rf  *  /var/www/html/zabbix/
chown -R apache:apache /var/www/html/zabbix
chmod +x /var/www/html/zabbix/conf/
cp /usr/src/zabbix/zabbix-4.2.0beta2/misc/init.d/fedora/core/zabbix_server /etc/init.d/zabbix_server
 cp /usr/src/zabbix/zabbix-4.2.0beta2/misc/init.d/fedora/core/zabbix_agentd /etc/init.d/zabbix_agentd
 ```
 ##### 14.添加Zabbix服务器和Zabbix代理服务
```
chkconfig --add /etc/init.d/zabbix_server
chkconfig --add /etc/init.d/zabbix_agentd
chkconfig httpd on
chkconfig mysqld on
chkconfig zabbix_server on
chkconfig zabbix_agentd on
```
##### 15.启动zabbix_server
```
[root@localhost php]# /etc/init.d/zabbix_server start    #报错
Starting zabbix_server:  /etc/init.d/functions: line 546: /usr/local/sbin/zabbix_server: No such file or directory
                                                           [FAILED]
 [root@localhost php]# vim /etc/init.d/zabbix_server
BASEDIR=/usr/local/zabbix     #更改下路径
[root@localhost php]# /etc/init.d/zabbix_server start
Starting zabbix_server:                                    [  OK  ]
 [root@localhost php]# vim /etc/init.d/zabbix_agentd
BASEDIR=/usr/local/zabbix    #更改下路径
[root@localhost php]# /etc/init.d/zabbix_agentd start
Starting zabbix_agentd:                                    [  OK  ]
```
 ##### 16.报错提示：
 ```
 启动服务时提示：error while loading shared libraries: libpcre.so.1: cannot open shared object file: No such file or directory ”

从错误看出是缺少lib文件导致，进一步查看下

ldd $(which /usr/local/zabbix/sbin/zabbix_server)

        　linux-vdso.so.1 =>  (0x00007fff2cbff000)

         libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f6a5758a000)

         libcrypt.so.1 => /lib64/libcrypt.so.1 (0x00007f6a57353000)

         libpcre.so.1 => not found  #看到这里没有找到这个文件

         libz.so.1 => /lib64/libz.so.1 (0x00007f6a56f1f000)

         libc.so.6 => /lib64/libc.so.6 (0x00007f6a56b8b000)

         /lib64/ld-linux-x86-64.so.2 (0x00007f6a577af000)

         libfreebl3.so => /lib64/libfreebl3.so (0x00007f6a56928000)

         libdl.so.2 => /lib64/libdl.so.2 (0x00007f6a56724000)   
 
先查找下系统里面有没有这个文件：

[root@slave_217 ~]# find / -name 'libpcre.so.1' 
/usr/lib64/libpcre.so.1
/usr/local/zabbix/lib/libpcre.so.1
/usr/local/lib/libpcre.so.1
/usr/lib/libpcre.so.1
/lib64/libpcre.so.1
/root/django/pcre-8.35/.libs/libpcre.so.1

上面看到系统有这个文件，然后把这个文件建立一个软连接，然后就可以正常启动服务
cd /lib64/
ln -s /usr/local/lib/libpcre.so.1 /lib64/ 


