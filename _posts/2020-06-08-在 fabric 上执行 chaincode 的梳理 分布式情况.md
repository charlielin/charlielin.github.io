---
layout:     post
title:      在 fabric 上执行 chaincode 的梳理 分布式情况
date:       2020-06-08
author:     Charlie Lin
catalog:    true
tags:
    - hyperledger/fabric
---
[toc]

# 在 fabric 上执行 chaincode 的梳理 分布式情况

本文继续梳理一些更复杂的情况。
原来的例子中，两个 org，每个 org 下只有一个 peer 节点，且都位于同一太机器上。通过修改官方提供的 shell 脚本，每个 org 下各增加两个 peer 节点，并且分别位于两台机器上。本文主要讨论在多个 peer 节点的情况下提交 chaincode 的操作。

## 组织结构

* longzhou-org1：peer0.org1.nd.com.cn，peer1.org1.nd.com.cn，peer2.org1.nd.com.cn
* longzhou-org2：peer0.org2.nd.com.cn，peer1.org2.nd.com.cn，peer2.org1.nd.com.cn
* longzhou-orderer：orderer.nd.com.cn

## 打包 chaincode 程序
与单机相同，略过


## install 程序
```
mySetGlobals 1 0
peer lifecycle chaincode install cp.tar.gz

mySetGlobals 1 1
peer lifecycle chaincode install cp.tar.gz

mySetGlobals 1 2
peer lifecycle chaincode install cp.tar.gz

mySetGlobals 2 0
peer lifecycle chaincode install cp.tar.gz

mySetGlobals 2 1
peer lifecycle chaincode install cp.tar.gz

mySetGlobals 2 2
peer lifecycle chaincode install cp.tar.gz
```
**注意**
每次执行 install 后，执行 `peer lifecycle chaincode queryinstalled` 检查安装结果。

## approve & commit

经过尝试，由于两个组织的 peer 节点分别位于两台机器上，需要分别尝试提交，并使用 sequence 作为区别。

### 提交 magnetocorp

**注意**：这里的 sequence 为 1 
```shell
mySetGlobals 2 0
export PACKAGE_ID=$(peer lifecycle chaincode queryinstalled --output json | jq -r '.installed_chaincodes[0].package_id')
echo $PACKAGE_ID

peer lifecycle chaincode approveformyorg  --orderer orderer.nd.com.cn:7050 --ordererTLSHostnameOverride orderer.nd.com.cn \
        --channelID mychannel  \
        --name papercontract  \
        -v 0  \
        --package-id $PACKAGE_ID \
        --sequence 1  \
        --tls  \
        --cafile $ORDERER_TLS_ROOTCERT_FILE
```
执行结果：
```
cp_1:2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1
2020-06-08 16:53:32.171 CST [chaincodeCmd] ClientWait -> INFO 001 txid [4c59280282d41e46704b32a6933ece4d4bd6dc5dbb403db3ea7b09cef9220644] committed with status (VALID) at
``` 
**换另一个组织继续 approve**
```shell
mySetGlobals 1 0
export PACKAGE_ID=$(peer lifecycle chaincode queryinstalled --output json | jq -r '.installed_chaincodes[0].package_id')
echo $PACKAGE_ID

peer lifecycle chaincode approveformyorg  --orderer orderer.nd.com.cn:7050 --ordererTLSHostnameOverride orderer.nd.com.cn \
        --channelID mychannel  \
        --name papercontract  \
        -v 0  \
        --package-id $PACKAGE_ID \
        --sequence 1  \
        --tls  \
        --cafile $ORDERER_TLS_ROOTCERT_FILE
```
执行结果：
```
cp_1:2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1
2020-06-08 16:53:32.171 CST [chaincodeCmd] ClientWait -> INFO 001 txid [4c59280282d41e46704b32a6933ece4d4bd6dc5dbb403db3ea7b09cef9220644] committed with status (VALID) at
```

切回 org2，继续提交程序
```shell
mySetGlobals 2 0
peer lifecycle chaincode commit -o orderer.nd.com.cn:7050 \
      --peerAddresses peer0.org1.nd.com.cn:7051 --tlsRootCertFiles ${PEER0_ORG1_CA} \
      --peerAddresses peer1.org1.nd.com.cn:9051 --tlsRootCertFiles ${PEER1_ORG1_CA} \
      --peerAddresses peer2.org1.nd.com.cn:11051 --tlsRootCertFiles ${PEER2_ORG1_CA} \
      --peerAddresses peer0.org2.nd.com.cn:7051 --tlsRootCertFiles ${PEER0_ORG2_CA} \
      --peerAddresses peer1.org2.nd.com.cn:9051 --tlsRootCertFiles ${PEER1_ORG2_CA} \
      --peerAddresses peer2.org2.nd.com.cn:11051 --tlsRootCertFiles ${PEER2_ORG2_CA} \
      --ordererTLSHostnameOverride orderer.nd.com.cn \
      --channelID mychannel --name papercontract -v 0 \
      --sequence 1 \
      --tls --cafile $ORDERER_CA --waitForEvent
```

