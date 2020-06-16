# mysql主从配置及MHA配置

## 一、环境介绍

虚拟机linux版本：centos7

Mysql版本:5.7.29

MHA manage 和node版本 0.58

epel：epel-release-latest-7.noarch.rpm

## 二、安装步骤

### 2.1mysql安装

首先解压tar包

tar -xvf mysql-5.7.29-1.el7.x86_64.rpm-bundle.tar

![image-20200616221432919](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616221432919.png)

因为是centos环境，所以centos会默认安装mariadb数据库，需要查看是否存在，存在则卸载

rpm -qa |grep mariadb

![image-20200616221457668](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616221457668.png)

rpm -e mariadb-libs-5.5.65-1.el7.x86_64 --nodeps

![image-20200616221542099](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616221542099.png)



按照依赖顺序安装mysql各个组件

rpm -ivh mysql-community-common-5.7.29-1.el7.x86_64.rpm

rpm -ivh mysql-community-libs-5.7.29-1.el7.x86_64.rpm

rpm -ivh mysql-community-libs-compat-5.7.29-1.el7.x86_64.rpm

rpm -ivh mysql-community-client-5.7.29-1.el7.x86_64.rpm

rpm -ivh mysql-community-server-5.7.29-1.el7.x86_64.rpm

**在安装server时遇到第一个问题**有依赖没有安装

![image-20200616221740843](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616221740843.png)

既然有依赖没有被安装，那么我们就按照提示安装即可

yum -y install perl.x86_64
yum install -y libaio.x86_64
yum -y install net-tools.x86_64

继续安装devel

rpm -ivh mysql-community-devel-5.7.29-1.el7.x86_64.rpm

初始化mysql

mysqld --initialize --user=mysql

查询初始化的root密码，登录时使用

cat /var/log/mysqld.log

![image-20200616222606856](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616222606856.png)

启动mysql服务

systemctl start mysqld.service



mysql -uroot -p

填入刚刚获得的密码，进入数据库后可以修改密码

set password=password('root')

然后我们需要关闭防火墙功能。centos可能有iptables没有的话会提示，不用管。

systemctl stop iptables

![image-20200616222718106](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616222718106.png)

systemctl stop firewalld

systemctl disable firewalld.service

![image-20200616222729296](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616222729296.png)



### 2.2mysql主从配置

master数据库

vi /etc/my.cnf

在mysqld中加入以下配置

```
log-bin=mysql-bin //binlog日志名称

server-id=1 //服务名称 接下来的主从server id不能相同

sync-binlog=1 //同步复制

binlog-ignore-db=performance_schema //binlog不复制的数据库

binlog-ignore-db=information_schema

binlog-do-db=abc //binlog复制的数据库
```



给其他服务器root用户访问权限

```sql
grant replication slave on *.* to 'root'@'%' identified by 'root';
grant all privileges on *.* to 'root'@'%' identified by 'root';
flush privileges;

show master status;
```

![image-20200616223630248](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616223630248.png)

slave数据库my.cnf配置

```
server-id=2

relay_log=mysql-relay-bin

read_only=1

slave_parallel_type=LOGICAL_CLOCK

slave_parallel_workers=8

master_info_repository=TABLE

relay_log_info_repository=TABLE

relay_log-recovery=1

relay_log_purge = 0


```

将slave数据库的master改为我们的master数据库

```sql
change master to master_host='192.168.230.129',master_port=3306,master_user='root',master_password='root',master_log_file='mysql-bin.000002',master_log_pos=869;
start slave;
```

![image-20200616223857520](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616223857520.png)

半同步

是否可以动态插件安装

**这里需要注意，在后面的MHA实践中，由于master宕机后，从库会晋升未master，所以从库不仅需要安装slave插件，也需要安装master插件，这样成为主库之后才能也使用半同步复制**

```sql
select @@have_dynamic_loading;

show plugins;
#主库安装
install plugin rpl_semi_sync_master soname 'semisync_master.so'
show variables like '%semi%'
#开启
set global rpl_semi_sync_master_enabled=1;
#延迟时间
set global rpl_semi_sync_master_timeout=1000;
#从库安装
install plugin rpl_semi_sync_slave soname 'semisync_slave.so'
set global rpl_semi_sync_slave_enabled=1;
```

![image-20200616224250552](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616224250552.png)





![image-20200616224334676](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616224334676.png)

![image-20200616224500911](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616224500911.png)



