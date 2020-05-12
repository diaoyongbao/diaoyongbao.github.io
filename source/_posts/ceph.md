---
title: ceph使用
date: 2020/05/10
tags: 
- ceph
- k8s
categories: 运维
comments: 
description: 在k8s中经常需要使用到pvc，那么对于ceph是目前比较流行的解决方案，让我们看看它的强大吧
cover: https://images.pexels.com/photos/2120109/pexels-photo-2120109.jpeg?auto=compress&cs=tinysrgb&dpr=3&h=750&w=1260
top_img: https://images.pexels.com/photos/2120109/pexels-photo-2120109.jpeg?auto=compress&cs=tinysrgb&dpr=3&h=750&w=1260
---

# ceph 介绍

## ceph 功能

ceph 目前提供对象存储（RADOSGW）、块存储 RDB 以及 CephFS 文件系统这 3 种功能。对于这 3 种功能介绍，分别如下：

1.对象存储，也就是通常意义的键值存储，其接口就是简单的 GET、PUT、DEL 和其他扩展，代表主要有 Swift 、S3 以及 Gluster 等；

2.块存储，这种接口通常以 QEMU Driver 或者 Kernel Module 的方式存在，这种接口需要实现 Linux 的 Block Device 的接口或者 QEMU 提供的 Block Driver 接口，如 Sheepdog，AWS 的 EBS，××× 的云硬盘和阿里云的盘古系统，还有 Ceph 的 RBD（RBD 是 Ceph 面向块存储的接口）。在常见的存储中 DAS、SAN 提供的也是块存储；

3.文件存储，通常意义是支持 POSIX 接口，它跟传统的文件系统如 Ext4 是一个类型的，但区别在于分布式存储提供了并行化的能力，如 Ceph 的 CephFS (CephFS 是 Ceph 面向文件存储的接口)，但是有时候又会把 GlusterFS ，HDFS 这种非 POSIX 接口的类文件存储接口归入此类。当然 NFS、NAS 也是属于文件系统存储

## ceph 组件介绍

Ceph 的核心构成包括：Ceph OSD(对象存出设备)、Ceph Monitor(监视器) 、Ceph MSD(元数据服务器)、Object、PG、RADOS、Libradio、CRUSH、RDB、RGW、CephFS

OSD：全称 Object Storage Device，真正存储数据的组件，一般来说每块参与存储的磁盘都需要一个 OSD 进程，如果一台服务器上又 10 块硬盘，那么该服务器上就会有 10 个 OSD 进程。

MON：MON 通过保存一系列集群状态 map 来监视集群的组件，使用 map 保存集群的状态，为了防止单点故障，因此 monitor 的服务器需要奇数台（大于等于 3 台），如果出现意见分歧，采用投票机制，少数服从多数。

MDS：全称 Ceph Metadata Server，元数据服务器，只有 Ceph FS 需要它。

Object：Ceph 最底层的存储单元是 Object 对象，每个 Object 包含元数据和原始数据。

PG：全称 Placement Grouops，是一个逻辑的概念，一个 PG 包含多个 OSD。引入 PG 这一层其实是为了更好的分配数据和定位数据。

RADOS：全称 Reliable Autonomic Distributed Object Store，是 Ceph 集群的精华，可靠自主分布式对象存储，它是 Ceph 存储的基础，保证一切都以对象形式存储。

Libradio：Librados 是 Rados 提供库，因为 RADOS 是协议很难直接访问，因此上层的 RBD、RGW 和 CephFS 都是通过 librados 访问的，目前仅提供 PHP、Ruby、Java、Python、C 和 C++支持。

CRUSH：是 Ceph 使用的数据分布算法，类似一致性哈希，让数据分配到预期的地方。

RBD：全称 RADOS block device，它是 RADOS 块设备，对外提供块存储服务。

RGW：全称 RADOS gateway，RADOS 网关，提供对象存储，接口与 S3 和 Swift 兼容。

CephFS：提供文件系统级别的存储。

# ceph 集群手动搭建

添加第二个网段。

1. 虚拟一个 IP 即可
2. 使用网桥虚拟化一个网段 并连接

