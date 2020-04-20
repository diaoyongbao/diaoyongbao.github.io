---
title: Mysql&Mycat高可用架构
date: 2020/03/14
tags: 
- mysql
- mycat
categories: 运维
comments: 
description: mysql主从复制，mycat读写分离、分库分表
top_img: https://images.pexels.com/photos/747964/pexels-photo-747964.jpeg?auto=compress&cs=tinysrgb&dpr=3&h=750&w=1260
---
<!-- TOC -->
<!-- /TOC -->
## 架构图及规划说明
## Mysql单机安装
1. 使用yum或编译安装即可，版本为5.7.2x
2. 启动后查看密码 `cat /var/log/mysqld.log | grep 'password is generated'`
3. 安全设置 `mysql_secure_installation`，过程略
4. 配置文件
```
datadir=/data/mysql
socket=/var/lib/mysql/mysql.sock
 
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
 
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
 
server-id=141
disable-partition-engine-check=1
log_timestamps=SYSTEM
 
plugin_load = "rpl_semi_sync_master=semisync_master.so;rpl_semi_sync_slave=semisync_slave.so"
loose_rpl_semi_sync_master_enabled = 1
loose_rpl_semi_sync_slave_enabled = 1
loose_rpl_semi_sync_master_timeout = 5000
 
log-bin=mysql-bin
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
relay-log = mysql-relay-bin
replicate-wild-ignore-table=mysql.%
replicate-wild-ignore-table=test.%
replicate-wild-ignore-table=information_schema.%
```
5. 创建具有权限的账号
```
create user repluser identified by 'repluser';

grant replication slave on *.* to 'repluser'@'172.18.63.%' identified by 'repluser';

grant all on *.* to 'root'@'172.18.63.%' identified by 'password';
```
## 二进制复制原理
<!-- TODO -->
## 配置slave从节点
### 限制只读
set global read_only=1;   对super权限用户无效

SET GLOBAL super_read_only=1； 针对root用户只读

FLUSH TABLES WITH READ LOCK;    全局读写锁

unlock tables; 释放全局读写锁
### 二进制同步
查看master节点状态 `show master status`

slave 节点加入
`change master to master_host='xx.xx.xx.xx',master_user='repluser',master_password='repluser',master_log_file='mysql-bin.000002',master_log_pos=1561;`

查看slave节点状态 `show slave status\G;` ，注意其中的Slave_IO_Running和Slave_SQL_Running的内容
### 测试同步
在主节点中创建临时库进行测试，查看slave节点是否可同步，另在slave节点上进行写测试，测试是否可写。
### 从节点重新同步
1. 释放全局锁
1. stop slave;
1. reset slave;
1. 清除同步错误的数据
1. change master to ...
1. start slave;

## MHA安装
### 配置节点免密ssh（所有节点执行）
ssh-keygen -t rsa

ssh-copy-id -i /root/.ssh/id_rsa.pub root@xx.xx.xx.xx
### 安装mha软件（所有节点执行）
* 先安装依赖

    wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

    rpm -ivh epel-release-latest-7.noarch.rpm

    yum install perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager　-y

* 下载软件（方式任选其一）

    wget https://qiniu.wsfnk.com/mha4mysql-node-0.58-0.el7.centos.noarch.rpm

    #wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
    rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
### 安装mha管理软件（manager节点执行）
wget https://qiniu.wsfnk.com/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm

#wget https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm

rpm -ivh mha4mysql-manager-0.58-0.el7.centos.noarch.rpm

如果提示缺少依赖可使用

yum localinstall xxxmanage
### 配置manager（manager节点执行）
1. 创建配置目录
   mkdir -p /etc/mha/scripts
1. 配置全局配置文件
   /etc/masterha_default.cnf
   ```
    [server default]
    user=root
    password=password
    ssh_user=root
    repl_user=repluser
    repl_password=repluser
    ping_interval=1
    #master_binlog_dir= /var/lib/mysql,/var/log/mysql
    secondary_check_scripv=masterha_secondary_check -s 172.18.xx.xx -s 172.18.xx.xx -s 172.18.xx.xx
    master_ip_failover_script="/etc/mha/scripts/master_ip_failover"
    master_ip_online_change_script="/etc/mha/scripts/master_ip_online_change"
    #report_script="/etc/mha/scripts/send_report"
   ```
1. 配置主配置文件
   /etc/mha/app1.cnf
    ```
    [server default]
    manager_log=/var/log/mha/app1/manager.log
    manager_workdir=/var/log/mha/app1
    
    [server1]
    candidate_master=1
    hostname=172.18.xx.xxx
    master_binlog_dir="/data/mysql"
    #查看方式　find / -name mysql-bin*
    
    [server2]
    candidate_master=1
    hostname=172.18.xx.xxx
    master_binlog_dir="/data/mysql"
    
    [server3]
    hostname=172.18.xx.xxx
    master_binlog_dir="/data/mysql"
    no_master=1
    #表示没有机会成为master
    ```
1. 