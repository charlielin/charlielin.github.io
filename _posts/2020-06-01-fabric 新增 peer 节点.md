---
layout:     post
title:      fabric 新增 peer 节点
date:       2020-06-01
author:     Charlie Lin
catalog:    true
tags:
    - hyperledger/fabric
---

# fabric 新增 peer 节点

在组织 org1 上新增节点 peer2
## 准备 fabric-ca 的密钥材料

```shell
# 登记 org1 的 admin 用户
fabric-ca-client enroll -u https://admin:adminpw@ca.org1.nd.com.cn:7054 --caname ca-org1 --tls.certfiles ${PWD}/organizations/fabric-ca/org1/tls-cert.pem

# 使用 org1 的证书生成 peer2 用户
fabric-ca-client register --caname ca-org1 --id.name peer2 --id.secret peer2pw --id.type peer --tls.certfiles ${PWD}/organizations/fabric-ca/org1/tls-cert.pem

# 生成 peer2 的 msp
fabric-ca-client enroll -u https://peer2:peer2pw@ca.org1.nd.com.cn:7054 --caname ca-org1 -M ${PWD}/organizations/peerOrganizations/org1.nd.com.cn/peers/peer2.org1.nd.com.cn/msp --csr.hosts peer2.org1.nd.com.cn --tls.certfiles ${PWD}/organizations/fabric-ca/org1/tls-cert.pem

cp ${PWD}/organizations/peerOrganizations/org1.nd.com.cn/msp/config.yaml ${PWD}/organizations/peerOrganizations/org1.nd.com.cn/peers/peer2.org1.nd.com.cn/msp/config.yaml

# 生成 peer2 的 tls-msp
fabric-ca-client enroll -u https://peer2:peer2pw@ca.org1.nd.com.cn:7054 --caname ca-org1 -M ${PWD}/organizations/peerOrganizations/org1.nd.com.cn/peers/peer2.org1.nd.com.cn/tls --enrollment.profile tls --csr.hosts peer2.org1.nd.com.cn --csr.hosts ca.org1.nd.com.cn --tls.certfiles ${PWD}/organizations/fabric-ca/org1/tls-cert.pem


cp ${PWD}/organizations/peerOrganizations/org1.nd.com.cn/peers/peer2.org1.nd.com.cn/tls/tlscacerts/* ${PWD}/organizations/peerOrganizations/org1.nd.com.cn/peers/peer2.org1.nd.com.cn/tls/ca.crt
cp ${PWD}/organizations/peerOrganizations/org1.nd.com.cn/peers/peer2.org1.nd.com.cn/tls/signcerts/* ${PWD}/organizations/peerOrganizations/org1.nd.com.cn/peers/peer2.org1.nd.com.cn/tls/server.crt
cp ${PWD}/organizations/peerOrganizations/org1.nd.com.cn/peers/peer2.org1.nd.com.cn/tls/keystore/* ${PWD}/organizations/peerOrganizations/org1.nd.com.cn/peers/peer2.org1.nd.com.cn/tls/server.key
```

## 启动 peer2 的 docker 服务

## 将 peer2 加入 mychannel