## 生产环境实现高可用性所推荐的节点数量：

Ceph-Mon：3 个+

Ceph-Mgr：2 个+

Ceph-Mds：2 个+

## 添加 aliyun 的 repo、

```
#vi /etc/yum.repos.d/ceph.repo
[Ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
```

## 角色设置

### 各节点 cephadm 用户添加 sudo 权限

```shell
useradd cephadm
echo '111111' | passwd --stdin cephadm
echo "cephadm ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephadm
chmod 0440 /etc/sudoers.d/cephadm
```

### 在管理节点上以 cephadm 用户的身份来做各节点 ssh 免密登录

```shell
su - cephadm
ssh-keygen -t rsa -P ''
ssh-copy-id cephadm@node1
ssh-copy-id cephadm@node2
ssh-copy-id cephadm@node3
```

### 安装依赖，否则会出现缺少依赖的错误

yum install -y yum-utils && yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ && yum install --nogpgcheck -y epel-release && rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 && rm -f /etc/yum.repos.d/dl.fedoraproject.org\*

管理节点安装 ceph-deploy

```shell
yum install ceph-deploy  python-setuptools python2-subprocess32 ceph-common
```

管理节点以 cephadm 用户身份在家目录建立 ceph-cluster 目录

```
su - cephadm
mkdir ceph-cluster
```

切换至 ceph-cluster 目录

```shell
cd ceph-cluster
```

## Mon 安装

在管理节点以 cephadm 用户运行

```shell
ceph-deploy new ceph-mon1 --cluster-network 192.168.100.0/24  --public-network 192.168.200.0/24
#mon节点，可以写第一个，也可以写多个
```

然后在所有节点——mon、mgr、osd 都要安装，一定要把源的地址改成阿里云的，不然卡半天，翻墙都没用。

```
sudo yum install ceph ceph-radosgw -y
```

在管理节点以 cephadm 用户运行

````shell
cd ceph-cluster
ceph-deploy install --no-adjust-repos node1 node2 node3

在管理节点以cephadm用户运行

