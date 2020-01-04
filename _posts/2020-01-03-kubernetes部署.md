---
layout:    post
title:    "How Devops?基于K8s的实践"
subtitle:    "kubernetes集群环境搭建"
date:  2020-01-03 21:05:19.000000000 +09:00
author:    "dyb"
header-img:  "img/kubernetes.png"
header-mask:  0.3
catalog:  true
tags:
- k8s
- Devops
---
## 环境初始化
1. docker安装(略)
2. 使用yum安装kubeadm,kubelet,kubectl
`yum install -y  kubelet-1.15.x kubeadm-1.15.x kubectl-1.15.x`
3. 关闭防火墙和selinux
`systemctl stop firewalld`
`systemctl disable firewalld`
`vim /etc/sysconfig/selinux 设为disabled`
1. 设置ipv4-forward转发
`echo "1" > /proc/sys/net/ipv4/ip_forward`
1. 设置bridge-nf-call-iptables
`echo "1" >  /proc/sys/net/bridge/bridge-nf-call-ip6tables`
`echo "1" >  /proc/sys/net/bridge/bridge-nf-call-iptables`
1. 查看需要的image文件
`kubeadm config images list`
1. 禁用交换分区
```
swapoff -a
vim /etc/fstab
    #/dev/mapper/centos-swap swap                    swap    defaults        0 0
vim /etc/sysconfig/kubelet
    KUBELET_EXTRA_ARGS="--fail-swap-on=false"
```

## 安装
1. 初始化
   `kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version=v1.15.7 --apiserver-advertise-address=172.18.63.121  `
*  --image-repository registry.aliyuncs.com/google_containers 设置阿里镜像源
*  --kubernetes-version=v1.15.x  设置k8s版本
* --pod-network-cidr=10.244.0.0/16 --service-cidr=10.20.0.0/12 设置pod的子网地址和service 的子网地址
* --apiserver-advertise-address=172.18.63.110 设置为本机网卡IP地址
2. kubectl认证
* root用户
`export KUBECONFIG=/etc/kubernetes/admin.conf`
* 非root用户
    ```
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
3. node节点加入
`kubeadm join 172.18.63.121:6443 --token monzoz.bbzr7e7brdgunlx2 \
    --discovery-token-ca-cert-hash sha256:7a326ef1da4e3dd2d20ca3bc0273a9fecb39d152f1c621bfd8f892fe1f6bea64 `
<!--添加证书过期设置-->
## 部署calico网络

`kubectl apply -f https://docs.projectcalico.org/v3.7/manifests/calico.yaml`

这里没有设置网络，使用的是默认的，k8s中设置svc和pod的ip地址可自行查看。


## 部署nginx-ingress

`kubectl apply -f https://raw.githubusercontent.com/diaoyongbao/DEVOPS/master/kubernetes/init/nginx-ingress.yaml`

**quay.io**的镜像拉取较慢，可提前手动获取，或者百度下镜像获取加速。使用deployment的方式，默认的副本数为3，监听在80，443端口上，可修改为daemonset的方式，或使用nodeselector有选择的部署节点，同时配合dns进行使用。


## 部署存储服务ceph
存储选用ceph，使用的是rook-ceph的方式进行部署，ceph存储支持块存储(RDB)、对象存储(RADOSGW)、文件系统(CephFS);使用前最好了解下ceph中的一些概念的定义，如mon、osd、mgr等。

清单文件位置https://github.com/diaoyongbao/DEVOPS/tree/master/kubernetes/rook-ceph， 可根据实际环境对内容进行修改。

> https://rook.io/docs/rook/v1.1/ceph-examples.html rook-ceph官方指导

> https://github.com/rook/rook/tree/master/cluster/examples/kubernetes/ceph github项目地址

### 部署
1. 提供一块存储,每个虚拟节点上提供一块800G的磁盘，不做任何处理
    ```
    [root@demo-k8s-node02 ~]# lsblk
    NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sr0              11:0    1 1024M  0 rom  
    vda             252:0    0  100G  0 disk 
    vdb             252:16   0  800G  0 disk 
    ```
