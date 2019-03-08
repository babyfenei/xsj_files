---
title: ZABBIX 微信、邮件告警并附带流量图的图文过程[转载]
tags: zabbix,wechat
grammar_cjkRuby: true
---
[原文链接](http://blog.sina.com.cn/s/blog_13f206dbe0102wip7.html)

此篇文章为本人博客的处女作，为什么开始写博客？不为别的，只为在我闲暇之时可以常过来看看，让一些必要的知识和见解不会被时间的潮流冲散。如诸位同学对以下内容有何见解之处，还望百忙之中回复一下，谢谢。

在做以下操作前，请先做到如下几点，如何做？baidu一下多了去了。

 1. 搭建好zabbix（我用的是zabbix3.0） 
 2. 搭建好邮件服务（我用的是postfix加mailx）
 3. 申请微信公众号（http://www.cnblogs.com/hanyifeng/p/5368102.html，其中的python脚本用我以下的shell脚本）
 4. 安装谷歌浏览器（好用）

相信看到这里的同学都对zabbix有了一定的了解了，让其告警其实一点也不复杂，复杂就复杂在附加流量图的问题上，那我可不可以把zabbix中的流量图当做一个图片来处理呢，大家都知道发邮件的时候是可以添加附件的，而微信的信息有纯文本的，有纯图片的，还有图文的，这样不就简单了吗？接下来看我如何提取zabbix流量图。

1、打开谷歌浏览器并打开你需要告警时附加流量图的监控项（这里有一点需要申明，你所需要告警附带图的监控项每一个都需要你手动获取图片id）

2、空白处点击鼠标右键→检查，如下图：
![enter description here](http://s10.sinaimg.cn/mw690/005Ql4Quzy7596IvEFz79&690)

3、展开标签查找“graphid=66608”，注意：每一个流量图所对应的id都不一样。
![enter description here](http://s8.sinaimg.cn/large/005Ql4Quzy7596WTNlRf7&690)
4、将graphid记录到一个文件中，注意空格和次序
![ZABBIX <wbr>微信、邮件告警并附带流量图的图文过程](http://s3.sinaimg.cn/bmiddle/005Ql4Quzy7598OQ4sG42&690)

5、添加触发器，这里需注意上图和下图中“-”符号之前的名称必须一致（该符号意义颇大，稍后脚本里见）
![ZABBIX <wbr>微信、邮件告警并附带流量图的图文过程](http://s10.sinaimg.cn/mw690/005Ql4Quzy7599laXG1b9&690)

6、添加动作，注意默认接收人里面中间的空格必须去掉，还有默认信息里面的内容尽量别有空格，有一个空格就多一个位置变量参数，不知道你们的是不是，反正我的是这样的，你们可以多试试，有什么发现还望回复下我，谢谢。其他zabbix设置和正常设置没区别，这里就不多做阐述了。
![ZABBIX <wbr>微信、邮件告警并附带流量图的图文过程](http://s1.sinaimg.cn/mw690/005Ql4Quzy759a8rwBOd0&690)


7、创建报警媒介
![ZABBIX <wbr>微信、邮件告警并附带流量图的图文过程](http://s7.sinaimg.cn/mw690/005Ql4Quzy759baVFQ276&690)



8、修改下图所示的配置，没有"/home/zabbix/shell"目录的自己mkdir一个，当然这不是强制性的，不过最好定义在zabbix用户的家目录下，毕竟涉及到权限的问题是很麻烦的，完事后重启zabbix_server
![ZABBIX <wbr>微信、邮件告警并附带流量图的图文过程](http://s9.sinaimg.cn/mw690/005Ql4Quzy759bnSwFGb8&690)

9、创建微信告警脚本：vim /home/zabbix/shell/weixin.sh（以下到“10、”之前，全部是一个脚本文件的内容）
```
#!/bin/bash

CropID='xxxxxxxxxxxx'    ## 查看你自己微信公众号的id
Secret='xxxxxxxxxxxx'    ## 查看你自己微信公众号的id

GURL="https://qyapi.weixin.qq.com/cgi-bin/gettoken?corpid=$CropID&corpsecret=$Secret"
Gtoken=`/usr/bin/curl -s -G $GURL | awk -F '\"' '{print $4}'`

URL="http://localhost/zabbix"
PURL="https://qyapi.weixin.qq.com/cgi-bin/message/send?access_token=$Gtoken"
PURL_image="https://qyapi.weixin.qq.com/cgi-bin/media/upload?access_token=$Gtoken&type=image"
cookie=/tmp/curl.zabbix.cookie
AppID=1              ## 微信公众号应用ID，注意看你自己的
UserID=$1           ## 微信接收者（刚开始弄得时候）
PartyID=1
Username=admin  ## zabbix登陆用户名
Password=password   ##zabbix登陆密码
Kehu=`echo $2 |awk -F "-" '{print $1}' |awk -F ":" '{print $2}'`
Bottle="$2-$3-$4-$5-$6-$7-$8-$9"

function weixinimg() {
	local Stime=`date -d -1hour +%Y%m%d%H%M%S`
	local Graphid_1=`echo $2 |awk -F "-" '{print $1}' |awk -F ":" '{print $2}'`
	local Graphid_2=`grep $Graphid_1 /tmp/data.db | awk '{print $1}'`
	source /etc/profile
	/usr/bin/curl -c $cookie -b $cookie  -d "request=&name=$Username&password=$Password&autologin=1&enter=Sign+in" $URL/index.php
	/usr/bin/curl -b $cookie -d "graphid=$Graphid_2&period=3600&stime=$Stime&width=800" $URL/chart2.php > /tmp/weixin-img.png
	local Media=`/usr/bin/curl -F "data=@/tmp/weixin-img.png" $PURL_image`
	local Media_id=`echo $Media | awk -F "\"" '{print $8}'`
	/usr/bin/curl --data-ascii "$(mpnews $1 $2 $3)" $PURL
}

function weixintext() {
	/usr/bin/curl --data-ascii "$(text $1 $2 $3)" $PURL
}
 
function text() {
	printf '{'
	printf '"touser": '"\"$UserID\","
	printf '"toparty": '"\"$PartyID\","
	printf '"msgtype": "text",'
	printf '"agentid": '"\"$AppID\","
	printf '"text": {'
	printf '"content": '"\"$Bottle\","
	printf '},'
	printf '"safe":"0"'
	printf '}'
}

function mpnews() {
	printf '{'
	printf '"touser":'"\"$UserID\","
	printf '"toparty":'"\"$PartyID\","
	printf '"msgtype":"mpnews",'
	printf '"agentid":'"\"$AppID\","
	printf '"mpnews":{'
	printf '"articles":['
	printf '{'
	printf '"title":'"\"$2\","
	printf '"thumb_media_id":'"\"$Media_id\","
	printf '"content":'"\"$3\","
	printf '"show_cover_pic":"1",'
	printf '"digest":"点击图片查看更多详情",'
	printf '},'
	printf '"safe":"0"'
	printf '}'
}

grep "$Kehu" /tmp/list.db
if [ $? -eq 0 ] ;then
    weixinimg $1 $2 $3
else
    weixintext $1 $2 $Bottle
    exit
fi
```
10、创建邮件告警脚本：vim /home/zabbix/shell/email.sh
```
#!/bin/bash
Kehu=`echo $2 |awk -F "-" '{print $1}' |awk -F ":" '{print $2}'`
Bottle="$3-$4-$5-$6-$7-$8-$9"
cookie=/tmp/curl.zabbix.cookie
URL=http://localhost/zabbix
Stime=`date -d -1hour +%Y%m%d%H%M%S`
Graphid_1=`echo $2 |awk -F "-" '{print $1}' |awk -F ":" '{print $2}'`
Graphid_2=`grep $Graphid_1 /tmp/data.db | awk '{print $1}'`
Name=admin                        ## zabbix登陆用户名
Password=password                ## zabbix登陆密码
grep "$Kehu" /tmp/list.db
if [ $? -eq 0 ] ;then
    source /etc/profile
    /usr/bin/curl -c $cookie -b $cookie  -d "request=&name=$Name&password=$Password&autologin=1&enter=Sign+in" $URL/index.php
    /usr/bin/curl -b $cookie -d "graphid=$Graphid_2&period=3600&stime=$Stime&width=800" $URL/chart2.php > /tmp/email-img.png
    echo "$3" | mail -s $2 -a /tmp/email-img.png $1
else
    echo "$Bottle" | mail -s "$2" "$1"
    exit
fi
```
11、测试
```
/home/zabbix/shell/weixin.sh zabbix 标题 正文

/home/zabbix/shell/email.sh 你的邮箱 标题 正文
```
12、下图为以上脚本触发后的效果

![ZABBIX <wbr>微信、邮件告警并附带流量图的图文过程](http://s3.sinaimg.cn/mw690/005Ql4Quzy759BtFeHE72&690)

很多时候，有些东西是不能通用的，就像以上我所展示的脚本，说实话，在此之前，我看过很多博客、论坛等各大网站，吸收了很多有用的东西，才总结出来了一个适合我自己用的东西，如果要做微信告警，强烈建议阅读微信开发者文档，很有用。

或许有人注意到了，我除了脚本文件在zabbix家目录下之外，其他文件都在/tmp下，那是因为我把目录改为/home/zabbix/shell的时候，切换zabbix用户手动执行完全没有问题，但是当zabbix触发器触发后，它就是不执行那个脚本，web页面全部显示消息已送达，无奈之下只能改为/tmp，如其他同僚知道原因所在，还望告知，谢谢。
