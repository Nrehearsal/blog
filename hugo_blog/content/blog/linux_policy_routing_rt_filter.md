---
title: "linux内核参数rp_filter对策略路由的影响"
date: 2018-12-16T20:06:12+08:00
draft: false
---

{{% summary %}}
> 爱国主义就是积极地为了微不足道的原因杀人并被杀。——勃特兰·罗素
{{% /summary %}}

linux内核参数rp_filter对策略路由的影响

### 环境说明

- vpn软件如openvpn，wireguard等，通过配置这样两条路由
	
	`0.0.0.0/1 via vpn_gateway dev tun0`
	
	`128.0.0.0/1 via vpn_gateway dev tun0`

    来实现全局路由，这两条路由的含义如下:
	
    `0.0.0.0/128.0.0.0 = 0.0.0.0/1 = 0.0.0.0 TO 127.255.255.255`
	
    `128.0.0.0/128.0.0.0 = 128.0.0.0/1 = 128.0.0.0 TO 255.255.255.255`
	
    即除了127.0.0.0这个网络外的所有流量都通过VPN出去。

- 如果只想特定的流量通过vpn，我们可以这样配置，例如只让8.8.8.8通过vpn

    ```sh
    //为路由表vpn_route_table添加默认路由
    $ ip route add default via vpn_gateway dev tun0 table vpn_route_table
    //添加策略，只有fwmark为0xffff的数据包使用vpn_route_table查询路由
    $ ip rule add fwmark 0xffff table vpn_route_table
    //在mangle表中的OUTPUT链上为目的地址为8.8.8.8的数据包打上标签
    $ iptables -t mangle -A OUTPUT -d 8.8.8.8 -j MARK --set-mark 0xffff
    //以下配置是为了将mark设置到整个网络连接过程中
    $ iptables -t mangle -A OUTPUT -j CONNMARK --save-mark
    $ iptables -t mangle -A PREROUTING -j CONNMARK --restore-mark
    //为tun0接口设置nat
    $ iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
    ```
    
-   配置完成后，发现不能正常访问8.8.8.8

### 问题排查

- 使用tcpdump查看网络接口流量，发现通往8.8.8.8的数据包通过vpn接口出去，并且有数据包返回至vpn接口，如下所示：
    ```sh
    $ sudo tcpdump -i tun0

    20:43:41.269724 IP 192.168.142.3 > google-public-dns-a.google.com: ICMP echo request, id 1879, seq 1, length 64
    20:43:41.448500 IP google-public-dns-a.google.com > 192.168.142.3: ICMP echo reply, id 1879, seq 1, length 64
    20:43:42.277595 IP 192.168.142.3 > google-public-dns-a.google.com: ICMP echo request, id 1879, seq 2, length 64
    20:43:42.457883 IP google-public-dns-a.google.com > 192.168.142.3: ICMP echo reply, id 1879, seq 2, length 64
    20:43:43.285593 IP 192.168.142.3 > google-public-dns-a.google.com: ICMP echo request, id 1879, seq 3, length 64
    20:43:43.901840 IP google-public-dns-a.google.com > 192.168.142.3: ICMP echo reply, id 1879, seq 3, length 64
    ```
    确定问题出现在本地路由配置，vpn软件工作正常。
    
- 通过google查询各种资料，发现问题出在rp_filter内核参数，使用命令`sudo sysctl -w net.ipv4.conf.tun0.rp_filter=2`，将tun0接口的rp_filter设置为2时，路由恢复正常。

### 关于rp_filter参数

- **What is IP address spoofing?**

    IP spoofing is a method adopted by attacker's to send forged source address in their attack traffic.Which means they can send an IP packet with an IP address of their wish.

    Most of the time's spoofing is used by an attacker mainly for the following reasons.
    - To conduct a DDOS attack ,and he does not want the response from the target machine to reach him
    - To compromise source based authentication
    
    Spoofing can be controlled to a cerain extent by using Reverse Path filtering(not fully although).
    
- **What is reverse path filtering?**

    Reverse path filtering is a mechanism adopted by the Linux kernel, as well as most of the networking devices out there to check whether a receiving packet source address is routable.

    So in other words, when a machine with reverse path filtering enabled recieves a packet, the machine will first check whether the source of the recived packet is reachable through the interface it came in.

    - If it is routable through the interface which it came, then the machine will accept the packet
    - If it is not routable through the interface, which it came, then the machine will drop that packet.

    Latest red hat machine's will give you one more option. This option is kind of liberal in terms of accepting traffic.

    - If the recieved packet's source address is routable through any of the interfaces on the machine, the machine will accept the packet.
  
- rp_filter的取值和代表的动作

    - 0：关闭反向路由校验
    - 1：开启严格的反向路由校验。对每个进来的数据包，校验其反向路由是否是最佳路由。如果反向路由不是最佳路由，则直接丢弃该数据包。
    - 2：开启松散的反向路由校验。对每个进来的数据包，校验其源地址是否可达，即反向路由是否能通（通过任意网口），如果反向路径不通，则直接丢弃该数据包。
    
参考链接：

- [Linux policy routing - packets not coming back](https://serverfault.com/questions/456308/linux-policy-routing-packets-not-coming-back)
    
- [Linux kernel rp_filter settings (Reverse path filtering)](https://www.slashroot.in/linux-kernel-rpfilter-settings-reverse-path-filtering)