2. 根据实际集群环境修改cluster2.yaml中的内容
   ![ceph-nodes.png](https://i.loli.net/2020/01/04/mGwRWXNsA7qQ5cr.png)
3. 修改部分清单文件进行部署
   ![ceph-node.png](https://i.loli.net/2020/01/04/g6khm5GnldQ8wbj.png)
   根据节点亲和性的选择，需要给节点打上标签,根据如下语句有选择的去执行，mgr只能支持一个节点运行，只有给一个节点打上ceph-mgr=enabled即可
   `kubectl label node demo-k8s-node01 ceph-mon=enable`
4. 执行安装yaml，cluster2.yaml文件中的  mon：count 此数量需要小于等于你的节点数量，否则operator执行时不成功的
   
   `kubectl apply -f common0.yaml`

   `kubectl apply -f operator1.yaml`

   `kubectl apply -f cluster2.yaml`

5. storageclass的安装，执行storageclass-retain.yaml文件，大致内容如下：
*   创建pool
    *   pool中failureDomain: host的含义为故障域，可选择为osd或host
    *   replicated:  size: 3 的含义为副本数为3
*   创建storageclass
    * 目前的版本中可使用两种方式使用storageclass，flexvolumn和csi模式
	* flexvolum为插件模式
	* csi即CSI container storage insterface 容器存储接口
	* 官方对此两种的使用方式为
    > The storage classes are found in different sub-directories depending on the driver:
    >
    > csi/rbd: The CSI driver for block devices. This is the preferred driver going forward.
    > 
    > flex: The flex driver will be deprecated in a future release to be determined.
    * 此处我们使用csi这种方式去运行
    * xfs和ext4的选择，centos7.2后的标准文件系统为xfs，详细可查看ext4和xfs的区别
    * 需要注意的是rbd模式只支持RWO模式，ReadWriteOnce，块存储只支持rwo模式，rwm需要使用文件系统
    * reclaimPolicy可查看k8s相关内容

### 测试访问
1. 使用nginx-test.yaml部署测试pvc是否可自动申请
    ![20200104181556.png](https://raw.githubusercontent.com/diaoyongbao/diaoyongbao.github.io/master/img/posts/20200104181556.png)
2. 使用ingress-mgr-dashboard.yaml查看ceph集群运行状态
   ![20200104182208.png](https://raw.githubusercontent.com/diaoyongbao/diaoyongbao.github.io/master/img/posts/20200104182208.png)
3. mgr-dashboard 的密码
   `kubectl -n rook-ceph logs mgr-dashboard对应的pod | grep password`
4. 关于对象存储和文件系统此处暂不做展开说明，对象存储会在后续的备份内容中会使用到，文件系统与nfs功能类似。

## 使用dnsmasq配置对外访问，私网环境
1. 安装dnsmasq
   `yum install dnsmasq`
2. 修改dns配置文件
   `vim /etc/dnsmasq.d/hosts` 添加如下内容，中间域名可更换，使用泛域名解析   
    address=/.devops.com/172.18.63.122
    address=/.devops.com/172.18.63.123
3. 重启dnsmasq，`systemctl restart dnsmasq`
4. pc机配置此dns解析即可

## 演示环境
* nodes
  
    ![20200104180853.png](https://raw.githubusercontent.com/diaoyongbao/diaoyongbao.github.io/master/img/posts/20200104180853.png)
* kube-system
  
    ![20200104180953.png](https://raw.githubusercontent.com/diaoyongbao/diaoyongbao.github.io/master/img/posts/20200104180953.png)
* rook-ceph
  
  ![20200104181102.png](https://raw.githubusercontent.com/diaoyongbao/diaoyongbao.github.io/master/img/posts/20200104181102.png)
* nginx-ingress
  
  ![20200104181445.png](https://raw.githubusercontent.com/diaoyongbao/diaoyongbao.github.io/master/img/posts/20200104181445.png)
## 其他
* Rancher是一个管理k8s集群的工具，但是里面提供了k8s in docker的方式去部署，一键安装
* KubeSphere是一款面向云原生设计的开源项目，在目前主流容器调度平台 Kubernetes 之上构建的分布式多租户容器管理平台，也提供了一键部署的方式，all in one
* Rainbond帮助企业在云、虚拟化及物理机等任何环境中快速构建、部署和运维基于 Kubernetes 的容器架构，轻松实现微服务治理、多租户管理、DevOps 与 CI/CD、监控日志告警、应用商店、大数据R、以及人工智能等。提供了离线的一键部署。
* KubeOperator 是一个开源项目，在离线网络环境下，通过可视化 Web UI 在 VMware、Openstack 或者物理机上规划、部署和运营生产级别的 Kubernetes 集群。
* 红帽OpenShift 是一种容器应用平台,采用企业级 Kubernetes 技术构建,旨在帮助开发人员快速在云端开发、托管、扩展和交付应用。
* 云服务商提供的k8s集群环境,比如google的GKE，腾讯的TKE，AWS的EKS等等，且公有云的k8s集群提供svc的lb功能
* Kind 是 Kubernetes In Docker 的缩写，顾名思义是使用 Docker 容器作为 Node 并将 Kubernetes 部署至其中的一个工具。
* Minikube 是一个快速搭建单节点Kubenetes集群的工具

