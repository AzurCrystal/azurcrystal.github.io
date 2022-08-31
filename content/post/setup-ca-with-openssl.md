---
title: "使用OpenSSL建立私有CA"
date: 2022-08-06T19:42:42+08:00
draft: false
tags: ['experiment','ops','openssl','ca']
---

本文是[网络基建实验]({{< relref "experiment-on-ops">}})的其中一章。有关其他实验，请参照该文章目录。

电子证书认证机构(**CA**,Certificate Authority)是组织内网络基础设施建设的重要一环。建立一个私有的局域网CA，可以以较小的成本提供较高的内网安全性。  

本文记录了使用**OpenSSL**搭建可用的**根**CA及**二级**CA，并签发可用证书的过程。

<!--more-->

## 背景资料

### CA

#TODO

### X.509证书体系

#TODO

### 椭圆曲线数字签名算法 (ECDSA)

ECDSA是当下非常流行，且安全度相对较高的一种基于椭圆曲线的数字签名算法。
由于无需考虑兼容性问题，本实验将全程采用ECDSA对私钥进行数字签名。

使用如下命令查看OpenSSL支持的EC签名方式:

```shell
$ openssl ecparam -list_curves
...
  secp256k1 : SECG curve over a 256 bit prime field
  secp384r1 : NIST/SECG curve over a 384 bit prime field
  secp521r1 : NIST/SECG curve over a 521 bit prime field
...
  prime256v1: X9.62/SECG curve over a 256 bit prime field
...
```

我们通常选取 `P-256` , `P-384` 这两种椭圆曲线对私钥进行数字签名。
在OpenSSL里面，`P-256` 对应的是 `prime256v1`，而 `P-384` 对应 `secp384r1`。  

对于其他的一些经常出现的椭圆曲线而言：  
`secp256k1`: 它并不是 `P-256` 的标准实现，但是你可以经常在区块链项目中看见它。
它没有什么用，也没有特定优化，所以尽量不要选择它。  
`P-521` : `secp521r1` 则没有任何实际意义——
主流浏览器（指Chrome）在2015年已经放弃了对其的支持，且其本质上只是加长的 `P-256`。
如果 `P-256` 被攻破了，那么 `P-521` 也将毫无意义，而且其效率远低于 `P-256`。

## 实验目的

建立起以根CA为核心的CA证书系统以解决内网证书签发问题。

## 实验步骤

本实验在一台安装有 `OpenSSL 1.1.1n` 的 `Debian 11` 主机上进行。
默认所有CA证书均采用 `P-384` 签名，所有普通证书均采用 `P-256` 签名。

### 建立根CA

本实验以 `/opt/pki/CA` 作为根CA的存放目录。

#### 文件树
 
``` shell
$ export ROOT_CA_DIR=/opt/pki/CA
$ mkdir -p ${ROOT_CA_DIR}/{newcerts,certs,csr,crl,private}
$ touch ${ROOT_CA_DIR}/{ca.cnf,crlnumber,index,serial}
$ echo "01" > ${ROOT_CA_DIR}/serial
$ cd ${ROOT_CA_DIR}
```

- `newcerts` : 存放新发布的证书
- `certs`    : 存放已发布的证书和旧证书  
- `csr`      : 存放证书请求文件 (非必需)  
- `crl`      : 存放吊销证书  
- `private`  : 存放证书私钥  

* `ca.cnf`   : Openssl的CA配置文件
* `crlnumber`: 吊销证书编号
* `index`    : 存放颁发证书的数据库文件
* `serial`   : 下一个颁发证书的序列号

``` shell
$ tree ${ROOT_CA_DIR}
/opt/pki/CA
|-- ca.cnf 
|-- certs
|-- crl
|-- crlnumber
|-- csr
|-- index
|-- newcerts
|-- private
`-- serial
```

#### 制订根证书的 `ca.cnf` 文件

用作建立**根证书**示例的 `ca.cnf` 文件如下。

```conf
[ ca ]
default_ca = CA_Default           # 指定该CA配置文件采用的配置

[ CA_Default ]               
name_opt = ca_default               
cert_opt = ca_default

