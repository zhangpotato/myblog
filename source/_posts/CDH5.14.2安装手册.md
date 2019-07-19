---
 title: CDH5.14.2安装手册
date: 2019-06-04 11:37:00
author: Potato
top: true # 如果top值为true，则会是首页推荐文章
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
categories: CDH
tags:
  - CDH安装
  - BigData
---

## 一、基础环境配置

1、修改主机名

```sh
#操作系统为CentOS7
#192.168.0.220节点上（root用户）
vim /etc/hostname
host0
#192.168.0.221节点上
vim /etc/hostname
host1
#192.168.0.222节点上
vim /etc/hostname
host2
```

### 2、配置host文件

```sh
#为所有机器配置(root用户)
vim /etc/hosts
192.168.0.220 host0
192.168.0.221 host1
192.168.0.222 host2
```

### 3、修改IP地址

```sh
#使用root用户进行操作
vim /etc/sysconfig/network-scripts/ifcfg-ens33 --最后为显卡名称
#修改配置文件
BOOTPROTO="static"
ONBOOT="yes"
IPADDR="192.168.0.220"
NETMASK="255.255.255.0"
GATEWAY="192.168.0.1"
DNS1="8.8.8.8"
#重启服务
systemctl restart network
#同理在host1、host2节点上修改IP地址并重启服务
```

### 4、配置免密

```sh
#在所有机器上操作（root用户）
ssh-keygen (三下回车)
ssh-copy-id host0
ssh-copy-id host1
ssh-copy-id host2
```

### 5、配置JAVA环境

```sh
#卸载系统原有Java环境（所有机器root用户下）
rpm -qa |grep java
yum remove java-*
#下载JAVA环境包(可能需要去官网注册下载tar包，然后上传到服务器)
wget https://download.oracle.com/otn/java/jdk/8u211-b12/478a62b7d4e34b78b671c754eaaf38ab/jdk-8u211-linux-x64.tar.gz?AuthParam=1559552580_152e8c88de42f475d6bdbeb45e5e1fdb

mv jdk-8u211-linux-x64.tar.gz\?AuthParam\=1559552580_152e8c88de42f475d6bdbeb45e5e1fdb jdk-8u211-linux-x64.tar.gz
mkdir /usr/java
tar -xvzf jdk-8u211-linux-x64.tar.gz  -C /usr/
mv jdk1.8.0_211 java
#配置环境变量
vim /etc/profile

export JAVA_HOME=/usr/java
export CLASSPATH=.:${JAVA_HOME}/jre/lib/rt.jar:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar
export PATH=$PATH:${JAVA_HOME}/bin

#生效配置
source /etc/profile

java -version

java version "1.8.0_211"
Java(TM) SE Runtime Environment (build 1.8.0_211-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.211-b12, mixed mode)
```

### 6、关闭防火墙

```sh
	#所有机器
	systemctl stop firewalld
	systemctl disable firewalld
```

### 7、关闭SELINUX

```sh
	#临时修改（所有机器）
	setenforce 0
	#永久修改
	vi /etc/selinux/config
	修改: SELINUX=disabled
```

### 8、时间同步

```sh
#在所有机器上 
yum -y install ntp 
#在主服务器上 
vim /etc/ntp.conf 
#在文件中配置 
restrict default nomodify notrap nopeer noquery 
server 127.127.1.0 fudge 127.127.1.0 stratum 10 
#在子节点中 crontab -e 0-59/10 * * * * /usr/sbin/ntpdate host0 
#在所有节点启动服务 systemctl start ntpd systemctl enable ntpd
```

### 8、关于系统参数配置

```sh
#在所有节点上
vim /etc/sysctl.conf

kernel.sysrq = 0 
kernel.core_uses_pid = 1 
kernel.msgmnb = 65536 
kernel.msgmax = 65536 
kernel.shmmax = 68719476736 
kernel.shmall = 4294967296 
##打开文件数参数(20*1024*1024) 
fs.file-max= 20971520 
##WEB Server参数 
net.ipv4.tcp_tw_reuse=1 
net.ipv4.tcp_tw_recycle=1 
net.ipv4.tcp_fin_timeout=30 
net.ipv4.tcp_keepalive_time=1200 
net.ipv4.ip_local_port_range = 1024 65535 
net.ipv4.tcp_rmem=4096 87380 8388608 
net.ipv4.tcp_wmem=4096 87380 8388608 
net.ipv4.tcp_max_syn_backlog=8192 
net.ipv4.tcp_max_tw_buckets = 5000 
##TCP补充参数 
net.ipv4.ip_forward = 0 
net.ipv4.conf.default.rp_filter = 1 
net.ipv4.conf.default.accept_source_route = 0 
net.ipv4.tcp_syncookies = 1 
net.ipv4.tcp_sack = 1 
net.ipv4.tcp_window_scaling = 1 
net.core.wmem_default = 8388608 
net.core.rmem_default = 8388608 
net.core.rmem_max = 16777216 
net.core.wmem_max = 16777216 
net.core.netdev_max_backlog = 262144 
net.ipv4.tcp_max_orphans = 3276800 
net.ipv4.tcp_timestamps = 0 
net.ipv4.tcp_synack_retries = 1 
net.ipv4.tcp_syn_retries = 1 
net.ipv4.tcp_mem = 94500000 915000000 927000000 
##禁用ipv6 
net.ipv6.conf.all.disable_ipv6 =1 
net.ipv6.conf.default.disable_ipv6 =1 
##swap使用率优化 
vm.swappiness=0 

#生效配置
sysctl -p
```

