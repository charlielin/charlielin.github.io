---
layout:     post
title:      在 fabric 上执行 chaincode 的梳理
date:       2020-03-26
author:     Charlie Lin
catalog:    true
tags:
    - hyperledger/fabric
---
[toc]

# 在 fabric 上执行 chaincode 的梳理

## 流程说明
以 commercial-paper 为例，梳理运行 chaincode 程序的流程。
在区块链上有两个组织——org1，org2。对应 digibank 与 magentocorp 这两家公司。magentocorp 为 paper 发行商，digibank 购买 paper，并在到期后赎回。
两家拥有一模一样的 chaincode 程序，在他们的网络环境中分别打包、验证并发布 chaincode 程序。最后各自使用不同的客户端程序分别完成发行、购买、赎回等动作。
这里要注意：两家公司使用相同的 chaincode 程序，但他们的客户端程序不相同。

## 准备环境变量
准备一个环境变量脚本 ~/.fabric.sh，用于切换环境变量以对应不同的 peer。
```shell
#! /bin/bash
setGlobals() {
  org=$1
  export ORDERER_TLS_ROOTCERT_FILE=${TEST_NETWORK}/organizations/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem
  if [ $org -eq 1 ];then
    export CORE_PEER_LOCALMSPID="Org1MSP"
    export CORE_PEER_TLS_ROOTCERT_FILE=${TEST_NETWORK}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
    export CORE_PEER_MSPCONFIGPATH=${TEST_NETWORK}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
    export CORE_PEER_ADDRESS=localhost:7051
  elif [ $org -eq 2 ]; then
    export CORE_PEER_LOCALMSPID="Org2MSP"
    export CORE_PEER_TLS_ROOTCERT_FILE=${TEST_NETWORK}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
    export CORE_PEER_MSPCONFIGPATH=${TEST_NETWORK}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
    export CORE_PEER_ADDRESS=localhost:9051
  else
    echo "need parameter!!!"
  fi
}
```
```shell
. ~/.fabric.sh
```

在第一个 terminal 中执行：
```shell
cd fabric-samples/commercial-paper/organizations/magentocorp
./magnetocorp.sh
```
把输出的内容复制到命令行中 export

在第二个 terminal 中执行：
```shell
cd fabric-samples/commercial-paper/organizations/digibank
./digibank.sh
```
把输出的内容复制到命令行中 export

## 打包 chaincode 程序
### for node.js
在第一个 terminal 中运行：
```shell
# setGlobals 2
peer lifecycle chaincode package cp.tar.gz --lang node --path ./contract --label cp_0
```
注意：在执行前要先设置 peer2 的环境变量

在第二个 terminal 中运行：
```shell
# setGlobals 1
peer lifecycle chaincode package cp.tar.gz --lang node --path ./contract --label cp_0
```

### for java
在打包前，先 build（注：实际操作中发现不用先 build，因为上传的是源码，在服务端 install 的时候会 build 一次）：
```shell
cd ./organizations/magentocorp/contract-java
./gradlew build
```
再打包：
```shelll
cd ..
peer lifecycle chaincode package cp.tar.gz --lang java --path ./contract-java --label cp_0
```
其余步骤与 node.js 一样。

## 安装 chaincode 程序
在第一个 terminal 中运行：
```shell
# setGlobals 2
peer lifecycle chaincode install cp.tar.gz
```

在第二个 terminal 中运行：
```shell
# setGlobals 1
peer lifecycle chaincode install cp.tar.gz
```
**报错**
执行过程中会有好几次报错：peer0.org1.example.com|2020-03-26 14:56:05.514 UTC [endorser] SimulateProposal -> ERRO 05a failed to invoke chaincode _lifecycle, error: timeout expired while executing transaction
这是客户端在得到服务端的回复之前就断开了连接的缘故。这里可以修改 core.yaml 中的 peer.keepalive.client.timeout 来修改超时时间。

分别在两个 terminal 中执行：
```shell
> peer lifecycle chaincode queryinstalled
Installed chaincodes on peer:
Package ID: cp_0:07fd097779797941c75b559b0c552b5335cc7fca95a098cb566cfb49673a6e76, Label: cp_0
```
说明已经安装成功。