```shell
ceph-deploy mon create-initial
#这一步其实是在生成keyring文件
````

在管理节点以 cephadm 用户运行

```shell
ceph-deploy admin ceph-mon1 ceph-mon2 ceph-mon3 ceph-osd4
#将配置和client.admin秘钥环推送到远程主机。
#每次更改ceph的配置文件，都可以用这个命令推送到所有节点上
```

在所有节点以 root 的身份运行

```shell
setfacl -m u:cephadm:r /etc/ceph/ceph.client.admin.keyring
#ceph.client.admin.keyring文件是 ceph命令行 所需要使用的keyring文件
#不管哪个节点，只要需要使用cephadm用户执行命令行工具，这个文件就必须要让cephadm用户拥有访问权限，就必须执行这一步
#这一步如果不做，ceph命令是无法在非sudo环境执行的。
```

## Mgr 部署

```shell
su - cephadm
cd ceph-cluster
ceph-deploy mgr create ceph-mon1
```

执行完成之后，在管理节点查看集群的健康状态，不过，这一步同样需要/etc/ceph/ceph.client.admin.keyring 文件

```shell
su - cephadm
sudo cp ceph-cluster/{ceph.client.admin.keyring,ceph.conf} /etc/ceph/
ceph -s
```

### mgr dashboard 启用

1. 开启 mgr 功能 `ceph mgr module enable dashboard`
2. 创建证书 `ceph dashboard create-self-signed-cert`
3. 创建用户和登录密码 `ceph dashboard set-login-credentials admin admin`
4. 查看服务访问方式 `ceph mgr services`
5. 在/etc/ceph/ceph.conf 中添加
   [mgr]
   mgr modules = dashboard
6. 设置 dashboard 的 ip 和端口

```
ceph config-key put mgr/dashboard/server_addr  172.18.63.111
ceph config-key put mgr/dashboard/server_port 7000
```

7. 在浏览器中访问即可

# 使用

## pool 的使用

### pool 的含义

Pool 是 ceph 中的存储池的盖南，要想使用 ceph 的存储功能，必须先创建存储池；是存储对象的逻辑分区，它规定了数据冗余的类型和对应的副本分布策略；支持两种类型：副本（replicated）和 纠删码（ Erasure Code）。

对于 pool、osd、pg 的理解大致如下：
osd 是管理物理磁盘的，而 pg 是一个放置策略组，相同 pg 内的对象会放置在相同的磁盘上，务端数据均衡和恢复的最小粒度就是 pg。

- 一个 pool 里可以有多个 pg
- 一个 pg 包含多个对象，一个对象只能属于一个 pg
- pg 有主从之分，一个 pg 分布在不同的 osd 上（针对多副本）

### pool 的创建

`ceph osd pool create mypool 64 64` 此命令的含义是创建了一个名为 mypool，pg 数量是 64 的存储池，的 完整的创建命令如下：
`osd pool create <poolname> <int[0-]> {<int[0-]>} {replicated|erasure} {<erasure_code_profile>} {<rule>} {<int>}`
`ceph osd pool application enable mypool rbd` 根据pool的使用不同需要分配不同的application,完整命令集含义如下
`osd pool application enable <poolname> <app> {--yes-i-really-mean-it}    enable use of an application <app> [cephfs,rbd,rgw] on pool <poolname>`

<!-- TODO 待完善内容 -->

## cephFS 的使用

1. 首先在 admin 节点上创建 mds 服务`ceph-deploy mds create node1 node2 node3`
2. 创建存储池`ceph osd pool create fs_data 32`
3. 创建元数据池`ceph osd pool create fs_metadata 32`
4. 创建名为 fs 的文件系统`ceph fs new fs fs_data fs_metadata`
5. 使用客户端挂载文件系统

- 安装 ceph-fuse `yum install ceph-fuse`
- 将 ceph.conf 和 ceph.client.admin.keyring 文件拷贝到客户端机器的/etc/ceph/目录下
- 在客户端目录创建测试文件夹`mkdir /test`
- 使用命令连接 cephfs `ceph-fuse -m 172.18.63.111,172.18.63.112,172.18.63.113:6789 /test`
- 拷贝文件至test文件夹，在另一台客户端机器上查看是否有此文件，目前cephfs的默认是一个，可使用命令建立多个。

## rdb的使用
块设备是ceph中使用最多的一种方式，下面讲下怎么使用
1. 创建pool存储池
2. 创建images对象`rbd create mypool/image --image-feature layering --size 10G`
3. 镜像伸缩容`rbd resize --size 15G mypool/image`，此操作后还需要在挂载的磁盘上进行扩容的操作
4. 删除镜像`rbd rm mypool/image`
5. 客户端挂载镜像
* 安装客户端`yum install ceph-common`
* 执行挂载镜像命令`rbd map mypool/image`
* 查看磁盘`lsblk`或`rbd showmapped`
* 上述命令的作用就像是给机器新加了一块磁盘，因此需要格式化、挂载后才能使用
6. 镜像添加快照`rbd snap create mypool/image --snap image-sn1`
7. 查看镜像快照`rbd snap ls mypool/image`
8. 删除镜像快照`rbd snap remove`
9. 还原镜像快照`rbd snap rollback`,需要重新挂载文件系统

## rgw对象存储的使用

1. 安装对象存储网关`ceph-deploy install --rgw node1`
2. 创建rgw实例`ceph-deploy  [--overwrite-conf] rgw create node1`，--overwrite可选，如果提示配置文件冲突，可添加此参数
3. 成功后可查看到在7480端口上可以访问
4. 创建有权限的用户`radosgw-admin user create --uid="admin" --display-name="admin"`,保留access_key和secret_key
"access_key": "EVK7AQ36LND5L8C11C2R",
"secret_key": "xXiBZn8v7plELwmAghMd9UZjjTAwNzoGeNtYiEfc"
5. 使用s3客户端进行访问测试，配置好.s3cfg文件，修改host_base 和host_bucket 的内容
6. s3cmd 创建bucket并上传文件  `s3cmd mb s3://test` `s3cmd put ceph.conf s3://test`

