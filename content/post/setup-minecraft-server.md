---
title: "Setup Minecraft Server"
date: 2023-04-20T14:58:58+08:00
draft: true
tags: ['minecraft','server','games']
---

由于我的一个朋友最近厌倦了网易的MC，想要体验一下之前玩过的Java版，
还想玩[AoA3]()这个模组~~玩刺激的RPG~~。
正好我手头有空闲的计算资源，用本文来记录一下搭建过程和遇到的问题。

<!--more-->

## 准备工作

调研的时候自然是优先打算采取[`Docker`]()方案的，
毕竟上次开服务器的时候`Docker`证明了它的易用性和稳定性。

但是很快遇到了和搭建`CI`系统时一样的问题，
我用于搭建MC服务器的主机并非我家里的主机一样有透明代理，
本地代理显然也不是一个优雅的解决方案，
而且该主机上面也挂了不少服务，考虑之下还是上传`jar`包手动搭建。

由于这个朋友的朋友们都是萌新，有正版的人很少数，
再考虑到之前搭建MC服务器的时候有被顶替爆破的问题，
最后还是顺手又搭建了一个基于[Blessing Skin]()的皮肤站，
并开启了验证码、[正版验证]()和[世界树API]()三个功能。

### 游戏要求

AoA3模组在[MCWiki]()下列出的一般要求是：

- `MineCraft Java Edition 1.16.5`
- `Forge`

MC 1.16.5版本主要使用`Java8`运行，
但是通过[`ModernFix`]()这个模组可以让1.16版本的MC[在更改虚拟机参数的前提下]()使用`Java17`运行，
所以这里服务端和客户端都采用`Java17`。

~~也是因为`Debian Bullseye`已经不提供`Java8`的JRE包了~~

### 游戏服务器

搭建Minecraft服务器用的主机是一台`Debian Bullseye`的`LXC`虚拟机，没有`Java`环境。

#### 安装Java环境

安装 `openjdk-17-jre-headless`:

```shell
$ sudo apt install openjdk-17-jre-headless
```

查看实际的Java文件执行位置:

```shell
$ readlink $(readlink /usr/bin/java)
/usr/lib/jvm/java-17-openjdk-amd64/bin/java
```

两次`readlink`是因为`Debian`会将Java的可执行文件`/usr/bin/java`
连接到版本控制软链接`/etc/alternatives/java`上。

#### 创建用户

用户目录放在`/opt/minecraft`下。

```shell
$ sudo mkdir -p /opt/minecraft
$ sudo useradd -M -d /opt/minecraft -s /bin/bash mc-server 
$ sudo chown -R mc-server:mc-server /opt/minecraft
$ sudo su - mc-server
```

#### 可执行文件下载