dir = /opt/pki/CA                 # !!所有有关于CA的文件存放位置
certs = $dir/certs                  
crl_dir = $dir/crl
serial = $dir/serial
database = $dir/index
unique_subject = no               # 设置为yes则database文件
                                  # 中的subject列不能出现重
                                  # 复值,建议为no         
new_certs_dir = $dir/newcerts
certificate = $dir/certs/ca.crt   # CA证书存放位置，签署用 
private_key = $dir/private/ca.key # CA私钥存放位置，签署用 
crlnumber = $dir/crlnumber
crl = $dir/crl.pem
RANDFILE = $dir/private/.rand

x509_extensions = usr_cert        # 使用 [ usr_cert ] 
                                  # 作为x509扩展插件         
                                  # 影响是否能签署CA证书
copy_extensions = copy

default_days = 365                # !!默认证书有效时长
default_crl_days = 30
default_md = sha384               # !!默认摘要算法
                                  # P-384对应SHA-384
                                  # P-256对应SHA-256
preserve = no                     # 兼容IE则yes，否则no

policy = default_policy           # 使用[ default_policy ]
                                  # 作为签署策略           
[ default_policy ]

# Policy一共有 match, supplied, optional三种
# - match    : 申请的证书该DN字段必须和CA证书的相关DN字段一致
# - supplied : 该DN字段必须存在，不要求一致
# - optional : 该DN字段可选，不要求存在

countryName = match
stateOrProvinceName = match
localityName = match
organizationName = match
commonName = supplied
organizationalUnitName = optional
emailAddress = optional

[ req ]
default_bits = 2048
default_keyfile = privkey.pem

# 证书申请时的默认DN字段，对应 
# [ req_distinguished_name ] 的内容
distinguished_name = req_distinguished_name

attributes = req_attributes
x509_extensions = v3_ca           # 使用 [v3_ca] 作为
                                  # x509扩展插件
string_mask = utf8only
utf8 = yes
prompt = yes                      # NO时必须存在默认DN字段
                                  # YES时则申请时可手动输入DN字段

[ req_distinguished_name ]        

# 默认DN字段
# 此处字段的存在情况应当与 [ default_policy ] 中的一致
countryName = CN 
stateOrProvinceName = Some_Province
localityName = Some_City
organizationName = An Organzition
commonName = An Organzition Root CA
organizationalUnitName = CA Services


[ usr_cert ]
# 来自于[ CA_Default ]的x509_extension设定
# 影响证书签发时的设定

# !!这张CA证书能否签发其他的CA证书
# 设定为TRUE时，可以签发CA证书，不可以签发普通证书
# 设定为FALSE时则反之

basicConstraints = CA:TRUE
 
[ v3_ca ]
# 来自于[ req ]的x509_extension设定
# 影响证书申请时的设定

# !!此处决定这张证书是否为CA证书
# 设定为TRUE时，则为CA证书
# 设定为FALSE时，则为普通证书

basicConstraints = CA:TRUE

[ req_attributes ] 
# 无实际作用，可以忽略
```

#### 生成根CA证书私钥

```shell
$ openssl ecparam           \
         -name secp384r1    \
         -genkey            \
         -noout             \
         -out private/ca.key
```

#### 创建根CA证书请求文件

```shell
$ openssl req -new          \
        -key private/ca.key \
        -out csr/ca.csr     \
        -config ca.cnf      
```

对证书请求文件进行校验：

```shell
openssl req                 \
       -text -noout -verify \
       -in csr/ca.csr
```

#### 根CA证书自签

如果之前在 `ca.cnf` 的 `[ req ]` 中设置了 `prompt = no`，
会直接以 ` [ req_distinguished_name ] ` 中的默认值进行自签。

```shell
$ openssl ca -selfsign      \
        -in csr/ca.csr      \
        -out certs/ca.crt   \
        -config ca.cnf      
```

对生成证书进行校验：

```shell
$ openssl x509        \
         -text -noout \
         -in certs/ca.crt