### 9、修改文件句柄数

```sh
#所有机器
vim /etc/security/limits.conf

*               soft    nofile          65535
*               hard    nofile          1029345
*               soft    nproc           unlimited
*               hard    nproc           unlimited
*               soft    memlock         unlimited
*               hard    memlock         unlimited
```

## 二、安装MySQL

### 1、卸载MySQL

```sh
#在所有机器上卸载原有MySQL
rpm -qa|grep -i mysql
yum remove … 卸载mysql
find / -name mysql
rm -rf /var/lib/mysql…删除mysql相关文件
rm -rf /var/log/mysql.log 删除该日志文件
```

### 2、安装MySQL

```sh
	#在主节点上运行，如host0
	wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
	rpm -ivh mysql-community-release-el7-5.noarch.rpm
	yum install mysql-server
	systemctl start mysqld
	systemctl enable mysqld
```

### 3、修改密码、设置权限

```sql
	mysql -u root -p (无密码)
	use mysql
	select host,user,password from user;
	delete from user where user='';
	update user set host='%' where user='localhost';(如果无user=localhost 可不执行这条命令)
	update user set password=PASSWORD('123456') where host='%'(这条命令是根据上条执行)
	update user set password=PASSWORD('123456') where user='root' and host='localhost';
	update user set password=PASSWORD('123456') where user='root' and host='host0';(可选操作，host0为主机名)
	 grant all privileges on *.* to 'root'@'%' identified by '123456' with grant option;(远程连接数据库权限设置)
	flush privileges; //要刷新权限
	例如：授权test用户拥有testDB数据库的所有权限：
	grant all privileges on testDB.* to “test”@”localhost” identified by “1234”; 
```

### 4、下载MySQl驱动

```sh
#以下操作在所有节点上
wget https://cdn.mysql.com//Downloads/Connector-J/mysql-connector-java-5.1.47.tar.gz
#将JAR包放在/usr/java/  下 没有文件夹则创建

#创建CDH复制jar包目录
mkdir /usr/share/java
cp mysql-connector-java-5.1.47* /usr/share/java/
#切换目录
cd /usr/share/java/
#修改jar包名称 mysql-connector-java.jar是CDH可识别的名称 避免在安装Oozie服务报错
mv mysql-connector-java-5.1.47-bin.jar mysql-connector-java.jar
```

### 5、创建表

```sql
#插图
--create database scm default character set utf8 default collate utf8_general_ci;
grant all on scm.* to 'scm'@'%' identified by '123456';

create database amon default character set utf8 default collate utf8_general_ci;
grant all on amon.* to 'amon'@'%' identified by '123456';

create database rman default character set utf8 default collate utf8_general_ci;
grant all on rman.* to 'rman'@'%' identified by '123456';

create database hue default character set utf8 default collate utf8_general_ci;
grant all on hue.* to 'hue'@'%' identified by '123456';

create database metastore default character set utf8 default collate utf8_general_ci;
grant all on metastore.* to 'hive'@'%' identified by '123456';

create database nav default character set utf8 default collate utf8_general_ci;
grant all on nav.* to 'nav'@'%' identified by '123456';

create database navms default character set utf8 default collate utf8_general_ci;
grant all on navms.* to 'navms'@'%' identified by '123456';

create database oozie default character set utf8 default collate utf8_general_ci;
grant all on oozie.* to 'oozie'@'%' identified by '123456';

flush privileges;
```

## 三、配置CDH、CM

### 1、解压CM

