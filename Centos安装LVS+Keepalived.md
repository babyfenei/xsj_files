---
title: Centos安装LVS+Keepalived
tags: Centos,lvs,Keepalived
grammar_cjkRuby: true
---


主机IP配置如下
```
#LVS-DR-Master    172.16.1.191
#LVS-DR-Backup    172.16.1.192
#LVS-DR-VIP       172.16.1.196
#Web_1-RealServer 172.16.1.193 
#Web_2-RealServer 172.16.1.194
```
在DS1 DS2上执行以下命令
```
yum -y install ipvsadm
wget http://www.keepalived.org/software/keepalived-1.2.19.tar.gz
yum install  -y kernel-devel kernel  openssl-devel popt-devel libnl-devel
uname -a
```
此部软链接必须执行，否则会出现编译出错的情况

```
ln -s /usr/src/kernels/2.6.32-642.6.2.el6.x86_64/ /usr/src/linux
tar xf keepalived-1.2.19.tar.gz 
cd keepalived-1.2.19
./configure --with-kernel-dir=/usr/src/kernels/2.6.32-642.6.2.el6.x86_64/ && make && make install
cp /usr/local/etc/rc.d/init.d/keepalived /etc/init.d/
cp /usr/local/etc/sysconfig/keepalived /etc/sysconfig/
ln -s /usr/local/sbin/keepalived /usr/sbin/
mkdir /etc/keepalived/
```
在DS1上编辑以下文件
```
vi /etc/keepalived/keepalived.conf

global_defs {
#	keepalived有自己的报警机制，但一般不用，所以注释掉，只留下router_id
#	notification_email {
#   acassen@firewall.loc
#   failover@firewall.loc
#   sysadmin@firewall.loc
#   }
#   notification_email_from Alexandre.Cassen@firewall.loc
#   smtp_server localhost
#   smtp_connect_timeout 30
   router_id LVS_DEVEL_1    # 全网唯一标识，不能有重复出现
#   vrrp_skip_check_adv_addr
#   vrrp_strict
#   vrrp_garp_interval 0
#   vrrp_gna_interval 0
}

#组块,同步实例组
vrrp_sync_group LVS {  #设置vrrp组
   group {
       VI_1   #实例名，可以配置多个实例，一个实例一行，一个实例一个虚拟块
   }
}

####虚拟块
vrrp_instance VI_1 {  #跟实例组中的实例名一样
   state MASTER   ##设置lvs的状态，MASTER和BACKUP两种，必须大写
   interface eth0  #对外提供服务的网络接口
   lvs_sync_daemon_interface eth0  #设置lvs监听的接口，类似于心跳检测，（HA中绑定VIP的网口）在DR模式中，interface和lvs_sync_daemon_interface的网卡是一致的
   virtual_router_id 51  #虚拟路由标识，MASTER和BACKUP是一致的，但在整个VRRP中是唯一的
   priority 150  #优先级，数值越大，优先级越高，MASTER要高于BACKUP
   advert_int 1  #同步检查间隔时间，MASTER和BACKUP之间同步检查的时间间隔，单位为秒
   authentication {  #同步验证，验证密码为明文，MASTER和BACKUP之间的密码和验证类型必须一致，此处一般不改
       auth_type PASS
       auth_pass 1111
   }
   virtual_ipaddress { #虚拟地址，可写多个，一个IP一行
       172.16.1.196 #这里设置的IP，一定要和real server的lvs脚本中的VIP一致
       #172.16.1.197
       
   }
}

virtual_server 172.16.1.196 80 { #虚拟服务器配置，此处IP就是之前的VIP,端口就是需要提供负载的端口，比如80,3306等等
   delay_loop 6  #延时等待时间，单位为秒
   lb_algo rr   #HA调度算法，互联网一般就用wlc加权最小连接调度和rr轮询round robin
   lb_kind DR   #HA的负载均衡转发规则，一般用DR方式
   persistence_timeout 0  #会话保持时间，单一链接重连保持时间秒，可以理解为session共享 
   protocol TCP   #转发协议，一般都是采用TCP方式，也可以使用UDP

   real_server 172.16.1.193 80 {  #真实服务器的IP+端口，理解为需要高可用的应用服务，比如80,3306等等
       weight 1  #权重值，值越大，权重越高，可以承担的负载雨大，服务器硬件比较好，可以使用高一点的权重
       TCP_CHECK {  #TCP协议检查
           connect_timeout 3  #连接超时时间，单位秒
           nb_get_retry 3    #检测失败后，重试次数，超出设定值，将后端服务器移除
           delay_before_retry 3 #失败重试时间
       connect_port 80  #需要检测的端口，比如80,3306等等
       }
   }
   real_server 172.16.1.194 80 {
       weight 1
       TCP_CHECK {
           connect_timeout 3
           nb_get_retry 3
           delay_before_retry 3
       connect_port 80
       }
   }
}
重启keepalived服务
/etc/init.d/keepalived restart
```
在DS2上编辑以下文件
```
vi /etc/keepalived/keepalived.conf
global_defs {
   notification_email {
     babyfenei@qq.com                  #报错发邮件地址
   }
   notification_email_from babyfenei@qq.com   #报错发邮件地址
   smtp_server localhost
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}
# VIP1
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    lvs_sync_daemon_inteface eth0    
    virtual_router_id 51               #id
    priority 50                    #优先级
    advert_int 5               
  #  nopreempt                      #不抢占 可以不加
    authentication {
        auth_type PASS
        auth_pass 1111                #验证密码 
    }
    virtual_ipaddress {
        172.16.1.196                 #vip
    }
}
virtual_server 172.16.1.196 80 {           #vip 监听80端口  
    delay_loop 6
    lb_algo wrr
    lb_kind DR                      #dr 轮训模式 
#persistence_timeout 60
    protocol TCP

    real_server 172.16.1.193 80 {         #web1     
        weight 100                  #权重
        TCP_CHECK {
        connect_timeout 10
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
    }
    real_server 172.16.1.194 80 {         #web2
        weight 100
        TCP_CHECK {
        connect_timeout 10
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80

        }
    }
        }
```
重启keepalived服务
```
/etc/init.d/keepalived restart
```
查看网卡IP配置信息
DS1
```
 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:a5:1e:f2 brd ff:ff:ff:ff:ff:ff
    inet 172.16.1.191/23 brd 172.16.1.255 scope global eth0
    inet 172.16.1.196/32 scope global eth0
    inet6 fe80::20c:29ff:fea5:1ef2/64 scope link 
       valid_lft forever preferred_lft forever
```
DS2
```
1 : lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet 172.16.1.196/32 brd 172.16.1.196 scope global lo:0
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:a6:76:58 brd ff:ff:ff:ff:ff:ff
    inet 172.16.1.192/23 brd 172.16.1.255 scope global eth0
    inet6 fe80::20c:29ff:fea6:7658/64 scope link 
       valid_lft forever preferred_lft forever
```