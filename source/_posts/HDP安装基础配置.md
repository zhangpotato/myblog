---
title: HDP安装——基础环境配置
date: 2019-01-29 13:39:00
author: Potato
top: true # 如果top值为true，则会是首页推荐文章
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
categories: HDP
tags:
  - HDP安装
  - BigData
---

## 一、下载安装包

**注：安装centos7时，安装英文版，避免环境安装时报错**

| **ambari**    | **<http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.3.0/ambari-2.7.3.0-centos7.tar.gz>** |
| :------------ | ------------------------------------------------------------ |
| **HDP**       | **<http://public-repo-1.hortonworks.com/HDP/centos7/3.x/updates/3.1.0.0/HDP-3.1.0.0-centos7-rpm.tar.gz>** |
| **HDP-UTILS** | **<http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/centos7/HDP-UTILS-1.1.0.22-centos7.tar.gz>** |
| **HDP-GPL**   | **<http://public-repo-1.hortonworks.com/HDP-GPL/centos7/3.x/updates/3.1.0.0/HDP-GPL-3.1.0.0-centos7-gpl.tar.gz>** |

------

## 二、系统环境配置

**所有操作都是root用户**

### 	1、修改主机名

```shell
#在所有机器上
vim /etc/hostname
```

```shell
master01 #主机名称
```

### 	2、修改网络

```shell
#在所有机器上
vim /etc/sysconfig/network-scripts/ifcfg-enp0s31f6 #ifcfg-enp0s31f6 为网卡名称
```

```shell
BOOTPROTO="static"          # 使用静态IP地址，默认为dhcp
IPADDR="192.168.0.80"   	# 设置的静态IP地址
NETMASK="255.255.255.0"     # 子网掩码
GATEWAY="192.168.0.1"       # 网关地址
DNS1="8.8.8.8"       	    # DNS服务器
ONBOOT="yes"         	    #是否开机启用
```

### 	3、配置hosts文件

```shell
#在所有机器上
vi /etc/hosts
```

```shell
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.0.80 master01
192.168.0.81 node01
192.168.0.82 node02
```

### 	4、关闭防火墙

```shell
#在所有机器上
systemctl disable firewalld
systemctl stop firewalld
```

### 	5、设置SSH免密登陆

```shell
#在所有机器上
ssh-keygen 
三下回车
ssh-copy-id master01
ssh-copy-id node01
ssh-copy-id node02
```

### 	6、检查DNS和NSCD

```shell
#在所有机器上
setenforce 0	#临时设置 重启失效
```

```shell
#在所有机器上
vim /etc/selinux/config	#永久有效 需重启机器
SELINUX=disabled
```

```shell
#在所有机器上
umask 0022	#临时设置 重启失效
echo umask 0022 >> /etc/profile	#永久有效 重启生效
umask #检查当前机器的umask码
```

### 	7、修改文件打开限制

```shell
#在所有机器上
vim /etc/security/limits.conf
#在文件最后
# End of file
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
```

### 	8、同步时钟

```shell
#在所有机器上
ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
yum -y install ntp
#在主服务器上
vim /etc/ntp.conf
#在文件中配置
restrict default nomodify notrap nopeer noquery
server 127.127.1.0
fudge 127.127.1.0 stratum 10
#在子节点中
crontab -e
0-59/10 * * * * /usr/sbin/ntpdate master01
#在所有节点启动服务
systemctl start ntpd
systemctl enable ntpd
```

### 	9、安装JDK

#### 		Ⅰ、**先安装openjdk**

​			安装openjdk解决包依赖的问题

```shell
#在所有机器上
yum -y install java
```

#### 		Ⅱ、卸载openjdk

​		卸载jdk包，但是保留依赖包

```shell
#在所有机器上
rpm -qa|grep java
yum remove java-1.8.0-openjdk-1.8.0.161-0.b14.el7_4.x86_64
```

#### 		Ⅲ、安装开源JDK

```shell
#在所有机器上
#下载jdk
wget https://download.oracle.com/otn-pub/java/jdk/8u201-b09/42970487e3af4f5aa5bca3f542482c60/jdk-8u201-linux-x64.rpm?AuthParam=1548732196_c3d078fb7f78308027532d04a26b4612
#重命名
mv jdk-8u201-linux-x64.rpm?AuthParam=1548732196_c3d078fb7f78308027532d04a26b4612 jdk-8u201-linux-x64.rpm
#安装jdk
rpm -ivh jdk-8u201-linux-x64.rpm
#检查jdk版本
java -version

java version "1.8.0_201"
Java(TM) SE Runtime Environment (build 1.8.0_201-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.201-b09, mixed mode)
#修改配置文件
export JAVA_HOME=/usr/java/jdk1.8.0_201-amd64
export CLASSPATH=.:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar 
export PATH=$JAVA_HOME/bin:$PATH
```

