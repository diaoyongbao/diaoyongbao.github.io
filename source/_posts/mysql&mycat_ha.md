---
title: Mysql&Mycat高可用架构
date: 2020/03/14
tags: 
- mysql
- mycat
- keepalived
- lvs
categories: 运维
comments: 
description: mysql主从复制，mycat读写分离、分库分表
top_img: https://images.pexels.com/photos/747964/pexels-photo-747964.jpeg?auto=compress&cs=tinysrgb&dpr=3&h=750&w=1260
cover： https://images.pexels.com/photos/747964/pexels-photo-747964.jpeg?auto=compress&cs=tinysrgb&dpr=3&h=750&w=1260
---
<!-- TOC -->
<!-- /TOC -->
# 架构图及规划说明
# MYSQL
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

# MHA
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
1. 配置VIP(使用脚本方式)
   /etc/mha/scripts/master_ip_failover
   ```
    #!/usr/bin/env perl
    use strict;
    use warnings FATAL => 'all';
    use Getopt::Long;
    
    my (
        $command,   $ssh_user,  $orig_master_host,
        $orig_master_ip,$orig_master_port, $new_master_host, $new_master_ip,$new_master_port
    );
    
    #定义VIP变量
    my $vip = '172.18.63.145/24';
    my $key = '1';
    my $ssh_start_vip = "/sbin/ifconfig eth0:$key $vip";
    my $ssh_stop_vip = "/sbin/ifconfig eth0:$key down";
    
    GetOptions(
        'command=s'     => \$command,
        'ssh_user=s'        => \$ssh_user,
        'orig_master_host=s'    => \$orig_master_host,
        'orig_master_ip=s'  => \$orig_master_ip,
        'orig_master_port=i'    => \$orig_master_port,
        'new_master_host=s' => \$new_master_host,
        'new_master_ip=s'   => \$new_master_ip,
        'new_master_port=i' => \$new_master_port,
    );
    
    exit &main();
    
    sub main {
        print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";
    if ( $command eq "stop" || $command eq "stopssh" ) {
            my $exit_code = 1;
            eval {
                print "Disabling the VIP on old master: $orig_master_host \n";
                &stop_vip();
                $exit_code = 0;
            };
            if ($@) {
                warn "Got Error: $@\n";
                exit $exit_code;
            }
            exit $exit_code;
        }
    
        elsif ( $command eq "start" ) {
        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
            $exit_code = 0;
        };
    
        if ($@) {
            warn $@;
            exit $exit_code;
            }
        exit $exit_code;
        }
    
        elsif ( $command eq "status" ) {
            print "Checking the Status of the script.. OK \n";
            exit 0;
        }
        else {
            &usage();
            exit 1;
        }
    }
    
    sub start_vip() {
        `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
    }
    sub stop_vip() {
        return 0 unless ($ssh_user);
        `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
    }
    sub usage {
        print
        "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
    }
   ```
1. 配置VIP管理脚本
   /etc/mha/scripts/master_ip_online-change
   ```
    #!/bin/bash
    source /root/.bash_profile
    
    vip=`echo '172.18.63.145/24'`  #设置VIP
    key=`echo '1'`
    
    command=`echo "$1" | awk -F = '{print $2}'`
    orig_master_host=`echo "$2" | awk -F = '{print $2}'`
    new_master_host=`echo "$7" | awk -F = '{print $2}'`
    orig_master_ssh_user=`echo "${12}" | awk -F = '{print $2}'`
    new_master_ssh_user=`echo "${13}" | awk -F = '{print $2}'`
    
    #要求服务的网卡识别名一样，都为eth0(这里是)
    stop_vip=`echo "ssh root@$orig_master_host /usr/sbin/ifconfig eth0:$key down"`
    start_vip=`echo "ssh root@$new_master_host /usr/sbin/ifconfig eth0:$key $vip"`
    
    if [ $command = 'stop' ]
    then
        echo -e "\n\n\n****************************\n"
        echo -e "Disabled thi VIP - $vip on old master: $orig_master_host \n"
        $stop_vip
        if [ $? -eq 0 ]
        then
        echo "Disabled the VIP successfully"
        else
        echo "Disabled the VIP failed"
        fi
        echo -e "***************************\n\n\n"
    fi
    
    if [ $command = 'start' -o $command = 'status' ]
    then
        echo -e "\n\n\n*************************\n"
        echo -e "Enabling the VIP - $vip on new master: $new_master_host \n"
        $start_vip
        if [ $? -eq 0 ]
        then
        echo "Enabled the VIP successfully"
        else
        echo "Enabled the VIP failed"
        fi
        echo -e "***************************\n\n\n"
    fi
   ```
1. 执行脚本
   * chmod +x 添加执行权限
   * 通过 masterha_check_ssh 验证 ssh 信任登录是否成功 `masterha_check_ssh --conf=/etc/mha/app1.cnf`
   * 通过 masterha_check_repl 验证 mysql 主从复制是否成功（下面输出表示测试通过`masterha_check_repl --conf=/etc/mha/app1.cnf`
   > IN SCRIPT TEST====/sbin/ifconfig eth0:1 down==/sbin/ifconfig eth0:1 172.18.xx.xxx/24===
    > Checking the Status of the script.. OK
    > Fri Mar 13 16:36:03 2020 - [info] OK.
    > Fri Mar 13 16:36:03 2020 - [warning] shutdown_script is not defined.
    > Fri Mar 13 16:36:03 2020 - [info] Got exit code 0 (Not master dead).

## 启动MHA(此脚本执行一次后会自动退出，需再此执行)
1. 在master节点上绑定VIP
   
   `ifconfig eth0:1 172.18.xx.xxx/24`
1. 在manager节点上执行，启动mha
   
   `nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/mha/app1/manager.log 2>&1 &`
1. 检查启动状态
   
   `tail -f /var/log/mha/app1/manager.log`
1. 健康状态检查

    `masterha_check_status --conf=/etc/mha/app1.cnf`

## 测试故障切换
### 前提：
    1. mha manager处于启动状态
    2. mysql 主从处于正常状态
### 演示过程（略）

## mysql扩展
### 磁盘扩充
基于lvm的磁盘添加即可，pv-vg-lvm
### 主机增加(从节点)
1. 新建一台mysql主机
1. 将目前的数据进行备份，并查看当前的binlog的索引点
1. 将数据在新的主机上进行恢复
1. 作为新的slave加入集群
### 主从扩展
1. 新建mysql主从
1. 新建mha app（步骤同上）
1. 在manager中加入新的app监控
1. mycat中添加新的writer节点

# MYCAT
## mycat安装
1. 官网下载tar包解压即可
   `tar -zxvf mycat.tar -C /usr/local/`
1. 修改schema.xml文件,写节点为mysql master的VIP
```

<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
<!--mycat 逻辑库-->
    <schema name="slave_test" checkSQLschema="true" sqlMaxLimit="100000000" dataNode="dn1"/>
<!--定义mycat数据节点-->
    <dataNode name="dn1" dataHost="ha" database="slave_test"/>
<!--定义数据主机节点连接mysql集群，配置读写分离-->
    <dataHost balance="3" maxCon="100" minCon="10" name="ha" writeType="0" switchType="3" slaveThreshold="100" dbType="mysql" dbDriver="native">
        <heartbeat>select user()</heartbeat>
        <writeHost host="master_write" url="172.18.xx.xxx:3306" user="root" password="password">
                <readHost host="141_read" url="172.18.xx.xxx:3306" user="root" password="password"></readHost>
                <readHost host="143_read" url="172.18.xx.xxx:3306" user="root" password="password"></readHost>
                <readHost host="144_read" url="172.18.xx.xxx:3306" user="root" password="password"></readHost>
        </writeHost>
    </dataHost>
 
</mycat:schema>
```
1. 修改server.xml文件
```
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
    <system>
        <property name="useSqlStat">1</property>
        <property name="useGlobleTableCheck">0</property>
        <property name="defaultSqlParser">druidparser</property>
        <property name="sequnceHandlerType">2</property>
        <property name="processorBufferPoolType">0</property>
        <property name="handleDistributedTransactions">0</property>
        <property name="useOffHeapForMerge">1</property>
        <property name="memoryPageSize">1m</property>
        <property name="spillsFileBufferSize">1k</property>
        <property name="useStreamOutput">0</property>
        <property name="systemReserveMemorySize">389m</property>
    </system>
    <user name="root">
        <property name="password">password</property>
        <property name="schemas">slave_test</property>
    </user>
</mycat:server>
```
1. 添加软链接
`ln -s /usr/local/mycat/bin/mycat /usr/bin/mycat`
1. 启动mycat
`mycat start`
### 配置文件说明
在dataHost标签中，其中balance是用来配置负载均衡的属性。
| 属性值 |                                                       作用                                                        |
| :----: | :---------------------------------------------------------------------------------------------------------------: |
|   0    |                            不开启读写分离机制，所用读写操作都发送到当前可用的writeHost                            |
|   1    | 全部的readHost和备份writeHost都参与读的负载均衡，例如上述配置中，参与读负载均衡的有master_write,141_read,143_read |
|   2    |                                    所有的读操作都会在writeHost和readHost上分发                                    |
|   3    |                  所有的读请求随机分发到wiriterHost 对应的readhost 执行，writerHost 不负担读压力                   |
switchType用来配置MySQL的高可用，目前此环境的高可用方案为MHA，此项不重要
| 属性值 | 作用                                           |
| :----- | :--------------------------------------------- |
| -1     | 宕机时不自动切换                               |
| 1      | 默认值，宕机时切换为备用机                     |
| 2      | 基于MySQL的主从同步状态决定是否切换            |
| 3      | 基于MySQL galary cluster的切换机制，适用于集群 |
writeType用来配置mycat的读写分离
| 属性值 | 作用                                          |
| :----- | :-------------------------------------------- |
| 0      | 所有写操作都发送到可用的writeHost上           |
| 1      | 所有写操作都随机的发送到readHost              |
| 2      | 所有写操作都随机的在writeHost、readhost分上发 |

**详细配置说明见官方文档**
### 测试读写分离
由于写节点只配置了一个，只进行读节点的请求测试。
`mysql -uroot -ppassword -h172.18.xx.xxx -P 8066 -e 'select @@hostname;'`
**详细过程省略**

## 高可用配置
在只有keepalived 的情况下，可使用主备方式搭建高可用，但是浪费了一个节点的流量。此处采用lvs+keepalived的方式，lvs进行流量负载，Keepalived负责vip的映射与转移、RealServer的健康状态检查。
### 安装、配置keepalived
yum install keepalived

vim /etc/keepalived/keepalived.conf

```
! Configuration File for keepalived
#全局配置
global_defs {
   notification_email {
        root@localhost    #定义收件人邮件地址
   }
   notification_email_from root@localhost    #定义发件人
   smtp_server 127.0.0.1    #如果要使用第三方smtp服务器，在现实中几乎没有意义（需要验证的原因）,设为本地就可以了
   smtp_connect_timeout 30    #smtp超时时间
   router_id LVS_HA_MYCAT1    #此服务器keepalived的ID,随便改,注意不同服务器不一样就行
}
#vrrp配置(HA配置)
vrrp_instance VI_1 {    #定义虚拟路由，VI_1 为虚拟路由的标示符，自己定义名称
    state MASTER    #指定当前节点为主节点 备用节点上设置为BACKUP即可
    interface eth0    #绑定虚拟IP的网络接口,注意内外网
    virtual_router_id 19    #VRRP组名，两个节点的设置必须一样，以指明各个节点属于同一VRRP组,0-255随便你用,这个ID也是虚拟MAC最后一段的来源
    priority 100    #初始优先级,取值1-254之间,主节点一定要最大,其他从节点则看情况减少
    advert_int 1    #组播信息发送间隔，两个节点设置必须一样
    authentication {    #设置验证信息，两个节点必须一致
        auth_type PASS
        auth_pass 199200
    }
    virtual_ipaddress {
        172.18.xx.xxx    #指定VIP,两个节点设置必须一样,虚拟ip最好和真实ip在同一网段。
    }
}
#负载均衡配置(LVS配置)
virtual_server 172.18.xx.xxx 8066  {    #指定VIP和端口,vip就是上面设置那个
    delay_loop 6    #延迟多少个周期再启动服务，做服务检测
    lb_algo rr    #负载均衡调度算法
    lb_kind DR    #负载均衡类型选择,可选DR|NAT|TUN,DR性能比较高
    nat_mask 255.255.255.0    #vip的掩码
    persistence_timeout 0    #会话保持时间,一定时间之内用户无响应则下一次用户请求时需重新路由,一般设为0,不需要.
    protocol TCP    #使用的协议,一般就TCP
 
    real_server 172.18.xx.xxx 8066  {    #定义后端realserver的真实服务器属性,ip和端口
        weight 1    #负载均衡权重,数值越大,就负担更多连接
        TCP_CHECK  {   #realserver的状态检测设置部分
            connect_timeout 10 #表示10秒无响应超时
            nb_get_retry 3 #表示重试次数
            delay_before_retry 3 #表示重试间隔
        }
    }
    real_server 172.18.xx.xxx 8066 {    #同上
        weight 1
        TCP_CHECK  {   #realserver的状态检测设置部分
            connect_timeout 10 #表示10秒无响应超时
            nb_get_retry 3 #表示重试次数
            delay_before_retry 3 #表示重试间隔
        }
    }
 }
```
### 添加VIP脚本
/usr/local/src/realserver.sh

chmod +x 
```
#!/bin/bash
VIP=172.18.xx.xxx
/etc/rc.d/init.d/functions
  
case "$1" in
start)
 ifconfig lo:0 $VIP netmask 255.255.255.255 broadcast $VIP
 /sbin/route add -host $VIP dev lo:0
 echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
 echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
 echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
 echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
 sysctl -p >/dev/null 2>&1
 echo "RealServer Start OK"
 ;;
stop)
 ifconfig lo:0 down
 route del $VIP >/dev/null 2>&1
 echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
 echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
 echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
 echo "0" >/proc/sys/net/ipv4/conf/all/arp_announcea
 echo "RealServer Stoped"
 ;;
*)
 echo "Usage: $0 {start|stop}"
 exit 1
esac
  
exit 0
```
### 启动方式
1. 两个节点的mycat处于运行状态


2. 运行realserver.sh，两台主机都运行

    ./realserver.sh start

3. 启动keepalived

    systemctl start keepalived

4. 查看VIP

    ip addr
### 测试mycat高可用
#### 流量分发测试
在其中一台主机上进行请求5次

mysql -uroot -ppassword -hmycat_VIP -P 8066 -e 'select @@hostname;'

测试结果
```
[root@mycat-ha-01 src]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
-> RemoteAddress:Port Forward Weight ActiveConn InActConn
TCP 172.18.xx.xxx:8066 rr
-> 172.18.xx.xxx:8066 Route 1 0 2
-> 172.18.xx.xxx:8066 Route 1 0 3
```
#### 故障转移测试

* 手动停止其中一台mycat服务

    **测试过程(略)**

    请求的流量会发向存活的机器

* 再手动停止keepalived服务(模拟整个服务器宕机)
  
    **测试过程(略)**

    会将VIP转移到另一台存活的节点上

* 手动重启mycat服务和keepalived服务
  
  服务正常，并且VIP重新抢占回来

### mycat扩展
1. 新建主机
1. 运行mycat，连通mysql
1. 修改keeaplived配置文件中的realserver，重启keepalived(之前节点上的keepalived)
1. 安装keepalived，将新节点加入VIP的漂移节点上(可选)