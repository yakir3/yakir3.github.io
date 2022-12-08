---
title: Openssl 原理与操作
categories:
  - Linux
tags:
  - openssl
abbrlink: b8bc
date: 2022-12-08 22:52:20
---
### 一、原理
> 参考：
> [https://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html](https://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html)
> [https://www.ruanyifeng.com/blog/2014/09/illustration-ssl.html](https://www.ruanyifeng.com/blog/2014/09/illustration-ssl.html)
> [https://www.nixops.me/articles/ssl-cetificate-create-and-signed.html](https://www.nixops.me/articles/ssl-cetificate-create-and-signed.html)

{% asset_img openssl-1.jpeg %}

### 二、证书操作
#### 1、第三方签发 SSL 证书
```shell
# 生成私钥，加密参数 -des3
openssl genrsa -out a.com.key 2048


# 生成 csr 文件
openssl genrsa -out a.com.key 2048 openssl req -new -sha256 -key a.com.key -out a.com.csr
# 查看 csr 信息
openssl req -noout -text -in a.com.csr


# 提交 csr 文件到 CA 或第三方证书机构获取数字签名后的公钥 crt 文件
# 部署 crt 与 key 文件至 web 服务器
```
<!--more-->

#### 2、自签名/自有 CA 签发证书

- 使用自签名方式
```shell
# 使用已有私钥和 csr 自签名
openssl x509 -req -days 3650 -in a.com.csr -extensions v3_ca -signkey a.com.key -out a.com.crt
# 或直接生成私钥+公钥
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout a.com.key -out a.com.crt


# 直接生成自签名证书
openssl req -x509 -days 3650 -new -newkey rsa:2048 -nodes -keyout a.com.key -out a.com.csr -subj "/C=hk/ST=hk/L=hk/O=hk/OU=hk/CN=*.a.com"
# 验证私钥和公钥
openssl rsa -check -in a.com.key 
openssl x509 -text -noout -in a.com.crt 
```

- 使用 CA 签名
```shell
# 生成 CA 公私钥
openssl genrsa -out ca.key 2048
openssl req -x509 -sha256 -new -nodes -key ca.key -out ca.crt -days 36500 -subj "/C=CN/ST=HK/L=HK/O=HK/OU=HK LTD/CN=a.com"
# 查看 CA 证书详细信息
openssl x509 -in ca.crt -text


# 使用 CA 签发证书
openssl x509 -req -days 365 -sha256 -CA ca.crt -CAkey ca.key -CAcreateserial -in a.com.csr -out a.com.crt
# 更好兼容浏览器
cat ca.crt >> a.com.crt


# 查看签发者信息
openssl x509 -noout -issuer -issuer_hash -in a.com.crt
```

#### 3、使用 openssl 加密解密文件
```shell
# 生成并验证私钥
openssl genrsa -out yakir.key 2048 
openssl rsa -in yakir.key -text -noout

# 导出公钥并验证
openssl rsa -in yakir.key -pubout -out yakir.pub
openssl rsa -in yakir.pub -pubin -text -noout 

# 加解密小文件
# 使用这种方式 1024 位的私钥可以加密小于 86 字节的文件，2048 位的私钥可以加密小于 214 字节的文件。
# 用公钥加密
openssl rsautl -encrypt -inkey yakir.pub -pubin -in file.txt -out file.bin 
# 用私钥解密
openssl rsautl -decrypt -inkey yakir.key -in file.bin 

# 加解密大文件
# 用公钥加密
openssl smime -encrypt -aes256 -in Large.zip -binary -outform DEM -out Encrypted.zip yakir.pub
# 用私钥解密
openssl smime -decrypt -in Encrypted.zip -binary -inform DEM -inkey yakir.key -out Large.zip 
```
### 
#### 4、证书格式转化
> 一般有以下几种标准格式
> - .DER .CER ： 二进制格式，只保存证书，不保存私钥。
> - .PEM ：文本格式，可保存证书，可保存私钥，通常网上的.key 后缀的私钥，其实就是 PEM 格式。
> - .CRT ：可以是二进制格式，可以是文本格式，只保存证书，不保存私钥。
> - .PFX .P12 ：即 PKCS12，是二进制格式，同时包含证书和私钥，一般有密码保护。
> - .JKS ：JAVA 的专属二进制格式，同时包含证书和私钥，一般有密码保护

```shell
# DER/CER/CRT 转 PEM
# 先查看证书信息，在转格式
openssl x509 -in cert.der -inform der -text -noout openssl x509 -in cert.der -inform der -outform pem -out cert.pem

# PEM 转 DER/CER/CRT
openssl x509 -in cert.pem -text -noout openssl x509 -in cert.pem -outform der -out cert.der

# PFX 转 PEM
openssl pkcs12 -info -nodes -in site.pfx openssl pkcs12 -in site.pfx -out site.pem -nodes

# JKS 转 PEM
# 需要 JDK 中提供的 keytool 工具配合 openssl, 先用 keytool 转成 PKCS12 格式：
keytool -importkeystore -srckeystore cert.jks -destkeystore cert.pkcs -srcstoretype JKS -deststoretype PKCS12
# 在用 openssl 转成 pem 格式
openssl pkcs12 -in cert.pkcs -out cert.pem
```

#### 5、其它一些技巧
```shell
# 移除证书中的密码
openssl rsa -in cert.key -out nopass.key

# 查看公钥的 hash
openssl x509 -noout -hash -in cert.pem

# 查看在线网站的证书信息
openssl s_client -connect www.baidu.com:443 -showcerts

# 查看网站证书的有效期
# 查看本地证书
openssl x509 -dates -noout -in file.pem #查看证书签发时间和有效期 openssl x509 -startdate -noout -in file.pem #查看签发时间 openssl x509 -enddate -noout -in file.pem #查看有效时间 openssl x509 -checkend 86400 -noout -in file.pem #检查证书是否在一天内过期，用echo $?即可判断
# 查看在线证书
openssl s_client -connect www.baidu.com:443 -servername www.baidu.com 2> /dev/null | openssl x509 -noout -dates
# 提取过期时间
openssl s_client -connect www.baidu.com:443 -servername www.baidu.com 2>/dev/null |openssl x509 -enddate -noout |cut -d "=" -f 2
# date 命令转换一下日期格式
date --date="$(openssl s_client -connect www.baidu.com:443 -servername www.baidu.com 2>/dev/null |openssl x509 -enddate -noout |cut -d "=" -f 2)" --iso-8601

# 检查网站是否接受指定版本的 SSL 协议
# 协议有 TLS 1.0 (tls1)、TLS 1.1 (tls1_1) TLS 1.2 (tls1_2), 在高版本的 openssl 中默认已经禁用了 SSL V2 (ssl2)、SSL V3 (ssl3)
openssl s_client -connect www.baidu.com:443 -tls1

# 检查网站是否支持指定的加密算法
openssl s_client -connect www.baidu.com:443 -tls1_2 -cipher 'ECDHE-RSA-AES128-GCM-SHA256'
```