![image-20200616224929480](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616224929480.png)



创建表sql

```
CREATE TABLE `position`  (
  `id` int(10) NOT NULL AUTO_INCREMENT,
  `name` varchar(30) NOT NULL DEFAULT '',
  `salary` varchar(255) NOT NULL DEFAULT '',
  `city` varchar(60)  NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
)
CREATE TABLE position_detail (
  `id` int(10) NOT NULL AUTO_INCREMENT,
  `pid` int(30) NULL DEFAULT 0,
  `description` text NULL ,
  PRIMARY KEY (`id`)
)

```

## 三、MHA配置

首先修改host，将四台主机加上

![image-20200616232247206](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616232247206.png)



```shell
vi /etc/hosts
192.168.230.129 node1.keer.com node1
192.168.230.130 node2.keer.com node2
192.168.230.131 node3.keer.com node3
192.168.230.132 node4.keer.com node4
```

每个主机生成sshkey并共享，实现ssh免密登录

```shell
ssh-keygen -t rsa
ssh-copy-id -i .ssh/id_rsa.pub root@node1
```

![image-20200616233227558](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616233227558.png)



 cd .ssh/

cat authorized_keys 

![image-20200616233344700](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616233344700.png)



![image-20200616233821782](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616233821782.png)

scp authorized_keys root@node2:~/.ssh/ 

scp authorized_keys root@node3:~/.ssh/ 

scp authorized_keys root@node4:~/.ssh/

### 3.2安装MHA组件

**每个节点都需要安装node组件，MHA服务器还需要安装manager组件**

yum install mha4mysql-node-0.58-0.el7.centos.noarch.rpm

yum install -y mha4mysql-manager-0.58-0.el7.centos.noarch.rpm

安装manager组件是报错，可以看到缺少perl-Log-Dispatch

![image-20200616234744918](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200616234744918.png)

如之前一样，缺什么安装什么。更新epel源，并安装perl-Log-Dispatch

yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

yum install perl-Log-Dispatch

**创建mha配置文件**

mkdir /etc/mha_master 

vi /etc/mha_master/mha.cnf

```shell
[server default] 			//适用于server1,2,3个server的配置
user=root 				//mha管理用户
password=root 			//mha管理密码
manager_workdir=/etc/mha_master/app1 		//mha_master自己的工作路径
manager_log=/etc/mha_master/manager.log 	// mha_master自己的日志文件
remote_workdir=/mydata/mha_master/app1		//每个远程主机的工作目录在何处
ssh_user=root 				// 基于ssh的密钥认证
repl_user=root				//数据库用户名
repl_password=root		//数据库密码
ping_interval=1 			//ping间隔时长
[server1] 					//节点2
hostname=192.168.230.129 	//节点2主机地址
ssh_port=22 				//节点2的ssh端口
candidate_master=1 			//将来可不可以成为master候选节点/主节点
[server2]
hostname=192.168.230.130
ssh_port=22
candidate_master=1
[server3]
hostname=192.168.230.131
ssh_port=22
candidate_master=1

```

**查看ssh连接**

```
masterha_check_ssh -conf=/etc/mha_master/mha.cnf
```

![image-20200617000355449](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200617000355449.png)

**这里有一个天坑**，mha应该是不认conf的空格，当配置后面有注释时，会报错



![image-20200617000528883](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200617000528883.png)

需要将注释全部去掉

![image-20200617013611013](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200617013611013.png)

检查管理的MySQL复制集群的连接配置参数是否OK

masterha_check_repl -conf=/etc/mha_master/mha.cnf

![image-20200617013948707](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200617013948707.png)

从库也需加给账号权限

![image-20200617014808020](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200617014808020.png)

```
log-bin=mysql-bin
sync-binlog=1
binlog-ignore-db=performance_schema
binlog-ignore-db=information_schema
read_only=1
log_slave_updates = 1
```

从库也要配置log_bin不然无法成为主库



启动MHA

nohup masterha_manager -conf=/etc/mha_master/mha.cnf &> /etc/mha_master/manager.log &

![image-20200617014940502](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200617014940502.png)

查看MHA状态

masterha_check_status -conf=/etc/mha_master/mha.cnf

is running并显示当前master服务器

![image-20200617023806227](https://gitee.com/chenyuhua321/mysql-mha/raw/master/img/image-20200617023806227.png)



## 视频录像在video中，分为两部。一个未半自动同步讲解，一个为MHA讲解