## approve the chaincode definition for my organizations
分别在两个 terminal 中运行：
```shell
export PACKAGE_ID=$(peer lifecycle chaincode queryinstalled --output json | jq -r '.installed_chaincodes[0].package_id')
echo $PACKAGE_ID

peer lifecycle chaincode approveformyorg  --orderer localhost:7050 --ordererTLSHostnameOverride orderer.example.com \
                                          --channelID mychannel  \
                                          --name papercontract  \
                                          -v 0  \
                                          --package-id $PACKAGE_ID \
                                          --sequence 1  \
                                          --tls  \
                                          --cafile $ORDERER_TLS_ROOTCERT_FILE
```

## 检查 chaincode 状态
在第一个 terminal 中运行：

```shell
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name papercontract -v 0 --sequence 1

Chaincode definition for chaincode 'papercontract', version '0', sequence '1' on channel 'mychannel' approval status by org:
Org1MSP: false
Org2MSP: true

显示 chaincode 已部署在 Org2MSP 中。
```

在第二个 terminal 中运行：
```shell
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name papercontract -v 0 --sequence 1

Chaincode definition for chaincode 'papercontract', version '0', sequence '1' on channel 'mychannel' approval status by org:
Org1MSP: true
Org2MSP: true
```

### 提交 chaincode
当两个组织都已安装了chaincode，并通过了认证后，可以提交 chaincode。
```shell
# 第一个 terminal 中 

peer lifecycle chaincode commit -o localhost:7050 \
                                --peerAddresses localhost:7051 --tlsRootCertFiles ${PEER0_ORG1_CA} \
                                --peerAddresses localhost:9051 --tlsRootCertFiles ${PEER0_ORG2_CA} \
                                --ordererTLSHostnameOverride orderer.example.com \
                                --channelID mychannel --name papercontract -v 0 \
                                --sequence 1 \
                                --tls --cafile $ORDERER_CA --waitForEvent
```

### 测试 chaincode
测试一：
```shell
peer chaincode invoke -o localhost:7050  --ordererTLSHostnameOverride orderer.example.com \
                                --peerAddresses localhost:7051 --tlsRootCertFiles ${PEER0_ORG1_CA} \
                                --peerAddresses localhost:9051 --tlsRootCertFiles ${PEER0_ORG2_CA} \
                                --channelID mychannel --name papercontract \
                                -c '{"Args":["org.papernet.commercialpaper:instantiate"]}' ${PEER_ADDRESS_ORG1} ${PEER_ADDRESS_ORG2} \
                                --tls --cafile $ORDERER_CA --waitForEvent
```
返回结果：
```output
2020-03-26 23:34:11.965 CST [chaincodeCmd] ClientWait -> INFO 001 txid [327cd7698d849f5cd7aba5a7b39793c1f303cc21fbd4686270ed52a074fcbd2c] committed with status (VALID) at localhost:9051
2020-03-26 23:34:12.007 CST [chaincodeCmd] ClientWait -> INFO 002 txid [327cd7698d849f5cd7aba5a7b39793c1f303cc21fbd4686270ed52a074fcbd2c] committed with status (VALID) at localhost:7051
2020-03-26 23:34:12.007 CST [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 003 Chaincode invoke successful. result: status:20
```
 测试二：
```shell
peer chaincode query -o localhost:7050  --ordererTLSHostnameOverride orderer.example.com \
                                        --channelID mychannel \
                                        --name papercontract \
                                        -c '{"Args":["org.hyperledger.fabric:GetMetadata"]}' \
                                        --peerAddresses localhost:9051 --tlsRootCertFiles ${PEER0_ORG2_CA} \
                                        --tls --cafile $ORDERER_CA | jq -C
```
 返回结果：
