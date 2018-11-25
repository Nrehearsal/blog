---
title: "使用unbound搭建DNS服务器，配置缓存以及防止DNS污染"
date: 2018-11-25T18:33:14+08:00
draft: false
---

{{% summary %}}
> 真理是时间之产物，而不是权威之产物 ——弗兰西斯·培根
{{% /summary %}}

使用unbound搭建DNS服务器，配置缓存以及防止DNS污染

[Unbound](https://nlnetlabs.nl/projects/unbound/about/) is a validating, recursive, caching DNS resolver. It is designed to be fast and lean and incorporates modern features based on open standards.

### 搭建环境
- ubuntu_server16.04

- 1Core, 1G

- 能够通过代理访问国外DNS服务器(8.8.8.8，1.1.1.1等)，因为如果直接通过国内路线访问上述DNS，某些域名会得到错误的ip地址。

### 安装unbound

- 源码安装：下载[unbound](https://nlnetlabs.nl/projects/unbound/download/)，更具官方[教程安装](https://nlnetlabs.nl/documentation/unbound/howto-setup/)

- 仓库安装:

    ```sh
    $ sudo apt-get update
    $ sudo apt-get install unbound
    ```
    
### 配置unbound

- unbound安装完成后会在/etc/unbound目录下生成相应的配置文件

- 配置基本功能，在/etc/unbound/unbound.conf文件中增加如下内容:

    ```
    server:
        verbosity: 1
        num-threads: 2 #线程数，设置与cpu核数相同，可提高unbound的cpu利用率
        interface: 0.0.0.0 #监听地址，如果只想某一网络访问，填写该对应ip即可
        interface: ::0 #ipv6配置，同上
        port: 53 #端口
        so-reuseport: yes  #为每个线程的传入查询打开专用侦听套接字。可以更均匀地将传入查询分布到线程
        cache-min-ttl: 60 #解析最小缓存时间
        cache-max-ttl: 600 #解析最大缓存时间
        outgoing-range: 8192  
        access-control: 192.168.1.0/24 allow  #访问控制（允许192.168.1段ip访问本机）
        access-control: 127.0.0.1/8 allow  #允许本机访问
        access-control: ::0/0 allow  #允许ipv6
        prefetch: yes    #消息缓存元素在它们到期之前被预取以保持缓存是最新的

        so-rcvbuf: 8m
        so-sndbuf: 8m
        msg-cache-size: 64m   #消息缓存的字节数。
        rrset-cache-size: 128m   #RRset缓存的字节数。
        outgoing-num-tcp: 256   #为每个线程分配的传出TCP缓冲区数
        incoming-num-tcp: 1024   #为每个线程分配的传入TCP缓冲区数
    ```

### 配置DNS转发

- 规则编写原理
    ```
    forward-zone:
            name: 'baidu.com.'   //单双引号都可以
            forward-addr: 114.114.114.114
    ```
	
    上述配置的意义：定义一个新的转发配置，baidu.com都使用114.114.114.114去解析，可以定义多个forward-zone，每个forward-zone都有优先级，定义越靠前，优先级越高。
    
    以下为本地域名配置:
    ```
    local-data: "my.domain.com 600 A 192.168.1.10"  //600为解析缓存时间
    ```

- 在/etc/unbound/下新建文件in_china.conf，内容如下：

    ```
    forward-zone:
            name: 'baidu.com.'
            forward-addr: 114.114.114.114
    
    forward-zone:
            name: 'taobao.com.'
            forward-addr: 114.114.114.114
            
    forwar-zone:
            name: 'qq.com.'
            forward-addr: 114.114.114.114
    ...
    ```
    国内域名都使用114.114.114.114解析，上述国内域名配置列表，托管在我的github，有空会补上链接的。
    
- 在/etc/unbound下新建文件out_china.conf，内容如下:

    ```
    forward-zone:
            name: '.'
            forward-addr: 8.8.8.8
            forward-addr: 1.1.1.1
    ```
    除国内域名外的其他域名使用8.8.8.8，1.1.1.1解析。
    
- include配置文件

    在/etc/unbound/unbound.conf的末尾添加如下内容:
    ```
    include "/etc/unbound/in_china.conf"
    include "/etc/unbound/out_china.conf"
    ```
    
    由与forward-zone是有优先级的，所以国内域名的配置必须引入在国外域名配置之前。
    
### 测试unbound

-    
    ```
    $ sudo service unbound start    //启动unbound服务
    $ sudo service unbound status   //查看服务状态
    ```

- 因为unbound启动失败并不会报任何错误，所以启动unbound后一定使用上述命令查看unbound服务状态，确定配置没有任何错误。

- 测试解析

    ```
    dig google.com @192.168.1.4 //替换为你自己的unbound服务器ip，输出google的ip地址即可。
    ```