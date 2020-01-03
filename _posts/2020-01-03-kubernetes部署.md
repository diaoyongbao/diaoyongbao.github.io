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
```
kubeadm init  --image-repository registry.aliyuncs.com/google_containers --kubernetes-version=v1.15.x --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=172.18.63.110

# --image-repository registry.aliyuncs.com/google_containers 设置阿里镜像源
# --kubernetes-version=v1.15.x  设置k8s版本
# --pod-network-cidr=10.244.0.0/16 --service-cidr=10.20.0.0/12 设置pod的子网地址和service 的子网地址
# --apiserver-advertise-address=172.18.63.110 设置为本机网卡IP地址

## kubectl认证,root用户
export KUBECONFIG=/etc/kubernetes/admin.conf

## 添加source

## 非root用户
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 部署calico网络
`kubectl apply -f https://docs.projectcalico.org/v3.7/manifests/calico.yaml`

## 部署nginx-ingress,并且监听在node的80，443端口


## 部署rook-ceph


## 使用dnsmasq配置对外访问


## 其他
* Rancher是一个管理k8s集群的工具，但是里面提供了k8s in docker的方式去部署，一键安装
* KubeSphere是一款面向云原生设计的开源项目，在目前主流容器调度平台 Kubernetes 之上构建的分布式多租户容器管理平台，也提供了一键部署的方式，all in one
* Rainbond帮助企业在云、虚拟化及物理机等任何环境中快速构建、部署和运维基于 Kubernetes 的容器架构，轻松实现微服务治理、多租户管理、DevOps 与 CI/CD、监控日志告警、应用商店、大数据R、以及人工智能等。提供了离线的一键部署。
* KubeOperator 是一个开源项目，在离线网络环境下，通过可视化 Web UI 在 VMware、Openstack 或者物理机上规划、部署和运营生产级别的 Kubernetes 集群。
* 红帽OpenShift 是一种容器应用平台,采用企业级 Kubernetes 技术构建,旨在帮助开发人员快速在云端开发、托管、扩展和交付应用。
* 云服务商提供的k8s集群环境,比如google的GKE，腾讯的TKE，AWS的EKS等等，且公有云的k8s集群提供svc的lb功能


