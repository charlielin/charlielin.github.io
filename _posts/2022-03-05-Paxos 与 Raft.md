---
layout:     post
title:      PAXOS 与 RAFT
date:       2022-03-05
author:     Charlie Lin
catalog:    true
tags:
    - PAXOS
    - RAFT
    - 一致性算法
---
## BASIC PAXOS

### 角色

* client： 客户端，系统之外的角色
* Proposer：Client 代理人，接收 Client 的提案发送给 Acceptor
* Acceptor：投票者
* Learner：记录员，记录投票结果

### 阶段

#### Prepare

确定提案号。  

1. 由 Proposer 发起 prepare 请求，发起一个全局单调自增的提案号 N 给 Acceptor。  
2. 当 N 大于 Acceptor 之前接收到的所有提案号时，返回接受。  
3. 当超过半数的 Acceptor 接受请求，则提案号 N 通过。

#### Accept

确定具体的值。

1. 由 Proposer 发起 Accept 请求，内容包括提案 N 以及一个具体的值 V 给 Acceptor。
2. 当 N 依旧是最大的提案号时，接受 Acceptor 请求。
3. 当超过半数的 Acceptor 接受请求，V 值通过。
4. Learner 记录 V 值。
5. 返回结果给客户端。  

## RAFT

可以查看一个可视化的 Raft 演示动画：https://raft.github.io。

### 角色

![](https://tva1.sinaimg.cn/large/e6c9d24ely1gzzfpx79e5j20y00eyac1.jpg)

* Leader：由 Candidate 投票选举产生，得到超过半数的投票
* Follower：节点初始状态，追随一个 Leader
* Candidate：与 Leader 超时后，进入 Candidate  

### 阶段

#### Leader Election

先由 Candidate 选举得到一个 leader，在执行后序的。

#### Log Replication

1. 由 Leader 发起复制请求。每个请求同样带上 Leader 任期 Term，提案号 N，值 V。
2. Follower 收到请求后，记录 Term，N，V。但此时并没有写入磁盘。返回 OK。
3. Follower 收到过半的 OK 后，将 V 值写入磁盘，同时发起写确认请求。
4. Follower 收到写确认请求后，将 V 值写入磁盘。