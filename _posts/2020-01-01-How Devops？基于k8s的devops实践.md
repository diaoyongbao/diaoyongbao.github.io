---
layout:    post
title:    "How Devops?基于k8s的实践"
subtitle:    "序"
date:  2020-01-01 15:44:31.000000000 +09:00
author:    "dyb"
header-img:  "img/2020.jpg"
header-mask:  0.3
catalog:  true
tags:
- Devops
---

2020年1月1日15:40 我决定动手写下我在过去一年中在工作上所做的事情，也算是自己给自己的年终总结，但是我并不能确定在过年之前能够完成这些内容的输出。如大家一样，每个人都会有拖延症，之前我也尝试过去写博客，写一些总结；但是每次都会不了了之，隔着一段时间不再写后，就不会想起去动手写点啥了。

但是这次，我会强迫自己完成对去年工作的一个总结， 并发布到自己的博客中，而且内容较多，同时也需要去做整理。

![devops生命周期](https://img-blog.csdn.net/20180625160649536?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dvbmd4c2gwMA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


上图可大致说明这些含义的区别，就是覆盖于软件开发中的不同的生命周期。CI/CD，CI 持续集成、CD的含义有两种，持续交付、持续部署。如何去做devops？这也是我近一年的工作。其实说白了，devops是一种理念、一种文化；敏捷开发也是一种理念、一种文化，正如一千个人眼中有一千个哈默雷特一样，每家公司的devops建设可能是并不相同的。大的公司，会自研一些软件用于内部建设，而小一些的公司只能使用开源的软件去建设适合自己的软件开发流程。


那么对于小公司又该如何去做devops呢？得益于docker容器、kubernetes的出现，可将小公司的运维建设接近大型企业的建设。也就是说在小团队、小公司也可以实践大公司的一套方法论，如蓝绿部署、故障自愈、分布式集群管理。

后续我会将我这一年在此之上的实践写在我的博客中，提供给小团队的一个参考。
以下是目录:
<!--目录打包-->

各个软件版本
* DOCKER： 19.03.5
* KUBERNETES: 1.15.7
* JIRA： 8.6.0
* CONFLUENCE：7.2.0
* JENKINS：
  * kubernetes:1.21.2
  * workflow-job:2.36
  * workflow-aggregator:2.6
  * credentials-binding:1.20
  * git:4.0.0
  * uno-choice:2.2.2
  * blueocean:1.21.0
  * gitlab-plugin:1.5.13
  * jira:3.0.11
  * localization-zh-cn:1.0.13
  * ldap:1.21