```

#### 添加根CA到操作系统当中

证书必须为 `*.crt`，否则不会被update命令识别。

```shell
$ cp certs/ca.crt /usr/local/share/ca-certificates/your-ca-name.crt
$ update-ca-certificates
```

**注意**：部分浏览器使用自带的CA，如Firefox，需要再在浏览器中添加根CA。


### 建立二级CA

本实验以 `/opt/pki/SecondaryCA` 作为二级CA的存放目录。

#### 文件树
 
``` shell
$ export SECONDARY_CA_DIR=/opt/pki/SecondaryCA
$ mkdir -p ${SECONDARY_CA_DIR}/{newcerts,certs,csr,crl,private}
$ touch ${SECONDARY_CA_DIR}/{ca.cnf,crlnumber,index,serial}
$ echo "01" > ${SECONDARY_CA_DIR}/serial
$ cd ${SECONDARY_CA_DIR}
```

#### 制订二级证书的 `ca.cnf` 文件

```shell
cp ${ROOT_CA_DIR}/ca.cnf ${SECONDARY_CA_DIR}/ca.cnf
```
参照根证书的 `ca.cnf` 文件，对如下区域进行了修改:

```conf
...
[ CA_Default ]               
...
dir = /opt/pki/CA                 # !!所有有关于CA的文件存放位置
...
[ req_distinguished_name ]        
...
commonName = An Organzition Secondary CA 
...
[ usr_cert ]
basicConstraints = CA:FALSE       # !! 该证书**不能**签发CA证书
                                  # 如果此处设定为 CA:TRUE，则可以
                                  # 继续签发下一级CA证书
 
[ v3_ca ]
basicConstraints = CA:TRUE        # !! 该证书**是**CA证书
...
```

#### 生成二级CA证书私钥

```shell
$ openssl ecparam           \
         -name secp384r1    \
         -genkey            \
         -noout             \
         -out private/ca.key
```

#### 创建二级CA证书请求文件

生成证书请求文件时，使用二级CA的配置，即二级CA的 `[ req ]` 字段：  
由于`[ req ]` 的 `x509_extension` 即 `[v3_ca]` 设置了 `basicConstraints = CA:TRUE` ，
此处申请的是一张 **CA证书** 。

```shell
$ openssl req -new          \
        -key private/ca.key \
        -out csr/ca.csr     \
        -config ca.cnf      
```

#### 使用根证书对二级CA证书进行签名

签名时使用的是签名证书的配置文件，此处为根证书 `${ROOT_CA_DIR}/ca.cnf` ；  
由于根证书的 `[ CA_Default ]` 配置的 `x509_extension` 设置了
`basicConstraints = CA:TRUE`，故根证书可用于**签署**CA证书。

```shell
$ openssl ca                \
        -in csr/ca.csr      \
        -out certs/ca.crt   \
        -config ${ROOT_CA_DIR}/ca.cnf      
```

### 使用二级CA签发证书

由于在二级CA的配置文件中签署部分我们配置了 
`basicConstraints = CA:FALSE`，
故此CA可以签发普通证书。
现在我们使用此二级CA签发 `foo.example.com` 的证书：

本实验在 `/opt/pki/foo.example.com` 中进行。

```shell
$ export CERT_NAME=foo.example.com
$ export NORMAL_CERT_DIR=/opt/pki/${CERT_NAME}
```

#### 制订证书请求所用的配置文件

编辑 `{NORMAL_CERT_DIR}/req.cnf` :

```conf
[ req ]
prompt = no                 # 以配置好的字段直接申请
distinguished_name = server_distinguished_name
req_extensions = req_ext
x509_extensions	= v3_req
attributes = req_attributes
 
[ server_distinguished_name ]
# 此处应当与CA配置文件当中所配置的字段一致性和存在性一致
countryName = CN
stateOrProvinceName = Some_Province
localityName = Some_City
organizationName = An Organzition
commonName = foo.example.com
organizationalUnitName = An Organzition Some App
 
[ v3_req ]
basicConstraints = CA:FALSE  # 申请普通证书           
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
 
[ req_attributes ]
 
[ req_ext ]
subjectAltName  = @alternate_names # SAN扩展
 
[ alternate_names ]
DNS.1 = foo.example.com
DNS.2 = bar.foo.example.com
```

申请证书的时候，`commonName`字段需要填写域名。  
SAN扩展字段影响了该证书可以解析的域名范围。

#### 生成私钥、请求文件、签发证书

```shell
$ cd ${NORMAL_CERT_DIR}
# 生成证书私钥
$ openssl ecparam           \
         -name prime256v1   \
         -md sha256         \
         -genkey            \
         -noout             \
         -out ${CERT_NAME}.key
