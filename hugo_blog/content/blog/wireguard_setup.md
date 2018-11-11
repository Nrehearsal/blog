---
title: "使用wireguard搭建VPN服务"
date: 2018-11-11T21:28:20+08:00
draft: false
---

{{% summary %}}
> 活了一百年却只能记住30M字节是荒谬的。你知道，这比一张压缩盘还要少。人类境况正在变得日趋退。--Marvin Minsky
{{% /summary %}}

使用wireguard搭建VPN服务

[wireguard](https://www.wireguard.com/)是一款新型的，快速的，安全的VPN软件，将来可能会正式并入linux内核。wireguard搭建十分简单，不需要像strongswan，openvpn那样配置繁琐的认证证书，以及网络接口，总的配置文件也不会超过50行。

### 安装wireguard

- 
	```sh
	$ sudo add-apt-repository ppa:wireguard/wireguard
	
	$ sudo apt-get update
	
	$ sudo apt-get install wireguard
	```

### 生成密钥对


- ```sh
    wg genkey | tee privatekey | wg pubkey > publickey
    ```
    
- 命令执行完成后，将在当前目录下生成privateKey, publicKey两个文件，分别包含了公钥和私钥

### 开启内核转发

- 编辑/etc/sysctl.conf文件，找到net.ipv4.ip_forward=1，并去掉这一行的注释

### 配置文件编写

- 在/etc/wireguard/目录下，新建文件wg0.conf， 文件名字可以任意取，文件内容如下:

- 
    ```
    [Interface] //本端信息
    Address = 192.168.142.1/32  //本端ip
    SaveConfig = true
    PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
    ListenPort = 56632 //监听端口，任意指定即可
    PrivateKey = //服务端的私钥

    [Peer]  //对端信息
    PublicKey = //对端的公钥，即客户端的公钥，每个客户端也需要生成一对公私钥。
    AllowedIPs = 192.168.142.2/32 //对端的ip，即客户端的ip。
    ```

### 启动wireguard服务

- 
    ```
    wg-quick up wg0    //启动wireguard, wg0为配置文件的名字去掉.conf
	
    wg-qucik down wg0    //关闭wg0
    ```

### 客户端配置

- 在客户端的/etc/wireguard/目录下（Linux和mac平台， windows平台有GUI客户端倒入配置文件即可），也编写wg0.conf，内容如下： 
    ```
    [Interface]
    PrivateKey = //本段的私钥，即客户端的私钥
    Address = 192.168.142.3/24  //客户端的ip
    
    [Peer]
    PublicKey = //服务端的公钥
    AllowedIPs = 0.0.0.0/0
    Endpoint = 123.123.123.123:56632    //对端，即服务端的ip和端口号
    PersistentKeepalive = 25    //心跳配置
    ```

- 启动客户端
    ```
    wg-quick up wg0 //启动客户端
    wg-quick down wg0 //关闭客户端
    ```
