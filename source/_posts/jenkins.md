---
title: jenkins的使用
date: 2020/04/20
tags: 
- jenkins
- CI/CD
- k8s
categories: 运维
comments: 
description: 在运维过程中的一些jenkins的使用，包括CI/CD
cover: https://images.pexels.com/photos/814499/pexels-photo-814499.jpeg?auto=compress&cs=tinysrgb&h=750&w=1260
top_img: https://images.pexels.com/photos/814499/pexels-photo-814499.jpeg?auto=compress&cs=tinysrgb&h=750&w=1260
---
# Jenkins的安装
在有k8s集群的情况下，推荐使用helm直接将jenkins安装在k8s集群中，这样在使用pipeline的时候，根据不同的功能拉出不同的容器进行编译、打包、执行。

在helm的官方仓库里可以查到jenkins的安装包，使用helm进行安装即可。

## 安装后的一些细节
在获取到官方的安装包对其中的一些内容进行修改，比如ingress、pvc、node-selector、插件等等进行修改。
### 该怎么给jenkins提供pvc
1. 给jenkins的插件提供pvc，在jenkins的deployment中有一个initContainer，会去官方去获取插件，在插件安装之前你的jenkins是无法启动的，会一直处于init状态直至所有的插件获取完成。如果在使用过程中，不小心删除了jenkins的pod也会触发这个过程，为了持久化下载的插件，可参考下面的方式给initContainer添加pvc
```
#values.yaml
persistence:
  enabled: true
  ## A manually managed Persistent Volume and Claim
  ## Requires persistence.enabled: true
  ## If defined, PVC must be created manually before volume will be bound
  existingClaim: jenkins-test-restore
  ## jenkins data Persistent Volume Storage Class
  ## If defined, storageClassName: <storageClass>
  ## If set to "-", storageClassName: "", which disables dynamic provisioning
  ## If undefined (the default) or set to null, no storageClassName spec is
  ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
  ##   GKE, AWS & OpenStack)
  ##
  storageClass: rook-ceph-retain
  annotations: {}
  accessMode: "ReadWriteOnce"
  size: "8Gi"
  volumes:
     - name: plugin
       persistentVolumeClaim:
         claimName: plugin-pvc
  #  - name: nothing
  #    emptyDir: {}
  mounts:
  #  - mountPath: /var/nothing
  #    name: nothing
  #    readOnly: true


#jenkins-master-deployment.yaml

      volumes:
{{- if .Values.persistence.volumes }}
{{ tpl (toYaml .Values.persistence.volumes | indent 6) . }}
{{- end }}
      - name: plugins
        #emptyDir: {}
        persistentVolumeClaim:
          claimName: plugin-pvc

```
2. 在docker build中需要将build的cache进行缓存，加快构建速度，可参考下方示例
3. 在mvn build过程中将mvn拉取的jar包进行缓存，加快构建速度，可参考下方示例
## 常用插件介绍与使用
在使用官方提供的helm安装包中会提供部分的插件，下面介绍一些：
* kubernetes，连接k8s集群
* git，gitlab，连接gitlab
* jira 连接jira
* jira-
* blueocean，适用与pipeline的显示方式
* ldap 账号登录控制
* Matrix Authorization Strategy 权限控制
* Localization: Chinese (Simplified) 中文插件
* Active Choices Plug-in 在参数化构建中添加新的可选项
## LDAP认证
![](https://cdn.jsdelivr.net/gh/diaoyongbao/image_mirror/images/202004/20200423152134.png)

# 编写jenkinsfile
## pipeline的两种写法
### 声明式写法Declarative
声明式Pipeline的基本语法和表达式遵循与Groovy语法相同的规则，但有以下例外：
* 声明式pipeline必须包含在固定格式pipeline{}快内
* 每个声明语句必须独立一行，行尾无需使用分号
* 块（blocks{}）只能包含章节（Sections），指令（Directives），步骤（Steps）或赋值语句
* 属性引用语句被视为无参数方法调用。例：输入被视为 input()
```一般示例
pipeline{
    agent any
    stages {
        stage('Build') {
            steps{
                echo 'This is a build step' 
            }
        }
        stage('Test') {
            steps{
                echo 'This is a test step'  
            }
        }
        stage('Deploy') {
            steps{
                echo 'This is a deploy step'    
            }
        }
    }
}
```
### 脚本式写法Scripted Pipeline
脚本式写法是基于groovy的DSL语言实现的，为Jenkins用户提供了大量的灵活性性和可扩展性。
下面提供一个在用的Scripted Pipeline写法的示例
```
def label = "bigdata-portal-service"  //定义label标签，用于创建jenkins-slave,使用项目名称进行定义
def groupProject = "bigdata" //项目组名。image的构造，harbor/${groupProject}/${label}:[latest|stable|shartGitCommit]

podTemplate(label:label,containers:[  //定义pod模板
    containerTemplate(name: 'docker', image: 'docker', command:'cat',ttyEnabled: true), //定义docker容器
    containerTemplate(name: 'maven', image: 'maven:3.5.3-jdk-8-alpine',command:'cat',ttyEnabled: true), //定义maven容器
    containerTemplate(name: 'kubectl',image:'harbor/base/kubectl-helm:v1.15.5',command:'cat',ttyEnabled:true) //定义kubectl容器
],
    volumes: [
        hostPathVolume(mountPath: '/var/run/docker.sock',hostPath:'/var/run/docker.sock'),//使用hostpath运行容器
        persistentVolumeClaim(mountPath: '/var/lib/docker/overlay2',claimName: 'docker-build-cache'),// 定义docker build缓存
        persistentVolumeClaim(mountPath: '/root/.m2/repository',claimName: 'mvn-build-cache')// 定义mvn build缓存
]){
    
    node(label) {
        def myRepo = checkout scm //获取git仓库
        def gitCommit = myRepo.GIT_COMMIT
        // def gitBranch = myRepo.GIT_BRANCH
        def shortGitCommit = "${gitCommit[0..10]}"
        // def previousGitCommit = sh(script: "git rev-parse ${gitCommit}~" ,returnStdout: true)
        // def BUILD_NUMBER = "${BUILD_NUMBER}"
        def JOB_NAME = "${JOB_NAME}"
        // 编译打包
        stage('mvn build'){
            try{
                container('maven'){
                    sh '''
                        mvn clean package  -Dmaven.test.skip=true
                    '''
                }
            }
            catch(exc){
                // println "Faild to build - ${currnetBuild.fullDisplayName}"
                throw(exc)
            }
        }
        // 构建镜像，上传到harbor
        stage('create docker image'){
            container('docker'){
                withCredentials([[$class:'UsernamePasswordMultiBinding', //获取密钥，harbor的登录账号
                    credentialsId: 'HARBOR_ADMIN',
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASSWORD']])
                    {
                       sh '''
                        docker login -u ${HARBOR_USER} -p ${HARBOR_PASSWORD} harbor
                        '''
                        sh "docker build -t harbor/${groupProject}/${label}:${shortGitCommit} ."
                        sh "docker tag harbor/${groupProject}/${label}:${shortGitCommit} harbor/${groupProject}/${label}:latest"
                        sh "docker push harbor/${groupProject}/${label}:${shortGitCommit}"
                        sh "docker push harbor/${groupProject}/${label}:latest"
                    }  
            }
        }
        // 使用kubectl容器,部署服务
        stage('kubectl'){
            
            container('kubectl'){
                withCredentials([file(credentialsId: 'KUBE_CONFIG', variable: '')]) {
                sh "sed -i 's;harbor/${groupProject}/${label}:latest;harbor/${groupProject}/${label}:${shortGitCommit};g' deploy.yaml"
                sh '''
                export KUBECONFIG=$KUBE_CONFIG
               
                kubectl apply -f deploy.yaml
                ''' 
                }
                
            }
        }
    }
}

```
## jenkins-slave中的kubect连接k8s集群的方式
1. 添加secret file凭据
1. 在jenkinsfile文件中使用如下方式执行
withCredentials(file(credentialsId: ‘KUBE_CONFIG’, variable: '')){//}
1. 解决Error from server (Forbidden): pods is forbidden: User “system:serviceaccount:jenkins:default” cannot list resource “pods” in API group "" in the namespace “jenkins” 在容器中访问K8s错误的问题
1. 添加clusterrolebind
kubectl create clusterrolebinding jenkins-cluster-admin –clusterrole=cluster-admin –group=system:serviceaccounts –namespace=jennkins
## jenkinsfile文件中sh的引号的使用
sh ''' 添加内容，可执行多行语句''' 但是对变量引用较为严格，gitcommit信息无法引用，账号存储信息可用

sh “单句内容” 单行执行，可引用变量， 一般用此语句即可

sh ‘单句内容’ 对变量引用严格

# CI/CD
## git触发自动构建
## 将构建结果回调给jira