# 生成证书请求文件 [使用请求配置文件]
$ openssl req -new               \
         -key ${CERT_NAME}.key   \
         -out ${CERT_NAME}.csr   \
         -config ${NORMAL_CERT_DIR}/req.cnf
# 签发证书 [使用二级CA的配置文件]
$ openssl ca                     \
         -in ${CERT_NAME}.csr    \
         -out ${CERT_NAME}.pem   \
         -config ${SECONDARY_CA_DIR}/ca.cnf
```

#### 整合证书链

由于证书颁发机构是二级CA，所以服务器需要根CA和二级CA的证书，需要对证书进行聚合操作：

```shell
$ cat ${CERT_NAME}.pem  \
      ${SECONDARY_CA_DIR}/certs/ca.crt \
      ${ROOT_CA_DIR}/certs/ca.crt | \
  tee ${CERT_NAME}_ALL.pem
```

其顺序以申请证书为首，高层级的CA排在上方，根CA排在最下方。
本例便是以`证书-二级CA-根CA`的方式排列的。

### 环境清理

结束后不要忘记清理无用的环境变量：

```shell
$ unset ROOT_CA_DIR SECONDARY_CA_DIR NORMAL_CERT_DIR CERT_NAME 
```

### 小结

本实验采用OpenSSL建立了实验性质的根CA和二级CA，
并使用二级CA签发了带有SAN扩展的证书，
初步实现了架设证书认证机构的需求。
然而，本实验有以下不足点仍然值得进一步探索：

- 未能做到限定CA证书的用途
- 签发操作繁琐，无法做到使用API自助签发
- 未能保证根证书私钥安全性
- 证书签发机构未能与其他组件进行联动

以及其他的，由于经验不足和负载压力不够，暂时并未考虑到的事务。

### 弯路

初步是想用 `cfssl` 进行公钥认证机构的搭建的。Cloudflare的工具非常好用，
但是文档非常的少，在处理申请认证二级CA证书的时候，
并不能确定是否真的符合根CA可以申请二级CA证书的条件，
于是弃用`cfssl`改用较为臃肿复杂的`OpenSSL`。

但不得不提，在集群当中，`cfssl`凭借其强大的基于`json`的证书申请配置，
有着相当好的`IaC`，即“基础设施即代码”的可用性，
相比`OpenSSL`更加适合集群的证书申请架构。

不过，对于CA这种几乎是一次性的证书申请而言，采用`OpenSSL`也是较为稳妥的方式。
虽然繁杂，但是过程明了，且文档清晰，国内外的参考实例也非常多，
非常适合搭建实验性质的CA机构。

在搭建CA的过程当中，选择证书的数字签名方式也走了不少弯路。
由于ECDSA相比RSA在相同强度下具有更高的效率，故这次搭建首选ECDSA，
然而却犯了“过度”的问题，凭借一己之见强行使用`P-521`，
并没有和`SHA-512`摘要算法相结合。在搜索过相关的椭圆曲线使用情况，以及参考了`LetsEncrypt`论坛的相关内容后，决定以`P-384`作为CA证书签名算法，
而以`P-256`这种效率最高的方式作为普通证书的签名算法。

之后的努力方向大致是先以其他的基础设施搭建为主，
在基础设施较为完备之后再考虑对CA机构进行优化。

## 参考资料

[RFC5480](https://tools.ietf.org/html/rfc5480)
[openssl.cnf参数详解](https://www.cnblogs.com/f-ck-need-u/p/6091027.html)  
[如何使用OpenSSL建立自己的CA](https://gist.github.com/Soarez/9688998)  
[如何建立根CA及二级CA](https://blog.csdn.net/mn960mn/article/details/85645805)  
[选取椭圆曲线加密方式的指南-2022](https://soatok.blog/2022/05/19/guidance-for-choosing-an-elliptic-curve-signature-algorithm-in-2022/)  
[Letsencrypt论坛对支持`P-521`的讨论](https://community.letsencrypt.org/t/does-lets-encrypt-support-secp521r1-a-k-a-p-521/178331/11)