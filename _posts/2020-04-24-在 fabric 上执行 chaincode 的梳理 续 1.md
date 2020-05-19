---
layout:     post
title:      在 fabric 上执行 chaincode 的梳理 续 1
date:       2020-04-24
author:     Charlie Lin
catalog:    true
tags:
    - hyperledger/fabric
---
[toc]

# 在 fabric 上执行 chaincode 的梳理 续 1

本文继续梳理一些更复杂的情况。
原来的例子中，两个 org，每个 org 下只有一个 peer 节点。通过修改官方提供的 shell 脚本，每个 org 下各增加一个 peer1 节点。本文主要讨论在多个 peer 节点的情况下提交 chaincode 的操作。

## 未在所有的 peer 节点上 install chaincode 程序
忽略 peer1，仅在 peer0 上 install chaincode 程序。这种情况下，commit 之后，在 peer1 上也可以通过 querycommitted 查询到提交的 chaincode 程序，但只有 peer0 才可以正确的执行程序。

在 peer1 上 querycommitted：
![](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge47vilgddj318u04ajs2.jpg)

在 peer1 上 执行，提示未 “is not installed”：
![](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge47z8oqolj32ks09sgom.jpg)

这个时候如果在 peer1 上执行 instal，再查询即正常：
![](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge480m7plpj32dm0b0n1p.jpg)