### 	10、安装MySQL

**默认是PostgreSQL，但是在生产环境中不建议使用，所以安装mysql或其他，mysql安装在主节点上，如有需要可以安装在其他节点上**

#### 		Ⅰ、卸载原来的MySQL

```shell
rpm -qa|grep -i mysql #查看是否安装mysql
yum remove … 	#卸载mysql所有内容
find / -name mysql	#查找mysql所有文件位置
rm -rf /var/lib/mysql…	#删除mysql相关文件
rm -rf /var/log/mysql.log #删除该日志文件
```

#### 		Ⅱ、下载安装MySQL5.6

```shell
wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
rpm -ivh mysql-community-release-el7-5.noarch.rpm
yum install mysql-server	#安装mysql
systemctl start mysqld	#启动mysql
systemctl enable mysqld	#开机启动
```

#### 		Ⅲ、修改默认密码

```shell
mysql -u root -p (无密码)
use mysql
select host,user,password from user;
delete from user where user='';
update user set host='%' where user='localhost';	#(如果无user=localhost 可不执行这条命令)
update user set password=PASSWORD('123456') where host='%'	#(这条命令是根据上条执行)
update user set password=PASSWORD('123456') where user='root' and host='localhost';
update user set password=PASSWORD('123456') where user='root' and host='master';	#(可选操作，master01为主机名)
grant all privileges on *.* to 'root'@'%' identified by '123456' with grant option;	#(远程连接数据库权限设置)
flush privileges; 	#要刷新权限
```

## 三、修改yum源，实现离线安装

### 1、安装httpd

```shell
#在master01上执行
yum -y install httpd
service httpd restart
systemctl enable httpd
```

### 2、修改安装文件位置

```shell
#在master01上执行
cd /var/www/html/
mkdir ambari
#解压文件
tar -xvzf ambari-2.7.3.0-centos7.tar.gz
tar -xvzf HDP-3.1.0.0-centos7-rpm.tar.gz
tar -xvzf HDP-GPL-3.1.0.0-centos7-gpl.tar.gz
tar -xvzf HDP-UTILS-1.1.0.22-centos7.tar.gz
#打开http://192.168.0.80/ambari 查看是否有结果
```

### 3、制作本地源

```shell
#在master01上执行
#在/var/www/html/ambari目录下创建yum源
yum install yum-utils createrepo yum-plugin-priorities -y
createrepo  ./
```

```shell
#在master01上执行
#修改ambari安装包源配置
cd /var/www/html/ambari/ambari/centos7/2.7.3.0-139
vim ambari.repo
#VERSION_NUMBER=2.7.3.0-139
[ambari-2.7.3.0]
#json.url = http://public-repo-1.hortonworks.com/HDP/hdp_urlinfo.json
name=ambari Version - ambari-2.7.3.0
baseurl=http://192.168.0.80/ambari/ambari/centos7/2.7.3.0-139
gpgcheck=1
gpgkey=http://192.168.0.80/ambari/ambari/centos7/2.7.3.0-139/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
```

```shell
#在master01上执行
#修改HDP配置文件
cd /var/www/html/ambari/HDP/centos7/3.1.0.0-78
vim hdp.repo
#VERSION_NUMBER=3.1.0.0-78
[HDP-3.1.0.0]
name=HDP Version - HDP-3.1.0.0
baseurl=http://192.168.0.80/ambari/HDP/centos7/3.1.0.0-78
gpgcheck=1
gpgkey=http://192.168.0.80/ambari/HDP/centos7/3.1.0.0-78/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1

[HDP-UTILS-1.1.0.22]
name=HDP-UTILS Version - HDP-UTILS-1.1.0.22
baseurl=http://192.168.0.80/ambari/HDP-UTILS/centos7/1.1.0.22
gpgcheck=1
gpgkey=http://192.168.0.80/ambari/HDP-UTILS/centos7/1.1.0.22/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
```

### 4、将repo文件辅助到YUM库

```shell
#在master01上执行
cp /var/www/html/ambari/ambari/centos7/2.7.3.0-139/ambari.repo /etc/yum.repos.d/
cp /var/www/html/ambari/HDP/centos7/3.1.0.0-78/hdp.repo /etc/yum.repos.d/
```

### 5、将文件拷贝到子节点

```shell
#在master01上执行
scp /etc/yum.repos.d/ambari.repo node01:/etc/yum.repos.d/
scp /etc/yum.repos.d/ambari.repo node02:/etc/yum.repos.d/
scp /etc/yum.repos.d/hdp.repo node01:/etc/yum.repos.d/
scp /etc/yum.repos.d/hdp.repo node02:/etc/yum.repos.d/
```

### 6、重新设置YUM缓存

```shell
#在所有机器上执行
yum clean all
yum makecache
```

## 四、安装ambari-server

### 1、在MySQL中创建用户和数据库

