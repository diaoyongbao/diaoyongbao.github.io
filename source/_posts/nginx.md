---
title: nginx的一些用法
date: 2020/05/05
tags:
  - nginx
  - http
categories: 运维
comments:
description: 在运维过程中nginx的一些设置的地方的记录。
top_img: https://images.pexels.com/photos/1152707/pexels-photo-1152707.jpeg?auto=compress&cs=tinysrgb&h=750&w=1260
---
# NGINX

# nginx 原理篇

内容来自[当初我要是这么学习 Nginx 就好了](https://mp.weixin.qq.com/s?__biz=MjM5ODI5Njc2MA==&mid=2655830056&idx=1&sn=fbd5a1968f861c3ecf831b453f1322c6&chksm=bd748dff8a0304e965ee8bf57e5b815841aee885fef515a0b99661e9298a0c71f97a882401ce&mpshare=1&scene=1&srcid=&sharer_sharetime=1584100159037&sharer_shareid=a9a2d735afa4ee3e3ddb53b3132db945#rd)

## nginx 简介及特点

Nginx (engine x) 是一个高性能的 Web 服务器和反向代理服务器，也是一个 IMAP/POP3/SMTP 服务器：

- 它由俄罗斯程序员 Igor Sysoev 于 2002 年开始开发。
- Nginx 是增长最快的 Web 服务器，市场份额已达 33.3％。
- 全球使用量排名第二，2011 年成立商业公司。

Nginx 社区分支：

- Openresty：作者 @agentzh（章宜春）开发的，最大特点是引入了 ngx_lua 模块，支持使用 Lua 开发插件，并且集合了很多丰富的模块，以及 Lua 库。
- Tengine：主要是淘宝团队开发。特点是融入了因淘宝自身的一些业务带来的新功能。
- Nginx 官方版本，更新迭代比较快，并且提供免费版本和商业版本。

Nginx 源码结构（代码量大约 11 万行 C 代码）：

- 源代码目录结构 Core（主干和基础设置）
- Event（事件驱动模型和不同的 IO 复用模块）
- HTTP（HTTP 服务器和模块）
- OS（操作系统相关的实现）
- Mail（邮件代理服务器和模块）
- Misc（杂项）

Nginx 特点如下：

- 反向代理，负载均衡器
- 高可靠性、单 Master 多 Worker 模式
- 高可扩展性、高度模块化
- 非阻塞
- 事件驱动
- 低内存消耗
- 热部署

## nginx 应用场景

- Nginx 的应用场景如下：
- 静态文件服务器
- 反向代理，负载均衡
- 安全防御
- 智能路由（企业级灰度测试、地图 POI 一键切流）
- 灰度发布
- 静态化
- 消息推送
- 图片实时压缩
- 防盗链

# nginx 配置篇

## 配置文件的目录结构说明

一个常规的 Nginx 配置文件可分为如下几个段落：
| 配置段   | 说明                 |
| -------- | -------------------- |
| main     | 主配置区域，全局设置 |
| server   | 主机设置             |
| upstream | 上游服务             |
| location | URL 匹配             |
其中 server 继承 main，location 继承 server;upstream 既不会继承也不会被继承

### main 配置段说明

`user` nginx 服务的运行身份,通常为 root 或 nginx

`worker_processes` work 角色的工作进程的个数，通常可设置为 cpu 核数-1，如果开启了 ssl 和 gzip 可设置为 cpu 数的两倍，减少 io 操作；其中`auto`的值即为 cpu 核数。

`worker_cpu_affinity` work 进程的 cpu 亲和，在高并发场景下，通过设置 cpu 亲和性来降低由于多 cpu 核心切换造成的寄存器等现场重建带来的性能损耗。

`worker_rlimit_nofile` work 进程最大打开文件数

`worker_connections` 单个 worker 进程能并发处理（发起）的最大连接数（包含与客户端或后端被代理服务器间等所有连接数），写在 events 区段内，nginx 作为反向代理服务时，计算方式为“最大连接数=worker_processes\*worker_connections/4”，nginx 作为 http 服务时，将 4 换成 2，最大值不可超过`worker_rlimit_nofile`

`pid` 运行时的进程位置，在日志切割和 systemd 的启动服务中可用到.

### http 配置段说明

`sendfile on` 开启高效文件传输模式，sendfile 指令指定 nginx 是否调用 sendfile 函数来输出文件，减少用户空间到内核空间的上下文切换。对于普通应用设为 on，如果用来进行下载等应用磁盘 IO 重负载应用，可设置为 off，以平衡磁盘与网络 I/O 处理速度，降低系统的负载。

`keepalive_timeout 65` 长连接超时时间，单位是秒，这个参数很敏感，涉及浏览器的种类、后端服务器的超时设置、操作系统的设置，可以另外起一片文章了。长连接请求大量小文件的时候，可以减少重建连接的开销，但假如有大文件上传，65s 内没上传完成会导致失败。如果设置时间过长，用户又多，长时间保持连接会占用大量资源。

`access_log /var/log/nginx/access.log main;` nginx 日志文件存放位置及日志输出 Level，详情见日志处理。

`underscores_in_headers on` 跨域参数，默认为 off，表示如果 header name 中包含下划线，则忽略掉。

`limit_req_zone $binary_remote_addr zone=app_convert:10m rate=50r/m` 访问限制，区域名称为 app_convert(自定义)，占用空间大小为 10m，平均处理的请求频率不能超过每分钟 50 次，或‘1r/s’代表每秒一次；`$binary_remote_addr`是`$remote_addr`（客户端 IP）的二进制格式。

`real_ip_header X-Forwarded-For` 从哪个 header 头检索出要的 IP 地址

`real_ip_recursive on` 递归排除 IP 地址,ip 串从右到左开始排除 set_real_ip_from 里面出现的 IP,如果出现了未出现这些 ip 段的 IP，那么这个 IP 将被认为是用户的 IP

`gzip on` 开启压缩传输

`gzip_min_length`当返回内容大于此值时才会使用 gzip 进行压缩,以 K 为单位,当值为 0 时，所有页面都进行压缩

`gzip_buffers 4 32k` 设置 gzip 申请内存的大小,其作用是按块大小的倍数申请内存空间,4 代表指定 Nginx 服务器需要向服务器申请的缓存空间的个数，32k 代表空间大小，单位 k。

`gzip_types` 设置需要压缩的 MIME 类型,非设置值不进行压缩

`gzip_vary on` 和 http 头有关系，会在响应头加个 Vary: Accept-Encoding，可以让前端的缓存服务器缓存经过 gzip 压缩的页面，例如，用 Squid 缓存经过 Nginx 压缩的数据

`client_max_body_size` 允许客户端请求的最大单文件字节数。

`client_body_buffer_size`缓冲区代理缓冲用户端请求的最大字节数。

`add_header X-Frame-Options SAMEORIGIN` 使用 X-Frame-Options 配置，解决点击劫持。DENY,浏览器会拒绝当前页面加载任何 frame 页面;SAMEORIGIN,frame 页面的地址只能为同源域名下的页面；ALLOW-FROM,可以定义允许 frame 加载的页面地址。

`vhost_traffic_status_zone` nginx-module-vts 模块配置，开启状态统计功能。

`vhost_traffic_status_filter_by_host on` 在 Nginx 配置有多个 server_name 的情况下，会根据不同的 server_name 进行流量的统计，否则默认会把流量全部计算到第一个 server_name 上。

#### http_proxy 模块配置

`proxy_ignore_client_abort`客户端断网时，nginx 服务器是否终端对被代理服务器的请求。默认为 off

`proxy_connect_timeout` nginx 与后端服务器连接超时时间(代理连接超时)，这个超时不能超过 75s。

`proxy_send_timeout` 发送请求给后端服务的超时时间，超时设置不是为了整个发送期间，而是在两次 write 操作期间。如果超时后，upstream 没有收到新的数据，nginx 会关闭连接

`proxy_read_timeout`连接成功后，与后端服务器两个成功的响应操作之间超时时间(代理接收超时)，它决定了 nginx 会等待多长时间来获得请求的响应。这个时间不是获得整个 response 的时间，而是两次 reading 操作的时间。

`proxy_buffer_size` 设置代理服务器（nginx）从后端 realserver 读取并保存用户头信息的缓冲区大小，默认与 proxy_buffers 大小相同,可将此值设置的小一些

`proxy_buffers` proxy_buffers 缓冲区，nginx 针对单个连接缓存来自后端 realserver 的响应。

`proxy_busy_buffers_size` 高负荷下缓冲大小（proxy_buffers\*2）

`proxy_temp_file_write_size` 当缓存被代理的服务器响应到临时文件时，这个选项限制每次写临时文件的大小。proxy_temp_path（可以在编译的时候）指定写到哪那个目录

`proxy_intercept_errors` 当上游服务器响应头回来后，可以根据响应状态码的值进行拦截错误处理，与 error_page 指令相互结合。用在访问上游服务器出现错误的情况下.

`proxy_next_upstream`当其中一个后端服务返回错误码 404,500...等错误时，可以分配到下一台服务器程序继续处理，提高平台访问成功率

`proxy_redirect`对发送给客户端的 URL 进行修改

`proxy_store` 后端缓存

### server 配置段说明

`listen` 监听端口，默认 80，小于 1024 的端口需要以 root 启动

`server_name` 服务名，如 localhost，127.0.0.1 等，可使用域名或 IP，可正则匹配

`ssl_certificate` ssl 证书文件路径，crt 文件

`ssl_certificate_key` ssl 证书私钥路径，pem 文件

`ssl_session_timeout`客户端可以重用会话缓存中 ssl 参数的过期时间

`ssl_protocols`用于启动特定的加密协议

`ssl_ciphers` 加密套件

### location 配置段说明

`root /var/www/html` 定义服务器根目录地址

`index` 定义默认访问文件名，一般接 root。

`proxy_pass http:/backend` 反向代理后端服务

`rewrite` 详见[rewrite 的几种写法](#jump)

### upstream 配置段说明

`upstream name {server address [parameters]}` 结构

后端服务调度方式，默认轮询，可使用`weight`进行加权。round-robin & weighted

`ip_hash` 需要在 upstream 的配置中知道“ip_hash;”,每个请求按访问的 ip 的 hash 结果分配，这样每个客户端固定访问一个后端服务器，可以解决 session 的问题

`fair` 需要在 upstream 的配置中指定"fair;"，按后端服务器的响应时间来分配请求，响应时间短的优先分配。least-connected

`hash $clientRealIp` 自定义 hash，根据用户的源 IP 进行 hash 访问后端。

`max_fails` 最大重试次数

`fail_timeout` 重试间隔时间

# 日志处理

nginx 中的日志级别分为 debug|info|notice|warn|error|crit|alert|emerg
log_format 可生成自定义的日志结构。

## 日志切割

1. 安装 logrotate 服务
2. 配置 nginx 的日志切割配置文件/etc/logrotate.d/nginx

```
/var/log/nginx/*log {
    create 0644 nginx nginx
    daily
    rotate 10
    missingok
    notifempty
    compress
    sharedscripts
    postrotate
        /bin/kill -USR1 `cat /run/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

**定时任务加脚本的方式，供参考**

```
#!/bin/bash
year=`date +%Y`
month=`date +%m`
day=`date +%d`
logs_backup_path="/usr/local/nginx/logs_backup/$year$month"               #日志存储路径

logs_path="/usr/local/nginx/logs/"                                                             #要切割的日志路径
logs_access="access"                                                                            #要切割的日志
logs_error="error"
pid_path="/usr/local/nginx/logs/nginx.pid"                                                 #nginx的pid

[ -d $logs_backup_path ]||mkdir -p $logs_backup_path
rq=`date +%Y%m%d`
#mv ${logs_path}${logs_access}.log ${logs_backup_path}/${logs_access}_${rq}.log
mv ${logs_path}${logs_error}.log ${logs_backup_path}/${logs_error}_${rq}.log
kill -USR1 $(cat /usr/local/nginx/logs/nginx.pid)
```

## 日志入 ES

1. 安装 filebeat、logstash 服务
2. 修改 filebeat 配置文件/etc/filebeat/filebeat.yml

```
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - "/var/log/nginx.error.log"
  tags: ["nginx-error"]
  fields:
    logsource: nginx_error_log
    logtype: nginx
- type: log
  enabled: true
  paths:
    - "/var/log/nginx/access.log"
  tags: ["nginx-access"]
  fields:
     logsource: nginx_access_log
     logtype: nginx
```

3. 修改 logstash 配置文件

```
input {
  beats {
        port => "5044"
        }
}

filter {
  if [fields][logsource] == "nginx_access_log"{
        grok{
                match => {
                        "message" => [
                                '%{IPV4:client_ip} - (?:%{USERNAME}|-) \[%{HTTPDATE:[@metadata][timestamp]}\] "%{WORD:method} %{URIPATHPARAM:request:string} HTTP/%{NUMBER}" %{NUMBER:reponse_code} (?:%{NUMBER:bytes}|-) "(?:%{URI:http_referer}|-)" %{QS:user_agent} "(?:%{WORD:x_forwarded_for}|-)"%{NUMBER:connection_time} (?:%{HOSTPORT:upstream_addr}|-) \"(?:%{DATA:cookie}|-)\"'
                                ]
                        }
                remove_field => [ "message" ]
                }

         date { match => [ "[@metadate][timestamp]","dd/MMM/yyyy:HH:mm:ss Z" ]}
         mutate {
                convert => [
                        "response_code","integer",
                        "bytes","integer",
                        "response_time","float"
                        ]
               }
        geoip {
              source => "client_ip"
              target => "geoip"
              database => "/etc/logstash/GeoLite2-City.mmdb"
              }
        }
}

output{
  if [fields][logsource] == "nginx_access_log"{
                elasticsearch {
                        hosts => ["172.18.63.63:9200","172.18.63.64:9200","172.18.63.65:9200"]
                        index => "logstash-ngxaccess-%{+YYYYMMdd}"
                }
        }
  if [fields][logsource] == "nginx_error_log"{
                elasticsearch {
                        hosts => ["172.18.63.63:9200","172.18.63.64:9200","172.18.63.65:9200"]
                        index => "logstash-ngxerror-%{+YYYYMMdd}"
                }
        }
}
```

4. IP 地址说明，使用 GeoLite2-City 文件解析 ip 对应的经纬度值，index 索引须以 logstah- 开头，方可将 geoip.location 的 type 自动设定为"geo_point"；

# 其他

## 泛域名解析设置

见[nginx 服务优化](http://confluence.jwt.com/pages/viewpage.action?pageId=2031679)

## rewrite 的几种写法<span id="jump"></span>

语法： rewrite regex replacement [flag]
rewrite 的功能是使用 nginx 提供的全局变量或自己设置的变量，然后结合正则表达式和标志位实现 url 重写以及重定向。
rewrite 指令只能放在 server、location 或 if 中，并且只能对域名后边的除去传递的参数外的字符串起作用。

如果一个 URI 匹配了 rewrite 指令指定的正则表达式（regex），则 URI 就按照 replacement 进行重写，而 rewrite 按配置文件中出现的顺序执行。其中 flag 标志可以停止继续处理。
如果 replacement 以”http://”或”https://”开始，将不再继续处理，那么这个重定向将直接返回给客户端。
**flag** 标志位的几个参数和含义

| 参数      | 含义                                                                                                                                                       |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| last      | 完成该 rewrite 规则的执行后，停止处理后续 rewrite 指令集；然后查找匹配改变后 URI 的新 location；                                                           |
| break     | 完成该 rewrite 规则的执行后，停止处理后续 rewrite 指令集，并不再重新查找；但是当前 location 内剩余非 rewrite 语句和 location 外的的非 rewrite 语句可以执行 |
| redirect  | 返回 302 临时重定向，地址栏会显示跳转后的地址                                                                                                              |
| permanent | 返回 301 永久重定向，地址栏会显示跳转后的地址；即表示如果客户端不清理浏览器缓存，那么返回的结果将永久保存在客户端浏览器中了                                |

**目前系统配置中的几种 rewrite 写法说明**
rewrite ^/portal/(.\*)$ /$1 break;
此语句表示将 portal 匹配路径丢弃，直接对后面的连接内容进行访问。

rewrite ^/goto/(.\*) https://bcs.jstec.com.cn/$1 permanent;
此语句表示将/goto 后的内容拼接到后续链接并进行重定向访问

rewrite .\* http://$goto permanent;
此语句表示将此 location 中的所有链接重定向到其他的 location(goto)

rewrite ^/goto/(.\*) https://jstec.com.cn/$1 redirect;
临时重定向

return 301 https://$host$request_uri;
301 重定向的另一种写法

## yum 安装的 nginx 如何添加新的模块

以监控模块 nginx-module-vts 为例

首先下载与 yum 安装的同版本号的源码，使用`nginx -V`查看版本

编译此版本，同时加入需要添加的模块`--add-module=/opt/nginx-module-vts/`

将系统上的 nginx 二进制文件和配置文件进行备份，替换编译后的 nginx
将 objs 下的 nginx 替换 /usr/sbin/nginx 文件后使用`nginx -V` 查看是否有新的模块

## 状态监控模块

在 server 段添加如下内容

```
server {
    listen 80;
    server_name 127.0.0.1;
    location /status {
        vhost_traffic_status_display;
        vhost_traffic_status_display_format html;
    }
  }
```

使用 nginx-vts-exporter 将 status 的内容进行格式化，以便 prometheus 获取

启动命令`nohup /root/nginx-vts-exporter -nginx.scrape_timeout 10 -nginx.scrape_uri http://127.0.0.1/status/format/json &`
监控端口 9913

加入 consul
`curl -X PUT -d '{"id": "nginx-node","name": "nginx-node","address": "172.18.63.33","port": 9913,"tags": ["node"],"checks": [{"http": "http://172.18.63.33:9913/","interval": "10s"}]}' http://consul.jwt.com/v1/agent/service/register`

grafana 显示 使用[Nginx-Vts-Stats.json](http://gitlab.jwt.com/YW/grafana/blob/master/Nginx-Vts-Stats.json)

