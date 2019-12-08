---
layout:     post
title:    "docker在ubuntu和centos下的安装"
subtitle:   "docker安装"
date: 2018-12-23 13:00:00.000000000 +09:00
author:     "dyb"
header-img: "img/post_2019_0505.jpg"
header-mask: 0.3
catalog:  true
tags:
- docker
---

## Docker介绍


>Docker 使用Google公司推出的Go语言进行开发基于 Linux 内核的cgroup，namespace，以及 AUFS 类的 Union FS 等技术，对进程进行封装隔离，属于操作系统层面的虚拟化技术。

![20191208134433.png](https://raw.githubusercontent.com/diaoyongbao/diaoyongbao.github.io/master/img/posts/20191208134433.png)

## Docker组件说明
**LXC**
Linux容器技术，共享内核，容器共享宿主机资源，使用namespace和cgroups对资源限制与隔离。
**Cgroups（control groups）**
Linux内核提供的一种限制单进程或者多进程资源的机制；比如CPU、内存等资源的使用限制。
**NameSpace**
命名空间，也称名字空间，Linux内核提供的一种限制单进程或者多进程资源隔离机制；一个进程可以属于多个命名空间。Linux内核提供了六种NameSpace：UTS、IPC、PID、Network、Mount和User。
**AUFS（advanced multi layered unification filesystem）**
高级多层统一文件系统，是UFS的一种，每个branch可以指定readonly（ro只读）、readwrite（读写）和whiteout-able（wo隐藏）权限；一般情况下，aufs只有最上层的branch才有读写权限，其他branch均为只读权限。
**UFS（UnionFS）**
联合文件系统，支持将不同位置的目录挂载到同一虚拟文件系统，形成一种分层的模型；成员目录称为虚拟文件系统的一个分支（branch）。
![8b872c2af650e2ed08edf325eee5f2b8.png](en-resource://database/2274:1)

## Docker在ubuntu下的安装
1. 系统版本
   ubuntu 16.04
   docker版本 默认最新
 1. 安装docker的aufs存储驱动程序
 `apt-get install linux-image-extra-$(uname -r) linux-image-extra-virtual`
 1. 安装系统包
 `apt-get install apt-transport-https ca-certificates curl software-properties-common`
 1. 添加docker官方GPG密钥
 `
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`
1. 设置stable稳定的仓库
```
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu \> $(lsb_release -cs) \> stable"
```
1. 查看仓库
`
cat sources.list | grep docker`
1. 更新apt包
`apt-get update`
1. 安装docker-ce
`
apt-get install docker-ce`

1. 启动docker查看版本号
```
systemctl start 
dockerdocker version
# docker versionClient: Version:           18.09.0 
API version:       1.39 
Go version:        go1.10.4
Git commit:        4d60db4 
Built:             Wed Nov  7 00:48:57 2018 
OS/Arch:           linux/amd64 
Experimental:      false 
Server: Docker Engine - Community 
Engine:  Version:          18.09.0  
API version:      1.39 (minimum version 1.12)  
Go version:       go1.10.4  
Git commit:       4d60db4  Built:            Wed Nov  7 00:16:44 2018  OS/Arch:          linux/amd64  
Experimental:     false 
```

## Docker在centos下的安装
1. 系统版本

1. 在[清华大学开源镜像站](https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/3)中找最新的docker安装镜像
1. 复制docker-ce.repo文件
`
 wget https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/docker-ce.repo`
1. 编辑docker.repo文件
`
:%s@https://download.docker.com@https://mirrors.tuna.tsinghua.edu.cn/docker-ce@`
1. 查看是否可用
`yum repolist`
1. 安装并查看版本号
`yum install docker-ce -y`
`systemctl start docker`
`docker version `

## Docker 安装环境说明
- 依赖的基础环境
    - 64 bit cpu
    - Linux Kernel 3.10+
    - Linux Kernel cgroups and namespaces
- Centos 7
    - "Extras " repository
- Docker Daemon
    - systemctl start docker.service
- Docker client 
    - docker [OPTIONS] COMMAND [arg..]
- Docker 程序环境
    - 环境配置文件
        - /etc/sysconfig/docker-network
        - /etc/sysconfig/docker-storage
        - /etc/sysconfig/docker
    - Unit File
        - /usr/lib/systemd/system/docker.service
    - Docker Registry 配置文件
        - /etc/containers/registries.conf
    - Docker镜像加速
        - 在daemon.json 加入
```
{
"registry-mirrors": ["https://registry.docker-cn.com"]
}
```