```shell
#根据安装软件需要自己选择要创建的数据库和用户
CREATE DATABASE ambari;  
use ambari;  
CREATE USER 'ambari'@'%' IDENTIFIED BY '123456';  
GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'%';  
CREATE USER 'ambari'@'localhost' IDENTIFIED BY '123456';  
GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'localhost';  
CREATE USER 'ambari'@'master01' IDENTIFIED BY '123456';  
GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'master01';  
FLUSH PRIVILEGES;  

CREATE DATABASE hive;  
use hive;  
CREATE USER 'hive'@'%' IDENTIFIED BY 'hive';  
GRANT ALL PRIVILEGES ON *.* TO 'hive'@'%';  
CREATE USER 'hive'@'localhost' IDENTIFIED BY 'hive';  
GRANT ALL PRIVILEGES ON *.* TO 'hive'@'localhost';  
CREATE USER 'hive'@'master01' IDENTIFIED BY 'hive';  
GRANT ALL PRIVILEGES ON *.* TO 'hive'@'master01';  
FLUSH PRIVILEGES;  

CREATE DATABASE oozie;  
use oozie;  
CREATE USER 'oozie'@'%' IDENTIFIED BY 'oozie';  
GRANT ALL PRIVILEGES ON *.* TO 'oozie'@'%';  
CREATE USER 'oozie'@'localhost' IDENTIFIED BY 'oozie';  
GRANT ALL PRIVILEGES ON *.* TO 'oozie'@'localhost';  
CREATE USER 'oozie'@'master01' IDENTIFIED BY 'oozie';  
GRANT ALL PRIVILEGES ON *.* TO 'oozie'@'master01';  
FLUSH PRIVILEGES;  
```

### 2、将jar包放入环境

```shell
#下载mysql-connector-java-5.1.47.tar.gz
cd /data
wget https://cdn.mysql.com//Downloads/Connector-J/mysql-connector-java-5.1.47.tar.gz
tar -xvzf mysql-connector-java-5.1.47.tar.gz
cp /data/mysql-connector-java-5.1.47/mysql-connector-java-5.1.47* /usr/share/java/
cd /usr/share/java/
mv mysql-connector-java-5.1.47.jar mysql-connector-java.jar
cp /usr/share/java/mysql-connector-java.jar /var/lib/ambari-server/resources/mysql-jdbc-driver.jar
```

### 3、初始化ambari-server并启动

```shell
#初始化
ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar
#配置ambari
ambari-server setup
#所有选项选择y即可，当遇到如下选项时
#配置JDK环境
Checking JDK...
Do you want to change Oracle JDK [y/n] (n)? y
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Oracle JDK 1.7 + Java Cryptography Extension (JCE) Policy Files 7
[3] Custom JDK
==============================================================================
Enter choice (1): 3
如果上面选择3自定义JDK,则需要设置JAVA_HOME。输入：/usr/java/jdk1.8.0_201-amd64
#配置数据库
Configuring database...
Enter advanced database configuration [y/n] (n)? y
Configuring database...
==============================================================================
Choose one of the following options:
[1] - PostgreSQL (Embedded)
[2] - Oracle
[3] - MySQL
[4] - PostgreSQL
[5] - Microsoft SQL Server (Tech Preview)
[6] - SQL Anywhere
==============================================================================
Enter choice (3): 3
#设置数据库的具体配置信息，根据实际情况输入，如果和括号内相同，则可以直接回车。如果想重命名，就输入。
Hostname (localhost):
Port (3306):
Database name (ambari):
Username (ambari):
Enter Database Password (bigdata):	123456	#创建用户时设置的密码
Re-Enter password: 123456
```

检查mysql中是否有新建的表

```shell
mysql -u root -p 123456
use ambari;
show tables;
#如果表为空的话
cd /var/lib/ambari-server/resources	#sql脚本位置
myslq -u ambari -p 123456
use ambari;
source Ambari-DDL-MySQL-CREATE.sql;
```

### 4、错误处理

如果出现错误，查看server日志，根据报错进行修改

```shell
cat var/log/ambari-server/ambari-server.log
```

也可以删除数据库重新配置

```shell
ambari-server stop
mysql -u root -p 123456
drop database ambari;
drop database hive;
drop database oozie;
#重新创建数据库
```

### 5、清除已经安装的包

```shell
#在ambari-server启动的状态下执行命令，清除已经安装的软件包
python /usr/lib/python2.6/site-packages/ambari_agent/HostCleanup.py --silent
```



### 6、安装向导本地源库地址

```shell
#HDP
http://192.168.0.80/ambari/HDP/centos7/3.1.0.0-78/
#HDP-UTILS
http://192.168.0.80/ambari/HDP-UTILS/centos7/1.1.0.22/
#HDP-GPL
http://192.168.0.80/ambari/HDP-GPL/centos7/3.1.0.0-78/
```