- [Minecraft JE 1.16.5](https://www.minecraft.net/zh-hant/article/minecraft-java-edition-1-16-5)
- [Forge 1.16.5-36.2.39](https://files.minecraftforge.net/net/minecraftforge/forge/index_1.16.5.html)
    - Forge的下载界面需要点右上角的`Skip`才会开始下载
- 大量优化性和辅助性的mod（如`ModernFix`)


### 皮肤站

#### 依赖要求

新版`Blessing Skin`需要`PHP8.0`和一个后端数据库，这里选择`Postgresql`，并用`Redis`作为缓存。

`Debian Bullseye`的仓库里面并没有`PHP8`（虽然`Bookworm`已经支持了），
所以需要第三方源安装，这里选取的是`sury`的第三方`PHP`仓库。

安装第三方源：

```shell
$ sudo wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
$ sudo echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list
$ sudo apt update
```

安装相应的PHP依赖：

```shell
$ sudo apt install php8.0-fpm php8.0-zip php8.0-gd php8.0-mbstring php8.0-xml php8.0-redis
```

为`Blessing Skin`创建Postgres数据库：

```shell
$ sudo -u postgres psql

psql (13.9 (Debian 13.9-0+deb11u1))
Type "help" for help.
postgres=# CREATE ROLE minecraft WITH LOGIN PASSWORD 'APASSWORD';
postgres=# CREATE DATABASE blessingskin WITH OWNER minecraft TEMPLATE template0 ENCODING UTF8 LC_COLLATE 'en_US.UTF-8' LC_CTYPE 'en_US.UTF-8';
```

调整数据库访问权限：

```shell
$ sudo -e /etc/postgresql/13/main/pg_hba.conf
```

```conf
host    blessingskin    minecraft       127.0.0.1/32            scram-sha-256
local   blessingskin    minecraft                               scram-sha-256
```

为访问Postgresql数据库的`php`用户加上`postgres`和`redis`的`unix socket`访问权限：

```shell
$ sudo usermod -aG pgsock www-data
$ sudo usermod -aG redis www-data
```

#### 可执行文件下载

- [`Blessing Skin Server 6.0.2`](https://github.com/bs-community/blessing-skin-server/releases/download/6.0.2/blessing-skin-server-6.0.2.zip)


## 搭建过程

### 游戏服务器

#### 安装`server`和`forge`

将服务器放在`/opt/minecraft/aoa3-1.16.5`目录下，`Forge`安装包放在`/opt/minecraft`下。

新建一个原版服务器：

```shell
$ pwd
/opt/minecraft
$ pushd aoa3-1.16.5
$ java -jar server.jar nogui
```

在服务器成功运行之后，使用`Ctrl-C`停止，

安装`Forge`:

```shell
$ popd
$ java -jar forge-1.16.5-36.2.39-installer.jar --installServer aoa3-1.16.5 nogui
```

该过程需要稳定的网络连接。

安装完成之后，就可以删除`Forge`的安装包，并且配置`server.properties`和放置`mod`。

### 皮肤站

#### 检查PHP环境

首先查看`php-fpm`是否在正常运行：

```shell
$ sudo systemctl status php8.0-fpm
```

查看`php-fpm`的`unix-socket`位置：

```shell
$ grep "listen =" /etc/php/8.0/fpm/pool.d/www.conf 
listen = /run/php/php8.0-fpm.sock
;pm.status_listen = 127.0.0.1:9001
```

#### 配置站点

将站点文件放置于`/opt/www/blessingskin/`下，
复制环境文件并生成设置密钥：

```shell
$ unzip -d /opt/www/blessingskin blessing-skin-server-6.0.2.zip
$ pushd /opt/www/blessingskin
$ cp .env.example .env
$ php artisan key:generate
```

#### 配置`.env`

```ini
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=blessingskin
DB_USERNAME=minecraft
DB_PASSWORD=APASSWORD
DB_PREFIX=
...
```

> BlessingSkin访问`Postgresql`的`unix socket`会出现无法插入的问题，
> 这里使用IP:Port作为临时解决方案。

```ini
...
CACHE_DRIVER=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=sync

REDIS_CLIENT=phpredis
REDIS_SCHEME=unix
REDIS_SOCKET_PATH=/var/run/redis/redis-server.sock
REDIS_PASSWORD=null
```

> 并没有向站内用户发信的需求，所以直接使用`sync`节省资源。

#### 配置`nginx`虚拟主机

在配置之前使用`acme.sh`申请了ecc的域名证书。

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name mc.example.com;
    ssl_certificate /path/to/ecc.crt;
    ssl_certificate_key /path/to/ecc.key;
    ssl_ciphers EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+ECDSA+AES128:EECDH+aRSA+AES128:RSA+AES128:EECDH+ECDSA+AES256:EECDH+aRSA+AES256:RSA+AES256:EECDH+ECDSA+3DES:EECDH+aRSA+3DES:RSA+3DES:!MD5;
    ssl_protocols TLSv1.3;
    ssl_session_timeout 5m;
    ssl_prefer_server_ciphers on;
    add_header Strict-Transport-Security "max-age=63072000" always;
    root /opt/www/blessingskin/public;
    location / {
        index  index.php index.html index.htm;
        try_files $uri $uri/ /index.php?$query_string;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }

    location ~ [^/]\.php(/|$){
        # try_files $uri =404;
        fastcgi_pass  unix:/run/php/php8.0-fpm.sock;
        include fastcgi.conf;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root/$fastcgi_script_name;
    }
}
```


## 后续优化

### 游戏服务器

### 皮肤站

#### 插件市场

#### 正版验证

