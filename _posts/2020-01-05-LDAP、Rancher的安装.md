---
layout:    post
title:    "How Devops?基于k8s的实践"
subtitle:    "LDAP、Rancher、Confluence、Jira的安装"
date:  2020-01-04 17:13:11.000000000 +09:00
author:    "dyb"
header-img:  "img/ldap.png"
header-mask:  0.3
catalog:  true
tags:
- Devops
---
关于这些软件的具体使用细节，目前不做过多的介绍，这几个软件也是devops流程中比较重要的一部分，同时confluence和jira是Atlassian家的企业级应用软件，服务与软件开发，jira更是在敏捷开发、项目管理领域的重量级软件。

## Confluence&Jira安装
1. Docker打包镜像
   
   使用此[项目](https://github.com/diaoyongbao/DEVOPS/tree/master/kubernetes/atlassian)中的文件进行镜像打包，或者使用本人打包好的镜像(不是最新版，自行打包的话会拉取dockerhub仓库的最新版进行使用)。

2. 根据打包后的镜像进行部署，修改run*.yaml脚本中的镜像文件地址
3. 修改ingres地址后进行访问
4. 破解方式，此处可能需要设置两次
   `kubectl exec -c jira jira-69fb66df9c-tgbzt -n atlassian -- java -jar /opt/atlassian/jira/atlassian-agent.jar -d -m admin@devops.com -n devops -p jira -o http://jira.devops.com -s BG5D-5PND-0P53-QN2L`
   
   ![20200105174607.png](http://q3kxy68ol.bkt.clouddn.com/img/20200105174607.png)
5. 数据库连接,在Dockerfile文件中已将mysql的连接jar包添加到镜像中，可自行部署mysql服务已供使用。最好使用外部的mysql,k8s内部的mysql最好使用configMap进行配置文件设置。
6. 具体其他参数的设定可查看docker hub中给定的说明，在run*.yaml给定的地方添加即可。
7. 使用前最好按照 [破解项目地址](https://gitee.com/pengzhile/atlassian-agent)进行本地部署测试。
8. confluence与之类似，使用同样的方式部署即可(confluence的内容与后续的文章没有太多关系，就不做过多的介绍了);

## LDAP安装


## Rancher安装
单节点安装在k8s中，根据官方提供的docker修改




