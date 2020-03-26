---
layout:     post
title:      在 fabric 上执行 chaincode 的梳理
date:       2020-03-26
author:     Charlie Lin
catalog:    true
tags:
    - hyperledger/fabric
---


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

在第二个 terminal 中执行：
```shell
cd fabric-samples/commercial-paper/organizations/digibank
./digibank.sh
```

## 打包 chaincode 程序
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
**执行过程中会有好几次报错：peer0.org1.example.com|2020-03-26 14:56:05.514 UTC [endorser] SimulateProposal -> ERRO 05a failed to invoke chaincode _lifecycle, error: timeout expired while executing transaction 多试几次可以成功**
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