# 企业布置zabbix监控

## **一、安装zabbix-server端**

```bash
1.安装Zabbix存储库：
# rpm -Uvh https://repo.zabbix.com/zabbix/5.0/rhel/7/x86_64/zabbix-release-5.0-1.el7.noarch.rpm
# yum clean all

2.安装Zabbix服务器和代理：
# yum install zabbix-server-mysql zabbix-agent

3.安装zabbix前端：
# yum install centos-release-scl

4.编辑配置文件 /etc/yum.repos.d/zabbix.repo
[zabbix-frontend]
name=Zabbix Official Repository frontend - $basearch
baseurl=http://repo.zabbix.com/zabbix/5.0/rhel/7/$basearch/frontend
enabled=1       #此处改为1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591

5.安装Zabbix前端包：
# yum install zabbix-web-mysql-scl zabbix-apache-conf-scl

6.安装mariadb：
# yum -y install mariadb-server
创建数据库：
# mysql
mysql> create database zabbix character set utf8 collate utf8_bin;
mysql> create user zabbix@localhost identified by 'password';
mysql> grant all privileges on zabbix.* to zabbix@localhost;
mysql> flush privileges；  刷新一下
mysql> quit;
导入初始架构和数据，系统将提示您输入新创建的密码：
# zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -ppassword zabbix

7.为Zabbix server配置数据库：
（1）编辑配置文件 /etc/zabbix/zabbix_server.conf
DBPassword=password     #更改文件 

8.为Zabbix前端配置PHP
（1）编辑配置文件： 
# vim /etc/opt/rh/rh-php72/php-fpm.d/zabbix.conf
  php_value[date.timezone] = Asia/Shanghai   # 配置文件解开注释并改时区为亚洲上海

9.启动Zabbix server和agent进程
（1）启动Zabbix server和agent进程，并为它们设置开机自启：
# systemctl restart zabbix-server zabbix-agent httpd rh-php72-php-fpm
# systemctl enable zabbix-server zabbix-agent httpd rh-php72-php-fpm

10.访问ip地址+/zabbix:
http://192.168.246.169/zabbix
访问网站：安装zabbix   设置好并进去用户Admin密码zabbix
```

![](png\1645703603309.png)

## **二、Agent端安装配置**

```bash
首先操作这个
[root@web1 ~]#rpm  -ivh  http://repo.zabbix.com/zabbix/3.4/rhel/7/x86_64/zabbix-release-3.4-2.el7.noarch.rpm
[root@web1 ~]# yum -y install zabbix-agent -y
[root@web1 ~]# vim /etc/zabbix/zabbix_agentd.conf

Server=192.168.122.206         #被动模式 zabbix-server的ip
ServerActive=192.168.122.206    #主动模式 zabbix-server的ip
Hostname=web1                #客户端主机名称
UnsafeUserParameters=1       #是否限制用户自定义 keys 使用特殊字符

启动服务：
[root@web1 ~]# systemctl start zabbix-agent    启动
[root@web1 ~]# systemctl enable zabbix-agent   开机启动
[root@web1 ~]# ss -anlp |grep :10050   查看服务是否启动
```

### **定义自定义key和监控脚本**

```bash
agent端操作
# vim /etc/zabbix/zabbix_agentd.conf

配置文件 添加key和执行的脚本
UserParameter=diskfree,/etc/zabbix/scripts/diskfree.sh

创建脚本存放目录：
# cd /etc/zabbix/
# mkdir scripts
# vim /etc/zabbix/scripts/diskfree.sh
写如下内容：
  echo  `df  / | awk 'NR==2{print$4}'`/1024 |bc
  
安装bc服务，不安装脚本不执行：
# yum -y install bc

给脚本添加执行权限：
# chmod +x  /etc/zabbix/scripts/diskfree.sh

重启服务使配置文件生效：
# systemctl restart zabbix-agent

在**server**端操作利用zabbix_get命令测试key的可用性
安装zabbix—get命令
# yum install zabbix-get -y
# zabbix_get -s 192.168.26.29 -p 10050 -k diskfree  此ip为被监控端agent
            
```

## **三、网页配置zabbix流程**

点击配置-----主机群组  --右上角添加群组名字为zabbix-server

点击主机---添加主机并选择刚创建好的群组zabbix-server

点击主机---点击监控项-添加监控项 键值为刚才弄好的脚本key  填写diskfree （key的名字）

效果图如下：

![](png\1645704457499.png)

**主机效果图：**

![](png\1645704475982.png)

**带参数的自定义key**

