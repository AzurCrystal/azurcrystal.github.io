---
title: "基于Rsync建立私有Linux镜像源"
date: 2022-08-15T00:27:59+08:00
draft: false
tags: ['infrastructure']
---

#TODO

本文是[网络基建实验]({{< relref "experiment-on-infrastructure">}})的其中一章。有关其他实验，请参照该文章目录。

<!--more-->

## 背景

由于基建实验环境位于独立的内网当中，为防止从内网环境进行网络穿透以造成不必要的影响，绝大部分无必要的内网基础设施均无法直接访问公网，仅少数专用机可以通过专门分配的NAT访问外网（如[VPN]({{< relref "setup-openvpn-server" >}})和[DNS]({{< relref "setup-bind9-dns-server" >}})）。

然而，不放通公网导致内网当中的已存主机和新建主机无法通过公网上的镜像源进行应用更新，且对于其他使用公网镜像的主机而言，一份本地同步的局域网镜像可以有效节省出口带宽，增加更新速度，达到了以**空间**换**时间**的效果。

因此，结合之前的[实践]({{< relref "experiment-on-infrastructure">}})，本文将进行从镜像站上游拉取本地镜像并对局域网提供服务的实验。

选用`rsync`协议的

## 实验目的

以`rsync`为基础，建立局域网镜像体系，对镜像进行定时更新并向局域网内主机进行http/**https**/ftp协议的本地镜像服务。  
初步计划引入如下镜像：

- `Alpine v3.16`
- `Debian Bullseye(11)`  

由于服务器存储空间有限加实验环境架构单一，故只镜像了`amd64`架构的预编译软件包，**不包含**源码。

## 实验步骤

本实验在一台全新安装的 `Debian 11` 主机上进行，镜像源为[北京外国语大学镜像站](https://mirrors.bfsu.edu.cn)，采用`rsync`协议进行镜像。

### 准备工作

在主机上安装 `rsync` 和用于对外提供服务的 `nginx` :

```shell
$ apt install -y rsync nginx
```

新建 `rsync` 使用的专用用户:

```shell
$ groupadd rsync
$ useradd -g rsync -m -s /bin/bash rsync
```

这里使用`Bash`作为执行用户的登入Shell而非`/sbin/nologin`，以方便后续通过`ssh`从上游接受镜像更新的推送信息。

建立镜像存放文件夹:

```shell
$ mkdir -p /opt/{mirrors,mirror-tools}
```

其中，`mirrors`文件夹用于存放镜像文件，而`mirror-tools`文件夹用于存放更新镜像用的脚本和工具文件。

### 镜像Alpine源

[Alpine Wiki](https://wiki.alpinelinux.org/wiki/How_to_setup_a_Alpine_Linux_mirror)对如何拉取一个rsync镜像做了详细的描述，其中提到了

### 镜像Debian源

## 总结

## 参考资料