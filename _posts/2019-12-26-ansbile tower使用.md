---
layout:    post
title:    "ansible tower使用指南 "
subtitle:    "Inventory动态数据源的几种方式"
date:  2019-12-26 16:47:43.000000000 +09:00
author:    "dyb"
header-img:  "img/posts/20191226180555.png"
header-mask:  0.3
catalog:  true
tags:
- ansible
---
> ansible想必大家都是了解的，目前也是用的较多的linux服务器集群管理工具，此处介绍的是它的web端产品ansible-tower（ansible-awx），来自RedHat的一款商业产品；好了，介绍到此为止。

## 使用自带的hosts添加主机
首先，新建INVENTORIES，在INVENTORIES中点击hosts，点击添加hosts即可;在选中某个hosts，可直接运行测试命令如PING。
![20191226194455.png](https://raw.githubusercontent.com/diaoyongbao/diaoyongbao.github.io/master/img/posts/20191226194455.png)
## 使用gitlab仓库中的hosts文件
同样在INVENTORIES的界面下，此处选择sources这个选项，在SOURCE源中，选择Sourced from a Project,Project来自git仓库;在此仓库中保存着hosts文件。
```
172.18.63.63
172.18.63.64
172.18.63.65
```
配置及执行结果如下
![20191226200129.png](https://raw.githubusercontent.com/diaoyongbao/diaoyongbao.github.io/master/img/posts/20191226200129.png)
## 使用代码动态生成hosts和groups信息
其实这一节才是此文的核心内容，上面的都是引子，同样使用SOURCE源，其实里面是有很多选项的，如AWS EC2、GCE、openstack等，由于公司的云环境是深信服的，所以此处选用Custom Script 自定义代码来实现获取groups、hosts。
核心内容为[开发动态的Inventory数据源](https://ansible-tran.readthedocs.io/en/latest/docs/developing_inventory.html),在ansible的参考文章中给我们提供了获取方式及json格式。json格式需要严格按照如下定义：
```
--host
# 简单定义
{'172.18.61.107': {}, '172.18.62.113': {}}
# 增加参数
{
        "10.5.189.180":
            {"ansible_ssh_host": "10.5.189.180", "ansible_ssh_port": 22, "ansible_ssh_user": "root",
             "ansible_ssh_port"",ansible_ssh_pass": "xxxx"},
        "10.5.189.181":
            {"ansible_ssh_host": "10.5.189.181", "ansible_ssh_port": 22, "ansible_ssh_user": "root",
             "ansible_ssh_port"",ansible_ssh_pass": "xxxx"}
    }

--list
# 简单定义
{"基础服务": ["172.18.64.200"], "k8s": ["172.18.63.102", "172.18.63.105", "172.18.63.79"]}
# 增加子分组和参数
 {
    "测试":{"hosts": ["10.5.189.181", "10.5.189.180"],"vars":{"ansible_ssh_user":"root","ansible_ssh_port": 22},"children":["ddz"]},
    "分组1":["10.5.189.180"],
    "ddz":["10.5.189.182"]
    }
```
此处提供python代码如下
```
#!/usr/bin/python
# coding:utf8
import json
import sys

def group():
    info_dict = {
    "测试":{"hosts": ["10.5.189.181", "10.5.189.180"],"vars":{"ansible_ssh_user":"root","ansible_ssh_port": 22},"children":["ddz"]},
    "分组1":["10.5.189.180"],
    "ddz":["10.5.189.182"]
    }
    print(json.dumps(info_dict, indent=4))
def host(ip):
    info_dict = {
        "10.5.189.180":
            {"ansible_ssh_host": "10.5.189.180", "ansible_ssh_port": 22, "ansible_ssh_user": "root",
             "ansible_ssh_port"",ansible_ssh_pass": "xxxx"},
        "10.5.189.181":
            {"ansible_ssh_host": "10.5.189.181", "ansible_ssh_port": 22, "ansible_ssh_user": "root",
             "ansible_ssh_port"",ansible_ssh_pass": "xxxx"}
    }
    print(json.dumps(info_dict[ip], indent=4))


if len(sys.argv) == 2 and (sys.argv[1] == '--list'):
    group()
elif len(sys.argv) == 3 and (sys.argv[1] == '--host'):
    host(sys.argv[2])
else:
    print("Usage: %s --list or --host <hostname>" % sys.argv[0])
    sys.exit(1)
```
爬虫代码获取示例：
```
#!/usr/bin/python
# coding:utf8
import json
import sys
import requests
reload(sys)  
sys.setdefaultencoding('utf8') 
requests.packages.urllib3.disable_warnings()
headers = {
    'content-type':'application/json;charset=UTF-8',
    'User-Agent':'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.108 Safari/537.36'
}

def get_cookie():
    url = 'https://172.18.64.223/vapi/extjs/access/ticket'
    request_data = {
        'username':'',
        'password':''
    }
    r = requests.post(url,data=request_data,headers=headers,verify=False)
    r_results=json.dumps(r.json())

    return json.loads(r_results)["data"]["ticket"]
def get_vms_ip_list():
    url='https://172.18.64.223/vapi/json/cluster/get_vms_ip_list'
    headers['Cookie']='LoginAuthCookie='+get_cookie()
    r = requests.get(url,verify=False,headers=headers)
    r_results=json.dumps(r.json())
    data = json.loads(r_results)["data"]
    for hostname,data_values in data.items():
        for vmid,vmid_values in data_values.items():
            for ip in vmid_values['net0'].keys():
                yield [hostname,vmid,ip]
def get_cluster():
    url='https://172.18.64.223/vapi/extjs/cluster/vms?group_type=group&sort_type=&desc=1'

    headers['Cookie']='LoginAuthCookie='+get_cookie()
    r = requests.get(url,verify=False,headers=headers)
    r_results=json.dumps(r.json())

    return r_results
def hosts():
    ip_dict = {}
    for ip_list in  get_vms_ip_list():
        ip = ip_list[2]
        ip_dict[ip]={}
    return ip_dict
def host(ip):
    info_dict = hosts
    print(json.dumps(info_dict[ip], indent=4))

def getitem(l):
    for item in l:
        if isinstance(item,list):
            getitem(item)
        else:
            # yield item
            if isinstance(item,dict):
                if 'data' in item.keys():
                    if item['data']!= []:
                        for i in item['data']:
                            yield i
                else: 
                    yield item

def groups():
    group = {}
    data = json.loads(get_cluster())
    ip_dict={}
    for ip_list in  get_vms_ip_list():
        host = ip_list[0]
        vmid  = ip_list[1]
        ip = ip_list[2]
        ip_dict[vmid]=ip

    for i in range(len(data['data'])):
        groups=data['data'][i]
        ips = []  #存储ip列表
        group_map=[]  #映射hosts和groupname 的对应关系
        # for key,values in groups.items():
        all_data = getitem(groups['data'])
        for a in all_data:
            ip_1 = []   
            if a['groupname'] in group.keys():
                if 'running' in a.keys() and a['running']==1 and a['label']!="264566dae68d":
                    for vmid in ip_dict.keys():
                        if vmid==str(a['vmid']):
                            ips[group_map.index(a['groupname'])].append(ip_dict[vmid])
                            group[a['groupname']] = ips[group_map.index(a['groupname'])]
            else:
                if 'running' in a.keys() and a['running']==1 and a['label']!="264566dae68d":
                    for vmid in ip_dict.keys():
                        if vmid==str(a['vmid']):
                            group_map.append(a['groupname'])
                            ip_1.append(ip_dict[vmid])
                            ips.append(ip_1)
                            group[a['groupname']] = ips[group_map.index(a['groupname'])]
    return group

def group():
    info_dict = groups()
    print(json.dumps(info_dict,indent=4,ensure_ascii=False))


if len(sys.argv) == 2 and (sys.argv[1] == '--list'):
    group()
elif len(sys.argv) == 3 and (sys.argv[1] == '--host'):
    host(sys.argv[2])
else:
    print("Usage: %s --list or --host <hostname>" % sys.argv[0])
    sys.exit(1)

```