```json
{
  "$schema": "https://fabric-shim.github.io/master/contract-schema.json",
  "contracts": {
    "org.papernet.commercialpaper": {
      "name": "org.papernet.commercialpaper",
      "contractInstance": {
        "name": "org.papernet.commercialpaper",
        "default": true
      },
      "transactions": [
        {
          "name": "instantiate",
          "tags": [
            "submitTx"
          ]
        },
        {
          "name": "issue",
          "tags": [
            "submitTx"
          ],
          "parameters": [
            {
              "name": "arg0",
              "description": "Argument 0",
              "schema": {
                "type": "string"
              }
            },
            {
              "name": "arg1",
              "description": "Argument 1",
              "schema": {
                "type": "string"
              }
            },
            {
              "name": "arg2",
              "description": "Argument 2",
              "schema": {
                "type": "string"
              }
            },
            {
              "name": "arg3",
              "description": "Argument 3",
              "schema": {
                "type": "string"
              }
            },
            {
              "name": "arg4",
              "description": "Argument 4",
              "schema": {
                "type": "string"
              }
            }
          ]
        },
        {
          "name": "buy",
          "tags": [
            "submitTx"
          ],
          "parameters": [
            {
              "name": "arg0",
              "description": "Argument 0",
              "schema": {
                "type": "string"
              }
            },
            {
              "name": "arg1",
              "description": "Argument 1",
              "schema": {
                "type": "string"
              }
            },
            {
              "name": "arg2",
              "description": "Argument 2",
              "schema": {
                "type": "string"
              }
            },
            {
              "name": "arg3",
              "description": "Argument 3",
              "schema": {
                "type": "string"
              }
            },
            {
              "name": "arg4",
              "description": "Argument 4",
              "schema": {
                "type": "string"
              }
            },
            {
              "name": "arg5",
              "description": "Argument 5",
              "schema": {
                "type": "string"
              }
            }
          ]
        },
        {
          "name": "redeem",
          "tags": [
            "submitTx"
          ],
          "parameters": [
            {
              "name": "arg0",
              "description": "Argument 0",
              "schema": {
                "type": "string"
              }
            },
            {
              "name": "arg1",
              "description": "Argument 1",
              "schema": {
                "type": "string"
              }
            },
            {
              "name": "arg2",
              "description": "Argument 2",
              "schema": {
                "type": "string"
              }
            },
            {
              "name": "arg3",
              "description": "Argument 3",
              "schema": {
                "type": "string"
              }
            }
          ]
        }
      ],
      "info": {
        "title": "",
        "version": ""
      }
    },
    "org.hyperledger.fabric": {
      "name": "org.hyperledger.fabric",
      "contractInstance": {
        "name": "org.hyperledger.fabric"
      },
      "transactions": [
        {
          "name": "GetMetadata"
        }
      ],
      "info": {
        "title": "",
        "version": ""
      }
    }
  },
  "info": {
    "version": "0.0.1",
    "title": "papernet-js"
  },
  "components": {
    "schemas": {}
  }
}
```

### 其他操作
查找已提交的 chaincode：
```shell
> peer lifecycle chaincode querycommitted -C mychannel
Committed chaincode definitions on channel 'mychannel':
Name: papercontract, Version: 0, Sequence: 1, Endorsement Plugin: escc, Validation Plugin: vscc
```
查找已安装的 chaincode：
（可以发现，在两个 peer 上分别执行，返回的 package id 不同）
```shell
# setGlobals 2
> peer lifecycle chaincode queryinstalled
Installed chaincodes on peer:
Package ID: cp_0:3d3227b96d0c8be52c6982cc103761d17235a4d74385f1f789ce5eed8811e2d6, Label: cp_0

# setGlobals 1
> peer lifecycle chaincode queryinstalled
Installed chaincodes on peer:
Package ID: cp_0:f004476a6d6d3fde2a356fea83c8885b385dd7948cd72b2a7a86b74586dac9eb, Label: cp_0
```
## commercial-paper 客户端
### magnetocorp 发行
```shell
> cd fabric-samples/commercial-paper/organization/magnetocorp/application
> npm install

> node issue.js
Connect to Fabric gateway.
Use network channel: mychannel.
Use org.papernet.commercialpaper smart contract.
Submit commercial paper issue transaction.
Process issue transaction response.{"class":"org.papernet.commercialpaper","key":"\"MagnetoCorp\":\"00001\"","currentState":1,"issuer":"MagnetoCorp","paperNumber":"00001","issueDateTime":"2020-05-31","maturityDateTime":"2020-11-30","faceValue":"5000000","owner":"MagnetoCorp"}
MagnetoCorp commercial paper : 00001 successfully issued for value 5000000
Transaction complete.
Disconnect from Fabric gateway.
Issue program complete.

> node issue.js
Connect to Fabric gateway.
Use network channel: mychannel.
Use org.papernet.commercialpaper smart contract.
Submit commercial paper issue transaction.
Process issue transaction response.{"class":"org.papernet.commercialpaper","key":"\"MagnetoCorp\":\"00001\"","currentState":1,"issuer":"MagnetoCorp","paperNumber":"00001","issueDateTime":"2020-05-31","maturityDateTime":"2020-11-30","faceValue":"5000000","owner":"MagnetoCorp"}
MagnetoCorp commercial paper : 00001 successfully issued for value 5000000
Transaction complete.
Disconnect from Fabric gateway.
Issue program complete.
node add
```