### mgr dashboard中的 Object Gateway 配置
可以使用上面的admin来连接，也可新建用户进行连接
具体配置信息可参考[官网文档](https://docs.ceph.com/docs/mimic/mgr/dashboard/#enabling-the-object-gateway-management-frontend)，此处不做解释
```
radosgw-admin user create --uid=system --display-name=system  --system
ceph dashboard set-rgw-api-access-key VH9L1XY18PKEX7AECPSN
ceph dashboard set-rgw-api-secret-key TRexoIvxMEFKvsEmBMykgfY0XeGQEBxM2rg8 
ceph dashboard set-rgw-api-host 172.18.63.111
ceph dashboard set-rgw-api-port 7480
ceph dashboard set-rgw-api-scheme http
ceph dashboard set-rgw-api-admin-resource all
ceph dashboard set-rgw-api-user-id system
ceph dashboard set-rgw-api-ssl-verify False
# 需要重启mgr dashboard
ceph mgr module disable dashboard
ceph mgr module enable dashboard
```

## k8s集群接入ceph
使用mypool这个pool作为自动配置的pvc
1. 创建用于k8s集群连接secret，可新建用户也可使用目前的admin用户
* 将此用户的密码进行base64加密`ceph auth get-key client.admin | base64`
* 编写secret.yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: storage-secret
  namespace: default
data:
  key: QVFEOHc3aGVVZGRyS1JBQXVEUWxhK1RnTFYvbmVQbjVwM0I3SXc9PQ==
type:
  kubernetes.io/rbd
```
2. 编写storageclass用于pvc的动态创建
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph
  namespace: default
  annotations:
   storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/rbd
reclaimPolicy: Retain
parameters:
  monitors: 172.18.63.111:6789
  adminId: admin
  adminSecretName: storage-secret
  adminSecretNamespace: default
  pool: mypool
  fsType: xfs
  userId: admin
  userSecretName: storage-secret
  imageFormat: "2"
  imageFeatures: "layering"
```
3. 测试ceph的使用
随便建一个pvc的连接，查看是否可以成功bound，以及ceph对应的pool中是否有此image


# rook-ceph简介
- Rook官网：https://rook.io
- Rook是[云原生计算基金会](https://www.cncf.io/)(CNCF)的孵化级项目.
- Rook是Kubernetes的开源**云本地存储协调**器，为各种存储解决方案提供平台，框架和支持，以便与云原生环境本地集成。
- 至于CEPH，官网在这：https://ceph.com/
- ceph官方提供的helm部署，至今我没成功过，所以转向使用rook提供的方案
- 官方指导手册https://rook.io/docs/rook/v1.1/ceph-examples.html
 

## 环境

```
centos 7.5
kernel 4.18.7-1.el7.elrepo.x86_64

docker 18.06

kubernetes v1.12.2
    kubeadm部署：
        网络: canal
        DNS: coredns
    集群成员：    
    192.168.1.1 kube-master
    192.168.1.2 kube-node1
    192.168.1.3 kube-node2
    192.168.1.4 kube-node3
    192.168.1.5 kube-node4

所有node节点准备一块200G的磁盘：/dev/sdb

```

---

## 准备工作
- 所有节点开启ip_forward

```
cat <<EOF >  /etc/sysctl.d/ceph.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

```

## 开始部署Operator
- 部署Rook Operator

```
#无另外说明，全部操作都在master操作

cd $HOME
git clone https://github.com/rook/rook.git

cd rook
cd cluster/examples/kubernetes/ceph
kubectl apply -f operator.yaml

```

- 查看Operator的状态

```
#执行apply之后稍等一会。
#operator会在集群内的每个主机创建两个pod:rook-discover,rook-ceph-agent

kubectl -n rook-ceph-system get pod -o wide

```

## 给节点打标签

- 运行ceph-mon的节点打上：ceph-mon=enabled

```
kubectl label nodes {kube-node1,kube-node2,kube-node3} ceph-mon=enabled

```

- 运行ceph-osd的节点，也就是存储节点，打上：ceph-osd=enabled

```
kubectl label nodes {kube-node1,kube-node2,kube-node3} ceph-osd=enabled

```

- 运行ceph-mgr的节点，打上：ceph-mgr=enabled

```
#mgr只能支持一个节点运行，这是ceph跑k8s里的局限
kubectl label nodes kube-node1 ceph-mgr=enabled

```

---

## 配置cluster.yaml文件

- 官方配置文件详解：https://rook.io/docs/rook/v0.8/ceph-cluster-crd.html

- 文件中有几个地方要注意：
  - **dataDirHostPath**: 这个路径是会在宿主机上生成的，保存的是ceph的相关的配置文件，再重新生成集群的时候要确保这个目录为空，否则mon会无法启动
  - **useAllDevices**: 使用所有的设备，建议为false，否则会把宿主机所有可用的磁盘都干掉
  - **useAllNodes**：使用所有的node节点，建议为false，肯定不会用k8s集群内的所有node来搭建ceph的
  - **databaseSizeMB和journalSizeMB**：当磁盘大于100G的时候，就注释这俩项就行了

- 本次实验用到的 cluster.yaml 文件内容如下：

```
apiVersion: v1
kind: Namespace
metadata:
  name: rook-ceph
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rook-ceph-cluster
  namespace: rook-ceph
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rook-ceph-cluster
  namespace: rook-ceph
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: [ "get", "list", "watch", "create", "update", "delete" ]
---
# Allow the operator to create resources in this cluster's namespace
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rook-ceph-cluster-mgmt
  namespace: rook-ceph
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: rook-ceph-cluster-mgmt
subjects:
- kind: ServiceAccount
  name: rook-ceph-system
  namespace: rook-ceph-system
---
# Allow the pods in this namespace to work with configmaps
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rook-ceph-cluster
  namespace: rook-ceph
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rook-ceph-cluster
subjects:
- kind: ServiceAccount
  name: rook-ceph-cluster
  namespace: rook-ceph
---
apiVersion: ceph.rook.io/v1beta1
kind: Cluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    # The container image used to launch the Ceph daemon pods (mon, mgr, osd, mds, rgw).
    # v12 is luminous, v13 is mimic, and v14 is nautilus.
    # RECOMMENDATION: In production, use a specific version tag instead of the general v13 flag, which pulls the latest release and could result in different
    # versions running within the cluster. See tags available at https://hub.docker.com/r/ceph/ceph/tags/.
    image: ceph/ceph:v13
    # Whether to allow unsupported versions of Ceph. Currently only luminous and mimic are supported.
    # After nautilus is released, Rook will be updated to support nautilus.
    # Do not set to true in production.
    allowUnsupported: false
  # The path on the host where configuration files will be persisted. If not specified, a kubernetes emptyDir will be created (not recommended).
  # Important: if you reinstall the cluster, make sure you delete this directory from each host or else the mons will fail to start on the new cluster.
  # In Minikube, the '/data' directory is configured to persist across reboots. Use "/data/rook" in Minikube environment.
  dataDirHostPath: /var/lib/rook
  # The service account under which to run the daemon pods in this cluster if the default account is not sufficient (OSDs)
  serviceAccount: rook-ceph-cluster
  # set the amount of mons to be started
  # count可以定义ceph-mon运行的数量，这里默认三个就行了
  mon:
    count: 3
    allowMultiplePerNode: true
  # enable the ceph dashboard for viewing cluster status
  # 开启ceph资源面板
  dashboard:
    enabled: true
    # serve the dashboard under a subpath (useful when you are accessing the dashboard via a reverse proxy)
    # urlPrefix: /ceph-dashboard
  network:
    # toggle to use hostNetwork
    # 使用宿主机的网络进行通讯
    # 使用宿主机的网络貌似可以让集群外的主机挂载ceph
    # 但是我没试过，有兴趣的兄弟可以试试改成true
    # 反正这里只是集群内用，我就不改了
    hostNetwork: false
  # To control where various services will be scheduled by kubernetes, use the placement configuration sections below.
  # The example under 'all' would have all services scheduled on kubernetes nodes labeled with 'role=storage-node' and
  # tolerate taints with a key of 'storage-node'.
  placement:
#    all:
#      nodeAffinity:
#        requiredDuringSchedulingIgnoredDuringExecution:
#          nodeSelectorTerms:
#          - matchExpressions:
#            - key: role
#              operator: In
#              values:
#              - storage-node
#      podAffinity:
#      podAntiAffinity:
#      tolerations:
#      - key: storage-node
#        operator: Exists
# The above placement information can also be specified for mon, osd, and mgr components
#    mon:
#    osd:
#    mgr:
# nodeAffinity：通过选择标签的方式，可以限制pod被调度到特定的节点上
# 建议限制一下，为了让这几个pod不乱跑
    mon:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: ceph-mon
              operator: In
              values:
              - enabled
    osd:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: ceph-osd
              operator: In
              values:
              - enabled
    mgr:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: ceph-mgr
              operator: In
              values:
              - enabled
  resources:
# The requests and limits set here, allow the mgr pod to use half of one CPU core and 1 gigabyte of memory
#    mgr:
#      limits:
#        cpu: "500m"
#        memory: "1024Mi"
#      requests:
#        cpu: "500m"
#        memory: "1024Mi"
# The above example requests/limits can also be added to the mon and osd components
#    mon:
#    osd:
  storage: # cluster level storage configuration and selection
    useAllNodes: false
    useAllDevices: false
    deviceFilter:
    location:
    config:
      # The default and recommended storeType is dynamically set to bluestore for devices and filestore for directories.
      # Set the storeType explicitly only if it is required not to use the default.
      # storeType: bluestore
      # databaseSizeMB: "1024" # this value can be removed for environments with normal sized disks (100 GB or larger)
      # journalSizeMB: "1024"  # this value can be removed for environments with normal sized disks (20 GB or larger)
# Cluster level list of directories to use for storage. These values will be set for all nodes that have no `directories` set.
#    directories:
#    - path: /rook/storage-dir
# Individual nodes and their config can be specified as well, but 'useAllNodes' above must be set to false. Then, only the named
# nodes below will be used as storage resources.  Each node's 'name' field should match their 'kubernetes.io/hostname' label.
#建议磁盘配置方式如下：
#name: 选择一个节点，节点名字为kubernetes.io/hostname的标签，也就是kubectl get nodes看到的名字
#devices: 选择磁盘设置为OSD
# - name: "sdb":将/dev/sdb设置为osd
    nodes:
    - name: "kube-node1"
      devices:
      - name: "sdb"
    - name: "kube-node2"
      devices:
      - name: "sdb"
    - name: "kube-node3"
      devices:
      - name: "sdb"

#      directories: # specific directories to use for storage can be specified for each node
#      - path: "/rook/storage-dir"
#      resources:
#        limits:
#          cpu: "500m"
#          memory: "1024Mi"
#        requests:
#          cpu: "500m"
#          memory: "1024Mi"
#    - name: "172.17.4.201"
#      devices: # specific devices to use for storage can be specified for each node
#      - name: "sdb"
#      - name: "sdc"
#      config: # configuration can be specified at the node level which overrides the cluster level config
#        storeType: filestore
#    - name: "172.17.4.301"
#      deviceFilter: "^sd."

```

---

## 开始部署ceph

- 部署ceph

```
kubectl apply -f cluster.yaml

# cluster会在rook-ceph这个namesapce创建资源
# 盯着这个namesapce的pod你就会发现，它在按照顺序创建Pod

kubectl -n rook-ceph get pod -o wide  -w

# 看到所有的pod都Running就行了
# 注意看一下pod分布的宿主机，跟我们打标签的主机是一致的

kubectl -n rook-ceph get pod -o wide

```

- 切换到其他主机看一下磁盘

  - 切换到kube-node1

  ```
  lsblk

  ```

  - 切换到kube-node3

  ```
  lsblk

  ```
  
---

## 配置ceph dashboard

- 看一眼dashboard在哪个service上

```
kubectl -n rook-ceph get service
#可以看到dashboard监听了8443端口
```

- 创建个nodeport类型的service以便集群外部访问

```
kubectl apply -f dashboard-external-https.yaml

# 查看一下nodeport在哪个端口
ss -tanl
kubectl -n rook-ceph get service

```

- 找出Dashboard的登陆账号和密码

```
MGR_POD=`kubectl get pod -n rook-ceph | grep mgr | awk '{print $1}'`

kubectl -n rook-ceph logs $MGR_POD | grep password

```

- 打开浏览器输入任意一个Node的IP+nodeport端口
- 这里我的就是：https://192.168.1.2:30290



## 配置ceph为storageclass

- 官方给了一个样本文件：storageclass.yaml
- 这个文件使用的是 **RBD 块存储**
- pool创建详解：https://rook.io/docs/rook/v0.8/ceph-pool-crd.html

```
apiVersion: ceph.rook.io/v1beta1
kind: Pool
metadata:
  #这个name就是创建成ceph pool之后的pool名字
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 1
  # size 池中数据的副本数,1就是不保存任何副本
  failureDomain: osd
  #  failureDomain：数据块的故障域，
  #  值为host时，每个数据块将放置在不同的主机上
  #  值为osd时，每个数据块将放置在不同的osd上
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: ceph
   # StorageClass的名字，pvc调用时填的名字
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  # Specify the namespace of the rook cluster from which to create volumes.
  # If not specified, it will use `rook` as the default namespace of the cluster.
  # This is also the namespace where the cluster will be
  clusterNamespace: rook-ceph
  # Specify the filesystem type of the volume. If not specified, it will use `ext4`.
  fstype: xfs
# 设置回收策略默认为：Retain
reclaimPolicy: Retain


```

- 创建StorageClass

```
kubectl apply -f storageclass.yaml
kubectl get storageclasses.storage.k8s.io  -n rook-ceph
kubectl describe storageclasses.storage.k8s.io  -n rook-ceph

```


---

- 创建个nginx pod尝试挂载

```
cat << EOF > nginx.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph


---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports: 
  - port: 80
    name: nginx-port
    targetPort: 80
    protocol: TCP

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /html
          name: http-file
      volumes:
      - name: http-file
        persistentVolumeClaim:
          claimName: nginx-pvc
EOF

kubectl apply -f nginx.yaml


```

- 查看pv,pvc是否创建了

```
kubectl get pv,pvc


# 看一下nginx这个pod也运行了
kubectl get pod

```

- 删除这个pod,看pv是否还存在

```
kubectl delete -f nginx.yaml

kubectl get pv,pvc
# 可以看到，pod和pvc都已经被删除了，但是pv还在！！！
```


--- 

## 添加新的OSD进入集群

- 这次我们要把node4添加进集群，先打标签

```
kubectl label nodes kube-node4 ceph-osd=enabled

```

- 重新编辑cluster.yaml文件

```
# 原来的基础上添加node4的信息

cd $HOME/rook/cluster/examples/kubernetes/ceph/
vi cluster.yam

```

- apply一下cluster.yaml文件

```
kubectl apply -f cluster.yaml

# 盯着rook-ceph名称空间,集群会自动添加node4进来

kubectl -n rook-ceph get pod -o wide -w
kubectl -n rook-ceph get pod -o wide
```

- 去node4节点看一下磁盘

```
lsblk
```

- 再打开dashboard看一眼




## 删除一个节点
- 去掉node3的标签

```
kubectl label nodes kube-node3 ceph-osd-

```

- 重新编辑cluster.yaml文件

```
# 删除node3的信息

cd $HOME/rook/cluster/examples/kubernetes/ceph/
vi cluster.yam

```

- apply一下cluster.yaml文件

```
kubectl apply -f cluster.yaml

# 盯着rook-ceph名称空间

kubectl -n rook-ceph get pod -o wide -w
kubectl -n rook-ceph get pod -o wide


# 最后记得删除宿主机的/var/lib/rook文件夹
```


## 常见问题
- 官方解答：https://rook.io/docs/rook/v0.8/common-issues.html

- **当机器重启之后，osd无法正常的Running，无限重启**

```
#解决办法：

# 标记节点为 drain 状态
kubectl drain <node-name> --ignore-daemonsets --delete-local-data

# 然后再恢复
kubectl uncordon <node-name>

```