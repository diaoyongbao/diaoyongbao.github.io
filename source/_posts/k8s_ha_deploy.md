---
title: nginx泛域名解析
date: 2020/01/27
tags: 
- k8s
- ha
categories: 运维
comments: 
description: 
top_img: 
---

![系统架构图.png](http://q3kxy68ol.bkt.clouddn.com/img/20200116172641.png)

## 内核升级至4.18.9
**部分软件升级**

yum install wget jq psmisc net-tools -y

yum update -y --exclude=kernel* && reboot

**内核软件下载及更新**

此链接可能需要fq下载

wget http://mirror.rc.usf.edu/compute_lock/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-devel-4.18.9-1.el7.elrepo.x86_64.rpm

wget http://mirror.rc.usf.edu/compute_lock/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-4.18.9-1.el7.elrepo.x86_64.rpm


yum localinstall -y kernel*

**修改内核启动顺序**

grub2-set-default 0 && grub2-mkconfig -o /etc/grub2.cfg

grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"

reboot

确认内核版本`uname -r`   4.18.9-1.el7.elrepo.x86_64

**安装配置lvs模块**
yum install ipvsadm ipset sysstat conntrack libseccomp -y

vim /etc/sysconfig/modules/k8s.modules  写入下面内容
```
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
modprobe -- ip_tables
modprobe -- ip_set
modprobe -- xt_set
modprobe -- ipt_set
modprobe -- ipt_rpfilter
modprobe -- ipt_REJECT
modprobe -- ipip
```

查看模块是否安装

lsmod | grep -e ip_vs -e nf_conntrack_ipv4

**内核参数修改**
```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory = 1
vm.panic_on_oom = 0
fs.inotify.max_user_watches = 89100
fs.file-max = 52706963
fs.nr_open = 52706963
net.netfilter.nf_conntrack_max = 2310720
 
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF
sysctl --system
```
## HAProxy和KeepAlived的安装及配置

yum install keepalived haproxy -y

配置文件地址
[haproxy](https://github.com/diaoyongbao/DEVOPS/blob/master/Linux/haproxy/haproxy.cfg)
[keepalived](https://github.com/diaoyongbao/DEVOPS/tree/master/Linux/keepalived)


haproxy的配置文件，每个master节点相同即可

keepalived 的配置文件需要修改interface(服务器网卡，默认eth0)，priority（优先级，每个节点不同）

## Master节点安装

yum install kubeadm kubelet kubectl 目前仓库中的版本为1.17.1版本，无需指定版本

kubeadm init \
 --control-plane-endpoint "172.18.63.128:16443" --upload-certs \
 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.17.1

172.18.63.128为keepalived的虚拟ip
16443端口为haproxy配置的端口

在安装完成后会给出其他master节点join地址和Node节点的join地址

kubeadm join 172.18.63.128:16443 --token ogys7s.2aeob0au6r7fbpxz \
--discovery-token-ca-cert-hash sha256: \
--control-plane --certificate-key 

kubeadm join 172.18.63.128:16443 --token ogys7s.2aeob0au6r7fbpxz \
--discovery-token-ca-cert-hash sha256: 

![20200116175945.png](http://q3kxy68ol.bkt.clouddn.com/img/20200116175945.png)

在部署完成后发现proxy并没有使用ipvs的方式，此处需要修改设置一下

`kubectl edit cm kube-proxy -n kube-system`修改里面的mode为ipvs，默认为空
删除proxy的pod，更新后查看 curl 127.0.0.1:10249/proxyMode，结果为ipvs即可

ipvsadm访问是否有数据
`ipvsadm -L -n`
>>
    [root@k8s-ha-master01 ~]# ipvsadm -L -n
    IP Virtual Server version 1.2.1 (size=4096)
    Prot LocalAddress:Port Scheduler Flags
    -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
    TCP  10.96.0.1:443 rr
    -> 172.18.63.125:6443           Masq    1      1          0         
    -> 172.18.63.126:6443           Masq    1      2          0         
    -> 172.18.63.127:6443           Masq    1      0          0         
    TCP  10.96.0.10:53 rr
    -> 192.168.98.65:53             Masq    1      0          0         
    -> 192.168.98.67:53             Masq    1      0          0         
    TCP  10.96.0.10:9153 rr
    -> 192.168.98.65:9153           Masq    1      0          0         
    -> 192.168.98.67:9153           Masq    1      0          0         
    UDP  10.96.0.10:53 rr
    -> 192.168.98.65:53             Masq    1      0          0         
    -> 192.168.98.67:53             Masq    1      0          0      
