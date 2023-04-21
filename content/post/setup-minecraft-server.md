---
title: "配置MineCraft游戏服务器"
date: 2023-04-20T14:58:58+08:00
draft: false
tags: ['minecraft','server','games']
---

由于我的一个朋友最近厌倦了网易的MC，想要体验一下之前玩过的Java版，
还想玩[AoA3](https://www.curseforge.com/minecraft/mc-mods/advent-of-ascension-nevermine)这个模组~~玩刺激的RPG~~。
正好我手头有空闲的计算资源，用本文来记录一下搭建过程和遇到的问题。

<!--more-->

## 准备工作

调研的时候自然是优先打算采取[`Docker`](https://hub.docker.com/r/itzg/minecraft-server/)方案的，
毕竟上次开服务器的时候`Docker`证明了它的易用性和稳定性。

但是很快遇到了和搭建`CI`系统时一样的问题，
我用于搭建MC服务器的主机并非我家里的主机一样有透明代理，
本地代理显然也不是一个优雅的解决方案，
而且该主机上面也挂了不少服务，考虑之下还是上传`jar`包手动搭建。

由于这个朋友的朋友们都是萌新，有正版的人很少数，
再考虑到之前搭建MC服务器的时候有被顶替爆破的问题，
最后还是顺手又搭建了一个基于[Blessing Skin](https://github.com/bs-community)的皮肤站，
并开启了验证码、[正版验证](https://github.com/bs-community/blessing-skin-plugins/tree/master/plugins/mojang-verification)
和[世界树API](https://github.com/bs-community/blessing-skin-plugins/tree/master/plugins/yggdrasil-api)三个功能。

### 游戏要求

AoA3模组在[MCWiki](https://www.mcmod.cn/class/1269.html)下列出的一般要求是：

- `MineCraft Java Edition 1.16.5`
- `Forge`

MC 1.16.5版本主要使用`Java8`运行，
但是通过[`ModernFix`](https://www.mcmod.cn/class/8714.html)这个模组，
可以让1.16版本的MC[在更改虚拟机参数的前提下](https://github.com/embeddedt/ModernFix/wiki/1.16---required-arguments-for-Java-17)使用`Java17`运行，
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

#### 编写启动脚本

由于使用`Java17`启动`Minecraft 1.16.5 + Forge`，所以必须对虚拟机参数做一定的修改，
并且限制MC本身分配的内存。

```shell
#!/bin/sh
/usr/lib/jvm/java-17-openjdk-amd64/bin/java \
	-Xms4G -Xmx8G \
	-Djava.security.manager=allow \
	-Dfile.encoding=UTF-8 \
	--add-opens java.base/jdk.internal.loader=ALL-UNNAMED \
	--add-opens java.base/java.net=ALL-UNNAMED \
	--add-opens java.base/java.nio=ALL-UNNAMED \
	--add-opens java.base/java.io=ALL-UNNAMED \
	--add-opens java.base/java.lang=ALL-UNNAMED \
	--add-opens java.base/java.lang.reflect=ALL-UNNAMED \
	--add-opens java.base/java.text=ALL-UNNAMED \
	--add-opens java.base/java.util=ALL-UNNAMED \
	--add-opens java.base/jdk.internal.reflect=ALL-UNNAMED \
	--add-opens java.base/sun.nio.ch=ALL-UNNAMED \
	--add-opens jdk.naming.dns/com.sun.jndi.dns=ALL-UNNAMED,java.naming \
	--add-opens java.desktop/sun.awt.image=ALL-UNNAMED \
	--add-modules jdk.dynalink \
    --add-opens jdk.dynalink/jdk.dynalink.beans=ALL-UNNAMED \
	--add-modules java.sql.rowset \
	--add-opens java.sql.rowset/javax.sql.rowset.serial=ALL-UNNAMED \
	--add-exports java.base/sun.security.util=ALL-UNNAMED \
	--add-opens java.base/java.util.jar=ALL-UNNAMED \
	-javaagent:authlib-injector-1.2.2.jar=https://mc.azurcrystal.com/api/yggdrasil \
	-jar forge-1.16.5-36.2.39.jar nogui
```

之后可以根据这个启动脚本编写`systemd`文件或者使用`tmux`直接管理，
由于MC服务器并不要求自动启动和自动重启，
而且为了方便快捷的使用`forge`内置的`Shell`，我这里选择了`tmux`。

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

### 皮肤站

#### 镜像插件市场

鉴于[插件市场的迁移](https://t.me/blessing_skin_news/781)，且`git.qvq.network`在国内的连接并不理想，
遂在服务器端本地复制了一份[插件市场](https://github.com/bs-community/plugins-dist)，
并将其`registry_zh_CN.json`和`registry_en.json`中的地址进行[替换](https://git.azurcrystal.com/AzurCrystal/plugins-dist/src/branch/master/registry_zh_CN.json)，
相当于在本地镜像了一份插件市场，之后更改`.env`：

```ini
PLUGINS_REGISTRY=https://git.azurcrystal.com/AzurCrystal/plugins-dist/raw/master/registry_{lang}.json
```

之后只需要定时向上游拉取并自动打上`Patch`就可以在本地镜像了。

#### 正版验证

由于劫持了世界树API，实际上正版验证在这个环节中已经成为可有可无的部分了。
在`yggdrasil api`由皮肤站运营的前提下，所有通过皮肤站账户连入的用户都是正版，
唯一的区别可能就是使用的`UUID`与实际正版账户不同了。
保持各个采用世界树API的`UUID`不同是一个明智的选择。

##### 环境配置

为了启用正版验证，需要在`.env`中配置以下三个值：
```ini
MICROSOFT_KEY=9fce0559-44b4-4c95-a144-d3ccf50ea62b
MICROSOFT_SECRET=secret@123
MICROSOFT_REDIRECT_URI=https://skin.bs-community.dev/mojang/callback
```

##### 注册微软应用

打开`https://aka.ms/aad`后，实际上会跳转到`https://azure.microsoft.com/en-us/products/active-directory/`，
也就是微软官方的身份验证服务。

点击`登录(Signin)` 并用微软账户登录以后，可以进到主控制台界面。

找到顶部的`Azure服务`中的`Azur Active Directory`，进入后选择`添加-应用注册`，输入应用名称（随意输入）；
勾选`受支持的账户类型`为以下的其中一项，

- `任何组织目录(任何 Azure AD 目录 - 多租户)中的帐户和个人 Microsoft 帐户(例如，Skype、Xbox)` 
- `仅 Microsoft 个人帐户`

并将`重定向URI`设置为`Web`，值为`https://mc.example.com/mojang/callback`。

之后点击完成创建，完成创建后的应用的`应用程序(客户端)ID`即是`MICROSOFT_KEY`。

`MICROSOFT_SECRET`则需点击左侧列表中的`证书和密码`，新建一个`客户端密码`，其`值`（非机密ID）即为变量值。
需注意，`客户端密码值`出现一次后就无法再查看，请及时保存。

之后按照所设置的变量更改`.env`，即可启用基于`Microsoft`账户的皮肤站内正版验证。

## 问题

### 皮肤站

- 无法直接从皮肤站的上传页面上传皮肤
    - 推测是`php8`的处理问题，正在考虑编译一份`php8.0.2`
    - 目前可以直接从启动器中上传皮肤
- 正版验证无法更改玩家UUID
    - 属于良性BUG
    - 更改UUID会导致玩家存档消失，而且容易混淆，反而不应当更改
    - 如需正版皮肤和披风应当使用`CustomSkinLoader`这个插件
