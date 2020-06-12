---
layout:     post
title:      在分布式部署的 fabric 网络上运行 chaincode 程序
date:       2020-05-26
author:     Charlie Lin
catalog:    true
tags:
     - hyperledger/fabric
---

# 在分布式部署的 fabric 网络上运行 chaincode 程序

本示例介绍如何在分布式部署的 fabric 网络上运行 commercial paper 程序。

## 打包程序

在 digibank 和 magentocorp 组织（分别对应 org1 和 org2） 窗口下分别运行：
```shell
peer lifecycle chaincode package cp.tar.gz --lang java --path ./contract-java --label cp_0
``` 

## 安装程序

在 digibank 和 magentocorp 组织（分别对应 org1 和 org2） 窗口下分别运行：

```shell
mySetGlobals 1 0
peer lifecycle chaincode install cp.tar.gz

mySetGlobals 1 1
peer lifecycle chaincode install cp.tar.gz

mySetGlobals 2 0
peer lifecycle chaincode install cp.tar.gz

mySetGlobals 2 1
peer lifecycle chaincode install cp.tar.gz
``` 
> 注：这里要在每一个 peer 节点都进行 install，否则后期会报 chaincode 不存在

## 检查安装结果

```shell
export PACKAGE_ID=$(peer lifecycle chaincode queryinstalled --output json | jq -r '.installed_chaincodes[0].package_id')
echo $PACKAGE_ID
```

## approve 程序
在 digibank 和 magentocorp 上都任意选一个 peer 执行：
```shell
peer lifecycle chaincode approveformyorg  --orderer localhost:7050 --ordererTLSHostnameOverride orderer.nd.com.cn \
     --channelID mychannel  \
     --name papercontract  \
     -v 0  \
     --package-id $PACKAGE_ID \
     --sequence 1  \
     --tls  \
     --cafile $ORDERER_TLS_ROOTCERT_FILE

```
## 检查状态

在 任意一个 terminal 中执行：
```shell
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name papercontract -v 0 --sequence 1
``` 
如果两个都是组织都是显示 true，则可以继续下一步。

## commit 程序
在 digibank 和 magentocorp 的任意一个 peer 上执行：
```shell
peer lifecycle chaincode commit -o orderer.nd.com.cn:7050 \
     --peerAddresses peer0.org1.nd.com.cn:7051 --tlsRootCertFiles ${PEER0_ORG1_CA} \
     --peerAddresses peer1.org1.nd.com.cn:9051 --tlsRootCertFiles ${PEER1_ORG1_CA} \
     --peerAddresses peer0.org2.nd.com.cn:7051 --tlsRootCertFiles ${PEER0_ORG2_CA} \
     --peerAddresses peer1.org2.nd.com.cn:9051 --tlsRootCertFiles ${PEER1_ORG2_CA} \
     --ordererTLSHostnameOverride orderer.nd.com.cn \
     --channelID mychannel --name papercontract -v 0 \
     --sequence 1 \
     --tls --cafile $ORDERER_CA --waitForEvent
```

## 测试程序提交情况

### 测试一
```shell
peer chaincode invoke -o orderer.nd.com.cn:7050  --ordererTLSHostnameOverride orderer.nd.com.cn \
     --peerAddresses peer0.org1.nd.com.cn:7051 --tlsRootCertFiles ${PEER0_ORG1_CA} \
     --peerAddresses peer1.org1.nd.com.cn:9051 --tlsRootCertFiles ${PEER1_ORG1_CA} \
     --peerAddresses peer0.org2.nd.com.cn:7051 --tlsRootCertFiles ${PEER0_ORG2_CA} \
     --peerAddresses peer1.org2.nd.com.cn:9051 --tlsRootCertFiles ${PEER1_ORG2_CA} \
     --channelID mychannel --name papercontract \
     -c '{"Args":["org.papernet.commercialpaper:instantiate"]}' ${PEER_ADDRESS_ORG1} ${PEER_ADDRESS_ORG2} \
     --tls --cafile $ORDERER_CA --waitForEvent
``` 
显示如下即为成功：
![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gf66z6s58qj319l08ajuf.jpg)

### 测试二
```shell
mySetGlobals 2 1

peer chaincode query -o orderer.nd.com.cn:7050  --ordererTLSHostnameOverride orderer.nd.com.cn \
     --channelID mychannel \
     --name papercontract \
     -c '{"Args":["org.hyperledger.fabric:GetMetadata"]}' \
     --peerAddresses peer1.org2.nd.com.cn:9051 --tlsRootCertFiles ${PEER1_ORG2_CA} \
     --tls --cafile $ORDERER_CA | jq -C
```

显示 json 内容即为正常。