### digibank 购买与兑换
```shell
> cd fabric-samples/commercial-paper/organization/digibank/application
> npm install

> node addToWallet.js
done

> node buy.js
Connect to Fabric gateway.
Use network channel: mychannel.
Use org.papernet.commercialpaper smart contract.
Submit commercial paper buy transaction.
Process buy transaction response.
MagnetoCorp commercial paper : 00001 successfully purchased by DigiBank
Transaction complete.
Disconnect from Fabric gateway.
Buy program complete.

> node redeem.js
Connect to Fabric gateway.
Use network channel: mychannel.
Use org.papernet.commercialpaper smart contract.
Submit commercial paper redeem transaction.
Process redeem transaction response.
MagnetoCorp commercial paper : 00001 successfully redeemed with MagnetoCorp
Transaction complete.
Disconnect from Fabric gateway.
Redeem program complete.
```
## 更新 chaincode 程序
当 chaincode 程序提交运行后，可以对其进行更新。更新的 chaincode 程序需要与原来的程序具有相同的 chaincode 名字，并对 version 进行升级。
### 打包 chaincode 程序
```shell
 peer lifecycle chaincode package cp.tar.gz --lang java --path ./contract-java --label cp_1
```
注意这里的 label 相比之前发生了变化。
### 安装 chaincode 程序
``` shell
peer lifecycle chaincode install cp.tar.gz
```
这一步骤没有区别。

### approve the chaincode definition for my organizations
```shell
> peer lifecycle chaincode queryinstalled 
Installed chaincodes on peer:
Package ID: cp_0:07fd097779797941c75b559b0c552b5335cc7fca95a098cb566cfb49673a6e76, Label: cp_0
Package ID: cp_1:2da28088c9f13c6b64e877fc29ef811b506207ef1d344cf4ab343b4becb0d2a1, Label: cp_1
```
可以看到这里的多了一个 cp_1开头的 package，记下 Package ID
```shell
peer lifecycle chaincode approveformyorg  --orderer localhost:7050 --ordererTLSHostnameOverride orderer.example.com \
                                          --channelID mychannel  \
                                          --name papercontract  \
                                          -v 1  \
                                          --package-id $PACKAGE_ID \
                                          --sequence 2  \
                                          --tls  \
                                          --cafile $ORDERER_TLS_ROOTCERT_FILE
```
注意这里 `-v` 和 `--sequence` 都发生了变化，但 `--name` 必须保持一致。不然 fabric 会认为这是一个全新的程序。

### 检查 chaincode 状态
```shell
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name papercontract -v 1 --sequence 2
```
检查 version=1 sequence=2 的新版本程序的状态

### 提交更新后的程序
```shell
peer lifecycle chaincode commit -o localhost:7050 \
                                --peerAddresses localhost:7051 --tlsRootCertFiles ${PEER0_ORG1_CA} \
                                --peerAddresses localhost:9051 --tlsRootCertFiles ${PEER0_ORG2_CA} \
                                --ordererTLSHostnameOverride orderer.example.com \
                                --channelID mychannel --name papercontract -v 1 \
                                --sequence 2 \
                                --tls --cafile $ORDERER_CA --waitForEven
```
同样注意 version 与 sequence。
提交后，mychannel 上的 chaincode——papercontract 就更新完毕了。