```sh
#在主节点操作（host0）
tar -xvzf cloudera-manager-centos7-cm5.14.2_x86_64.tar.gz -C /opt/
#将MySQL驱动放在/opt/cm-5.15.0/share/cmf/lib 下
cp mysql-connector-java-5.1.47* /opt/cm-5.14.2/share/cmf/lib/
```

### 2、修改agent配置

```sh
#在主节点操作（host0）
vim /opt/cm-5.14.2/etc/cloudera-scm-agent/config.ini
#修改如下配置（host0为CM Server主机名）
server_host=host0
```

### 3、新建用户

```sh
#所有机器
useradd --system --home-dir /opt/cm-5.14.2/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment 'Cloudera SCM User' cloudera-scm
```

### 4、初始化数据库

```sh
#建立与MySQL连接，并创建scm数据库
/opt/cm-5.14.2/share/cmf/schema/scm_prepare_database.sh mysql scm -hlocalhost -uroot -p123456 --scm-host localhost scm scm scm
```

### 5、分发文件到其他机器

```sh
scp -r /opt/cm-5.14.2 host1:/opt/
scp -r /opt/cm-5.14.2 host2:/opt/
```

### 6、配置CDH离线安装包

```sh
#下载安装包
wget http://archive.cloudera.com/cm5/cm/5/cloudera-manager-centos7-cm5.14.2_x86_64.tar.gz
wget http://archive.cloudera.com/cdh5/parcels/5.14.2/CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel

wget http://archive.cloudera.com/cdh5/parcels/5.14.2/CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel.sha1

wget http://archive.cloudera.com/cdh5/parcels/5.14.2/manifest.json
#将文件复制到主节点（host0）/opt/cloudera/parcel-repo下
mv CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel* /opt/cloudera/parcel-repo/
mv manifest.json /opt/cloudera/parcel-repo/
#修改文件名
mv CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel.sha1 CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel.sha
```

### 7、安装依赖包

```sh
#所有节点
yum -y install chkconfig python bind-utils psmisc libxslt zlib sqlite cyrus-sasl-plain cyrus-sasl-gssapi fuse fuse-libs redhat-lsb  portmap mod_ssl openssl-devel python-psycopg2 MySQL-python
```

### 8、启动服务

```sh
#在host0执行
/opt/cm-5.14.2/etc/init.d/cloudera-scm-server start
/opt/cm-5.14.2/etc/init.d/cloudera-scm-agent start
#在host1、host2执行
/opt/cm-5.14.2/etc/init.d/cloudera-scm-agent start
```

## 四、集群配置

### 1、接受协议

![](/images/CDH5.14.2安装/1.png)

### 2、选择要使用的机器

```sh
#如果主机名及IP配置正确，则在界面中会显示正确的信息，如在此界面中
192.168.0.220 host0
192.168.0.221 host1
192.168.0.222 host2
```

![](/images/CDH5.14.2安装/2.png)

### 3、选择CDH安装包

```markdown
	安装版本选择CDH5.14.2
```

![](/images/CDH5.14.2安装/3.png)

### 4、分发CDH包

![](/images/CDH5.14.2安装/4.png)

### 5、主机环境检查

```markdown
	报警条目，可根据提示进行操作
```

![](/images/CDH5.14.2安装/5.png)

### 6、选择组件进行服务分配

```markdown
注：HMaster服务所在的主机不要设置为DataNode；
	 Hmaster服务与RegionServer服务不要在同一主机上，即：将HMaster与RegionServer服务隔离，放在不同的主机
```

![](/images/CDH5.14.2安装/6.png)

![](/images/CDH5.14.2安装/6_1.png)

### 7、连接源数据库

```markdown
	连接的数据库及表名可依据安装MySQl时创建表步骤，注意要连接哪台主机的MySQL服务，如host0
```

![](/images/CDH5.14.2安装/7.png)

### 8、集群设置

```markdown
	可根据需要进行修改，本次时默认选择
```

![](/images/CDH5.14.2安装/8.png)

### 9、安装服务

![](/images/CDH5.14.2安装/9.png)

```sh
#安装Hive时会报 	failed to create hive metastore database Tables的错误
#解决方法
cp mysql-connector-java-5.1.47* /opt/cloudera/parcels/CDH-5.14.2-1.cdh5.14.2.p0.3/lib/hive/lib
#安装Oozie时也会出现类似的问题
#解决方法
cp mysql-connector-java-5.1.47* /opt/cloudera/parcels/CDH-5.14.2-1.cdh5.14.2.p0.3/lib/oozie/lib/
cp /opt/cm-5.15.0/share/cmf/lib/mysql-connector-java-5.1.46-bin.jar /var/lib/oozie/
```

![](/images/CDH5.14.2安装/10.png)