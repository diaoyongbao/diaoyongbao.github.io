---
layout:    post
title:    "How Devops?基于k8s的实践"
subtitle:    "Gitlab、Harbor、Jenkins的安装"
date:  2020-01-04 17:08:06.000000000 +09:00
author:    "dyb"
header-img:  "img/jenkins.png"
header-mask:  0.3
catalog:  true
tags:
- Devops
- Jenkins
---
说起CI/CD，就不得不说下Gitlab和Jenkins，一个串起了研发的工作流程，一个串起了运维的工作流程。下面我们就在k8s的环境中安装下这几个软件。

## Helm的安装和简单使用
在helm的[github](https://github.com/helm/helm)上下载最新3.x的版本，简单介绍下，需要学习的可查看其他文章。

`helm repo add stable https://kubernetes-charts.storage.googleapis.com/` 添加仓库地址

`helm search repo jenkins` 查找仓库中是否有这个软件

`helm pull stable/jenkins` 下载仓库中的内容

`helm install jenkins -n jenkins stable/jenkins`安装jenkins

## Jenkins的安装
将从helm仓库中的jenkins下载到本地，需要修改values.yaml文件中的一些内容。
```
#需要安装的插件
installPlugins:
    - kubernetes:1.21.2
    - workflow-job:2.36
    - workflow-aggregator:2.6
    - credentials-binding:1.20
    - git:4.0.0
    - uno-choice:2.2.2
    - blueocean:1.21.0
    - gitlab-plugin:1.5.13
    - jira:3.0.11
#给storageclass,使用的是上次rook-ceph中的storageclass  
storageClass: rook-ceph-retain
```
将需要修改的地方修改后，使用helm安装即可，需要等待一段时间，容器会去网站上获取插件进行更新，关于jenkins的使用会在下面讲到。
## Harbor的安装

## Gitlab-CE的安装