```bash
[root@agent ~]# vim /etc/zabbix/zabbix_agentd.conf
# vim /etc/zabbix/zabbix_agentd.conf 
UserParameter=diskfree[*],/etc/zabbix/scripts/diskfree.sh $1
脚本如下：
[root@agent ~]# vim /etc/zabbix/scripts/diskfree.sh 

#!/bin/bash
if [ $1 == "swap" ]
then
echo `free -m |awk 'NR==3{print $4}'`
else
echo `df $1 |awk 'NR==2{print $4}'`/1024 |bc
fi

```

**添加触发器**

**监控项：**

![](png\1645759188732.png)



**-触发器：**

![](png\1645759233846.png)

**添加图形**

![](png\1645760925068.png)

把贺哥发的字符集复制到目录下覆盖掉

```shell
把文件覆盖掉.ttf结尾的字体文件
[root@z-server ~]# mv  /root/贺哥的字符集 /usr/share/zabbix/assets/fonts/graphfont.ttf   
```

###      **配置邮箱报警**

```bash
1.注册邮箱
2.登陆网页邮箱设置客户端授权密码
```

![](png\1.png)

```bash
安装测试mailx软件：
# yum install mailx  -y
[root@zabbix-server	~]# mailx -V
12.5 7/5/10
配置公网邮箱信息:
[root@zabbix-server ~]# vim /etc/mail.rc   追加以下内容
 set from=xiaozhi2104@163.com             （邮箱地址） 
 set smtp=smtp.163.com                    （smtp服务器） 
 set smtp-auth-user=xiaozhi2104@163.com    (用户名) 
 set smtp-auth-password=QXXGSXOFQJSUSVWW  （邮箱密码-授权码） 
 set smtp-auth=login
 
# mailx语法：
方式1：mailx -s "邮件标题" 收件箱Email < 包含正文的文件
方式2：cat 包含正文的文件 | mailx -s "邮件标题" 收件箱Email
方式3：echo "正文内容" | mailx -s "邮件标题" 收件箱Email
方式4：mailx -s "邮件标题" 收件箱Email，回车按CTRL+D发送

手动发送邮件测试：
[root@zabbix-server ~]# mailx -v -s 'hello' '1484070401@qq.com'    手写邮件内容 （回车，然后ctrl+d正常结束)
**如果报错请检查mailx的配置文件是否写错**

```

**创建脚本**

```shell
查看zabbix-server配置文件脚本存放路径
[root@z-server ~]# vim /etc/zabbix/zabbix_server.conf
AlertScriptsPath=/usr/lib/zabbix/alertscripts    脚本存放路径

在目录下创建脚本：
[root@z-server ~]# vim /usr/lib/zabbix/alertscripts/sendmail.sh

#!/bin/sh 
echo "$3" | sed s/'\r'//g | mailx -s "$2" $1
修改权限，属组属主：
[root@z-server ~]# chmod +x sendmail.sh && chown zabbix.zabbix sendmail.sh
```

**创建报警媒介**

```shell
======================================================================================
管理--报警媒介--右上角创建媒介
```

![](png\2.png)

![](png\3.png)

```shell
名称：邮件报警                   //名称任意
类型：脚本                      //选择脚本模块
脚本名称：sendmail.sh           //zabbix-server脚本名字  **不要写错**
脚本参数：                      //一定要写，否则可能发送不成功
{ALERT.SENDTO}                //照填，收件人变量
{ALERT.SUBJECT}               //照填，邮件主题变量，变量值来源于‘动作’中的‘默认接收人’
{ALERT.MESSAGE}               //照填，邮件正文变量，变量值来源于‘动作’中的‘默认信息’
```

![](png\4.png)

```shell
#消息模板
告警主机：{HOSTNAME1} 
告警时间：{EVENT.DATE} {EVENT.TIME}
告警等级：{TRIGGER.SEVERITY} 
告警信息：{TRIGGER.NAME}
告警项目：{TRIGGER.KEY1} 
问题详情：{ITEM.NAME}：{ITEM.VALUE}
当前状态：{TRIGGER.STATUS}：{ITEM.VALUE1} 
事件ID：{EVENT.ID}
```

**创建用户**

```bash
=================================================================================
管理---用户----Admin
```

![](png\5.png)

**创建动作**

```shell
======================================================================================
配置--动作--创建动作   #**动作绑定触发器**
```

![](png\6.png)

```shell
========================================================================================
**添加操作**
```

![](png\7.png)

```shell
#解释
发送间隔：1分
步骤：发送10次发送到：admin用户
仅使用：sendmail方式发送
```

