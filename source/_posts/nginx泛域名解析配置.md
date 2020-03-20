---
title: nginx泛域名解析
date: 2019/06/28
tags: 
- nginx
- https
categories: 运维
comments: 
description: 由于公司业务，目前需要配置全站的https，但由于子站较多，包括各个地市站点，所以想到了nginx的泛域名解析,而且之前的nginx的配置文件全部都写到了一起，基于种种，修改了nginx的目前的配置方式
top_img: 
---

## nginx泛域名解析配置
一种配置的nginx的子域名的方式是有网关的方式，就是让所有的接口请求走总站的url路径映射为子域名，如 api.jstec.com 就是由 jstec.com/api 映射而来，还有一种方式是直接由子域名访问的方式。由于之前就存在了很多的子域名，所以此处就沿用之前的子域名即可，即使用第二种的方式。

一般企业的网站的访问都是由dns解析到指定的公网地址上的，再由公网通过防火墙连接到内网的nginx作为后端服务的代理服务。所以大致的访问流程大致如下：

dns-->公网地址-->私网地址-->后端服务

## nginx server的访问优先级
   * 完全匹配  jstec.com.cn 这种可拆分为文件
   * 通配符匹配  *.jstec.com.cn 用于全局匹配如重定向全站80端口
   * 正则表达式  子域名匹配 ~^(?<subdomain>.+).jstec.com.cn; 匹配子域名，优先级最低

## 详细配置文件
1. 内部有http链接，需要去除
```
   server {
    #  全局301 80重定向到443
    listen 80;
    server_name *.jstec.com.cn;
    return 301 https://$host$request_uri;
  }
```
2. 泛域名解析使用正则匹配的方式进行子域名的匹配，但是请求时的url为的子域名的url，会有的重复的，在日志信息中加上$host参数查看请求的域名
   
```
 server {
    # listen 80;
    listen 443 ssl http2;
    listen [::]:443 ssl http2; 
    ssl_certificate jstec.com.cn.crt;
    ssl_certificate_key jstec.com.cn.pem;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers AESGCM:ALL:!DH:!EXPORT:!RC4:+HIGH:!MEDIUM:!LOW:!aNULL:!eNULL;

    server_name ~^(?<subdomain>.+).jstec.com.cn;
    # 可不用重新写规则，但是日志中的request无法找到正确的请求url
    # 解决办法： 在日志文件中加入host域名串
    location /$subdomain {
        proxy_set_header        X-Real-IP $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header        Host $http_host;
        proxy_set_header        X-NginX-Proxy true;
        proxy_pass              http:/$subdomain.jstec.com.cn;
    }
  }
```
3. 添加conf.d文件夹，将子域名进行分拆
```
# upstream内容忽略
server {
    listen  443;
    server_name   monitor.jstec.com.cn;
    location / {
      proxy_pass http://screen_ui;
    }
    location /api {
      rewrite ^/api/(.*)$ /$1 break;
      proxy_pass http://screen_service;
    }
    location /file/ {
      proxy_pass http://screen_service;
    }
    location /bss {
      proxy_pass http://screen_bss;
    }
  }
```