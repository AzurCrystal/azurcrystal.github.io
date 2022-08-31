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

### rsync

#### 定义

#TODO

#### 选用原因

选用`rsync`协议的原因有如下几点：

- 协议方面
    1. `rsync`在完成初始同步以后，无需额外设置即可进行增量同步
    1. `rsync`同步时可以对源文件进行压缩，节省同步时使用的带宽

- 上游方面：
    1. 主流上游镜像站点均鼓励使用`rsync`协议进行同步
    1. http/https/ftp同步容易触发连接限制导致同步失败

但是，在对镜像站的原子性做强要求时，`rsync`[并不能保证其同步具有原子性](https://linux.die.net/man/1/rsync)(`--delay-updates`)，需要外部脚本进行原子性的保障。

#TODO 本实验目前并未将镜像站的原子性列入强要求，但之后会考虑利用写时复制（Copy-On-Write)使镜像更新成为原子性操作，此操作需要支持COW的文件系统（如`btrfs`），但目前没有条件。

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

#### Alpine源的组织形式

#### 使用rsync直接同步Alpine源

[Alpine Wiki](https://wiki.alpinelinux.org/wiki/How_to_setup_a_Alpine_Linux_mirror)对如何拉取一个rsync镜像做了详细的描述，其中提到了

### 镜像Debian源

#### Debian源的组织形式

Debian源的组织目录较为简洁。  
现以只同步了`Debian-bullseye`的`amd64`架构的预构建二进制包，未同步源码的镜像源为例：

```shell
$ tree -L 2

.
|-- dists
|   |-- bullseye
|   |-- bullseye-backports
|   |-- bullseye-updates
|   |-- stable -> bullseye
|   --- stable-updates -> bullseye-updates
--- pool
    |-- contrib
    |-- main
    --- non-free
```

- `dists`: 记录发行版当中软件包的哈希值
- `pool`: **存放**软件包


由于Debian的发行版较多，且存在同版本软件包横跨数个发行版提供的情况，因此Debian采用的是 **只记录发行版中软件包Hash值** 的形式进行单一发行版的软件包提供。该Hash被记录在`dists`目录下，每个文件夹对应一个发行版的软件包总Hash，而`dists`下记录的hash对应的则是`pool`目录下存放的软件包。  
例如，`dists/bullseye` 的结构如下：

```shell
$ tree -d
.
|-- contrib
|   |-- binary-all
|   |-- binary-amd64
|   |-- dep11
|   --- i18n
|-- main
|   |-- binary-all
|   |-- binary-amd64
|   |-- debian-installer
|   |   |-- binary-all
|   |   --- binary-amd64
|   |-- dep11
|   --- i18n
--- non-free
    |-- binary-all
    |-- binary-amd64
    |-- dep11
    --- i18n
```

通过Debian官方建议方式(`ftpsync`,`debmirror`,`apt-mirror`S)同步任意架构（如`amd64`和`i386`）时，全架构通用（`all`）也会一同被同步。

`dists/stable` 则指向 `bullseye`，代表 `Debian-bullseye` 是当前的 `stable` 发行版。

#### Debian社区提供的镜像方式

[Debian Mirror Site](https://www.debian.org/mirror/ftpmirror)提供了建立一个对外开放的Debian镜像的相关建议，结合网上已有的经验，共可根据目的分为以下几种；  
Debian的镜像仍然可以通过`rsync`协议进行同步，但是具有多种不同的手段。

如果想要全量拉取所有架构和发行版，可以直接使用`rsync`进行同步，也可以

拉取Debian镜像可以通过`apt-mirror`

## 总结

## 参考资料