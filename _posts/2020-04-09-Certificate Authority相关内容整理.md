---
layout:     post
title:      Hyperledger Fabric:A Distributed Operating System forPermissioned Blockchains
date:       2020-04-07
author:     Charlie Lin
catalog:    true
tags:
    - CA
    - Certificate
    - 数字证书
    - 密码学
---

# Certificate Authority 相关内容整理

## 一些密码学基础知识

### 非对称密钥体系

* 公钥与私钥一一对应
* 公钥加密后，由私钥进行解密
* 私钥加密生成数字签名，由公钥解密来验证数字签名的合法性 

## 数字证书

数字证书是由第三方的权威、受信任的第三方 CA（Certificate Authority） 颁发的一个证书，用来证明某人的公钥是属于某人的，没有被人篡改过。
例如 Bob 的数字证书，最简单的内容包括：Bob 的身份信息，Bob 的公钥，以及CA 私钥加密过的数字签名。

## CA
Certificate Authority 是负责管理和签发证书的第三方机构。世界上的 受信任的 CA 就那么几家，他们自己的证书称为根证书，已经事先嵌入在浏览器内部了。

## 使用 CA 进行安全通信

当 Bob 要与 Alice 通信的时候，Bob 会先使用自己的私钥对消息内容做数字签名，发送：内容原文+ 数字签名 + Bob 的数字证书。当 Alice 拿到 Bob 的数字证书后，使用已经存在浏览器中的 CA 的公钥对证书里的数字签名进行校验，确认确实是 Bob 的公钥后，才用 Bob 的公钥对信息进行加密，与 Bob 进行通信。