执行结果：
```
2020-06-08 16:55:24.607 CST [chaincodeCmd] ClientWait -> INFO 001 txid [c0ba2bce27726e13580f620297e1669caab6696235a97504f74c99a125f742ea] committed with status (VALID) at peer2.org1.nd.com.cn:11051
2020-06-08 16:55:24.635 CST [chaincodeCmd] ClientWait -> INFO 002 txid [c0ba2bce27726e13580f620297e1669caab6696235a97504f74c99a125f742ea] committed with status (VALID) at peer1.org2.nd.com.cn:9051
2020-06-08 16:55:24.639 CST [chaincodeCmd] ClientWait -> INFO 003 txid [c0ba2bce27726e13580f620297e1669caab6696235a97504f74c99a125f742ea] committed with status (VALID) at peer0.org1.nd.com.cn:7051
2020-06-08 16:55:24.671 CST [chaincodeCmd] ClientWait -> INFO 004 txid [c0ba2bce27726e13580f620297e1669caab6696235a97504f74c99a125f742ea] committed with status (VALID) at peer1.org1.nd.com.cn:9051
2020-06-08 16:55:25.034 CST [chaincodeCmd] ClientWait -> INFO 005 txid [c0ba2bce27726e13580f620297e1669caab6696235a97504f74c99a125f742ea] committed with status (VALID) at peer0.org2.nd.com.cn:7051
2020-06-08 16:55:25.058 CST [chaincodeCmd] ClientWait -> INFO 006 txid [c0ba2bce27726e13580f620297e1669caab6696235a97504f74c99a125f742ea] committed with status (VALID) at peer2.org2.nd.com.cn:11051
```

执行完毕后，执行 `docker ps -a` 命令，观察是否有 chaincode 节点的容器存在，如下所示：
```shell
CONTAINER ID        IMAGE                                                                                                                                                             COMMAND                  CREATED             STATUS                  PORTS                                        NAMES
d3e2c3c7ebc3        dev-peer0.org2.nd.com.cn-cp_1-2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1-3cadd3418cec9fd77facaa85e337974a6512bc3065fbe457c68250026f72f3c3   "/root/chaincode-jav…"   4 seconds ago       Created                                                              dev-peer0.org2.nd.com.cn-cp_1-2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1
4a9c94070e49        dev-peer2.org2.nd.com.cn-cp_1-2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1-d910e322699f5639997abf130d99b2f09030732d8d4ccf6bbafa85ca2d6a3233   "/root/chaincode-jav…"   4 seconds ago       Created                                                              dev-peer2.org2.nd.com.cn-cp_1-2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1
ffa2a1252c74        dev-peer1.org2.nd.com.cn-cp_1-2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1-0ffdcb981fe804c1b7ab40b491d144f1588c068d15a7d29918f6f07119633455   "/root/chaincode-jav…"   4 seconds ago       Up Less than a second                                                dev-peer1.org2.nd.com.cn-cp_1-2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1
418f535892d2        hyperledger/fabric-peer:latest
```

### 提交 digibank
**注意**：在 digibank 处理时，sequence 为 2 
```shell
mySetGlobals 1 0
export PACKAGE_ID=$(peer lifecycle chaincode queryinstalled --output json | jq -r '.installed_chaincodes[0].package_id')
echo $PACKAGE_ID

peer lifecycle chaincode approveformyorg  --orderer orderer.nd.com.cn:7050 --ordererTLSHostnameOverride orderer.nd.com.cn \
        --channelID mychannel  \
        --name papercontract  \
        -v 0  \
        --package-id $PACKAGE_ID \
        --sequence 2  \
        --tls  \
        --cafile $ORDERER_TLS_ROOTCERT_FILE
```
执行结果：
```
cp_1:2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1
2020-06-08 16:53:32.171 CST [chaincodeCmd] ClientWait -> INFO 001 txid [4c59280282d41e46704b32a6933ece4d4bd6dc5dbb403db3ea7b09cef9220644] committed with status (VALID) at
``` 
**换另一个组织继续 approve**
```shell
mySetGlobals 2 0
export PACKAGE_ID=$(peer lifecycle chaincode queryinstalled --output json | jq -r '.installed_chaincodes[0].package_id')
echo $PACKAGE_ID

peer lifecycle chaincode approveformyorg  --orderer orderer.nd.com.cn:7050 --ordererTLSHostnameOverride orderer.nd.com.cn \
        --channelID mychannel  \
        --name papercontract  \
        -v 0  \
        --package-id $PACKAGE_ID \
        --sequence 2  \
        --tls  \
        --cafile $ORDERER_TLS_ROOTCERT_FILE
```
执行结果：
```
cp_1:2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1
2020-06-08 16:53:32.171 CST [chaincodeCmd] ClientWait -> INFO 001 txid [4c59280282d41e46704b32a6933ece4d4bd6dc5dbb403db3ea7b09cef9220644] committed with status (VALID) at
```

