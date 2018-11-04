---
title: "使用kubeadmin搭建kubernetes集群(一)"
date: 2018-11-05T00:47:20+08:00
draft: false
---

{{% summary %}}
> 没有一个人的记性，好到可以作个成功的说谎者——林肯
{{% /summary %}}

想在vps上搭建一个kubernetes单节点Demo，遇到很多问题，仍未搭建完成。如果仅是用来测试的话，建议使用minikube+virtualBox安装。

**使用kubeadmin搭建kubernetes集群(一)**

- 安装环境
    · Ubuntu16.04
    · 2GB RAM
    . 能够访问google的网络环境(非常重要)
	
    master节点占用的端口:

    协议 | 流向 | 端口号 | 用途 | 占用组件
    -----|------|--------|------|---------
    TCP  |入站  | 6443 | kubernetes API server | all
    TCP  | 入站 | 2379-2380 | etcd server client API | kube-apiserver, etcd
    TCP  | 入站 | 10250 | Kubelet API | Self, Control plane
    TCP  | 入站 | 10251	| kube-scheduler | self
    
    worker节点占用的端口:
	
     协议 | 流向 | 端口号 | 用途 | 占用组件
    -----|------|--------|------|---------
    TCP  | 入站 | 10250	| Kubelet API | Self, Control plane
    TCP	 | 入站 | 30000-32767 | NodePort Services** | All

- 安装docker
    . 卸载旧版本
    ```
    $ sudo apt-get remove docker docker-engine docker.io
    ```
    . 为apt添加https传输支持
    ```
    $ sudo apt-get update
    $ sudo apt-get install apt-transport-https ca-certificates curl software-properties-common      //curl用来下载一些小文件
    ```
    . 添加仓库GPG密钥(中科大源)
    ```
    $ curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
    ```
    . 添加仓库地址到source.list
    ```
    sudo add-apt-repository "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
    ```
    .安装docker
    ```
    $ sudo apt-get update
    $ sudo apt-get install docker-ce
    ```
    . 启动docker服务
    ```
    $ sudo systemctl enable	docker
    $ sudo systemctl start docker
    ```
    . 创建docker用户组并添加当前用户到docker用户组
    ```
    $ sudo groupadd docker
    $ sudo usermod	-aG	docker $USER
    ```

- 安装kubeadm, kubelet, kubectl, 
    .kubeadm: 初始化群集的命令。

    .kubelet: 在群集中的所有计算机上运行的组件，用于启动pod和容器。

    .kubectl: 与您的集群通信的命令行工具。

    · 添加官方源的GPG密钥
    ``` sh
    $ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    ```

    · 添加kubernetes仓库地址
    ``` sh
    $ echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
    ```

    . 安装kubeadm, kubelet, kubeead
    ``` sh
    $ apt-get install -y kubelet kubeadm kubectl
    ```
- 初始化节点
    ``` sh
    $ kubeadm init  //该命令有很多参数，--pod-network-cidr=0.0.0.0/0 可指定pot子网
    --apiserver-advertise-address=<ip> 指定apiserver地址
    ```
    
    未完待续...
    
- 安装过程中遇到的一些问题
    . 一定要能够访问google，否则很多东西不能下载，导致安装过程报错。

    . kubenetes运行需要禁用swap分区功能，参考[这里](https://askubuntu.com/questions/214805/how-do-i-disable-swap)。
    
    . 初始化节点失败，可使用kubeadm reset重置环境，有时候需要手动结束一些进程，才能从新初始化节点。

参考资料: 
    . [Installing Kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)
    . [Creating a single master cluster with kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)

