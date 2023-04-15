---
title: "使用OpenLDAP建立LDAP服务"
date: 2022-08-07T22:31:53+08:00
draft: false
tags: ['experiment','ops','ldap']
---

#TODO

本文是[网络基建实验]({{< relref "experiment-on-ops">}})的其中一章。有关其他实验，请参照该文章目录。

<!--more-->

## 背景

### LDAP服务应用
 
### 中间层

### `mdb`/`bdb`/`hdb`以及其他后端

在许多LDAP配置教程当中，采用的多是`hdb`,`bdb`等存储后端，
而现在在主要发行版上安装的`OpenLDAP`则多是`mdb`作为存储后端。

依据[OpenLDAP FAQ](https://www.openldap.org/faq/data/cache/756.html)所述，
 `back-bdb` 和 `back-hdb` 均是基于 `BerkleyDB` 的存储后端，且**已弃用**。



## 实验步骤

本实验在一台安装有 `OpenLDAP2.6.2` 的 `Alpine 3.16` 主机上进行。
在 `Alpine` 上，`openldap` 包的安装默认不携带 `backend` 和 `overlay`。 为了最基础的

### 修改默认配置

Alpine默认安装的`OpenLDAP`采用的是slapd.conf配置文件的配置方式，但是从 `OpenLDAP2.3` 开始，使用 `slapd.d` 目录成为了默认选项。



## 总结

## 参考资料