**切回 org1，提交程序**
```shell
mySetGlobals 1 0
peer lifecycle chaincode commit -o orderer.nd.com.cn:7050 \
      --peerAddresses peer0.org1.nd.com.cn:7051 --tlsRootCertFiles ${PEER0_ORG1_CA} \
      --peerAddresses peer1.org1.nd.com.cn:9051 --tlsRootCertFiles ${PEER1_ORG1_CA} \
      --peerAddresses peer2.org1.nd.com.cn:11051 --tlsRootCertFiles ${PEER2_ORG1_CA} \
      --peerAddresses peer0.org2.nd.com.cn:7051 --tlsRootCertFiles ${PEER0_ORG2_CA} \
      --peerAddresses peer1.org2.nd.com.cn:9051 --tlsRootCertFiles ${PEER1_ORG2_CA} \
      --peerAddresses peer2.org2.nd.com.cn:11051 --tlsRootCertFiles ${PEER2_ORG2_CA} \
      --ordererTLSHostnameOverride orderer.nd.com.cn \
      --channelID mychannel --name papercontract -v 0 \
      --sequence 2 \
      --tls --cafile $ORDERER_CA --waitForEvent
```

执行结果：
```
2020-06-08 16:55:24.607 CST [chaincodeCmd] ClientWait -> INFO 001 txid [c0ba2bce27726e13580f620297e1669caab6696235a97504f74c99a125f742ea] committed with status (VALID) at peer2.org1.nd.com.cn:11051
2020-06-08 16:55:24.635 CST [chaincodeCmd] ClientWait -> INFO 002 txid [c0ba2bce27726e13580f620297e1669caab6696235a97504f74c99a125f742ea] committed with status (VALID) at peer1.org2.nd.com.cn:9051
2020-06-08 16:55:24.639 CST [chaincodeCmd] ClientWait -> INFO 003 txid [c0ba2bce27726e13580f620297e1669caab6696235a97504f74c99a125f742ea] committed with status (VALID) at peer0.org1.nd.com.cn:7051
2020-06-08 16:55:24.671 CST [chaincodeCmd] ClientWait -> INFO 004 txid [c0ba2bce27726e13580f620297e1669caab6696235a97504f74c99a125f742ea] committed with status (VALID) at peer1.org1.nd.com.cn:9051
2020-06-08 16:55:25.034 CST [chaincodeCmd] ClientWait -> INFO 005 txid [c0ba2bce27726e13580f620297e1669caab6696235a97504f74c99a125f742ea] committed with status (VALID) at peer0.org2.nd.com.cn:7051
2020-06-08 16:55:25.058 CST [chaincodeCmd] ClientWait -> INFO 006 txid [c0ba2bce27726e13580f620297e1669caab6696235a97504f74c99a125f742ea] committed with status (VALID) at peer2.org2.nd.com.cn:11051
```

执行完毕后，执行 `docker ps -a` 命令，观察是否有 chaincode 节点的容器存在，如下所示：
```shell
CONTAINER ID        IMAGE                                                                                                                                                             COMMAND                  CREATED             STATUS                  PORTS                                        NAMES
d3e2c3c7ebc3        dev-peer0.org2.nd.com.cn-cp_1-2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1-3cadd3418cec9fd77facaa85e337974a6512bc3065fbe457c68250026f72f3c3   "/root/chaincode-jav…"   4 seconds ago       Created                                                              dev-peer0.org2.nd.com.cn-cp_1-2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1
4a9c94070e49        dev-peer2.org2.nd.com.cn-cp_1-2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1-d910e322699f5639997abf130d99b2f09030732d8d4ccf6bbafa85ca2d6a3233   "/root/chaincode-jav…"   4 seconds ago       Created                                                              dev-peer2.org2.nd.com.cn-cp_1-2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1
ffa2a1252c74        dev-peer1.org2.nd.com.cn-cp_1-2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1-0ffdcb981fe804c1b7ab40b491d144f1588c068d15a7d29918f6f07119633455   "/root/chaincode-jav…"   4 seconds ago       Up Less than a second                                                dev-peer1.org2.nd.com.cn-cp_1-2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1
418f535892d2        hyperledger/fabric-peer:latest
```

### 校验提交结果 

同单机模式