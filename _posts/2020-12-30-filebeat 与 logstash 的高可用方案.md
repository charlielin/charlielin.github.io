---
layout:     post
title:      filebeat 与 logstash 聚合的高可用方案
date:       2020-12-31
author:     Charlie Lin
catalog:    true
tags:
    - SQL
---
# filebeat 与 logstash 聚合的高可用方案

## 起因
在目前虚幻魔域的项目中，需要采集各个服务器上的所有日志（主要是登录日志），然后统一进行分析。 
其中最主要的是玩家登录分析。在游戏逻辑中，一个登录分为若干个子过程，举个例子，玩家首先要打开客户端输入帐密，在登录服务器端产生一条 LoginAccountValidation 日志，校验成功后，再校验客户端版本，在登录服务器产生一条 CheckVersion 日志，接着，需要分配角色到对应的游戏服务器，在游服产生一条 AssignPlayerToGS 日志，然后接着是确认客户端连接到对应的游服，记录一条 ClientConnectToGS 日志。这样玩家的一次登录过程，在登录服务器、游戏服务器分别产生了 4 条日志，记录了此次完整的登录过程。  
为了将分散在各个服务器的日志集中收集起来，产生一条完整的玩家登录日志，我们在游服、登录服各部署一个 filebeat 进行采集日志，每一个登录过程都用一个 transID 唯一标识。采集后的日志发送到 logstash 进行聚合，logstash 会将所有相同 transID 的日志聚合起来产生一条记录了完整登录过程的日志。整个过程如图所示：
![](https://tva1.sinaimg.cn/large/0081Kckwly1gm5ybk2t2fj30tf0gyq4e.jpg)  
使用这个方式简单明了，在小数据量时也能满足基本功能。但是次架构的缺点也很明显，一方面，随着需要采集的服务器不断增多，日志数量不断增大（目前虚幻魔域在开发阶段就已经累计了亿级的日志），单个 logstash 的聚合处理能力有限，logstash 将成为整个系统的瓶颈；另一方面，一旦这个 logstash 奔溃退出，整个系统就将无法正常运行，同时 filebeat 采集的数据得不到及时消费，造成数据丢失。

## 解决方案
### 引入 kafka
为了解决当 logstash 宕机时，数据丢失的问题，在 filebeat 与 logstash 中间加入 kafka 做为消息中间件。
在 filebeat 的配置中设定如下：
![](https://tva1.sinaimg.cn/large/0081Kckwly1gm60yd6lsaj30yo01v0su.jpg)
加入 kafka 的优点如下：
1. 当logstash 宕机时，数据可以存储在 kafka 中，等待 logstash 恢复后再做处理；
2. 当logstash 聚合压力较大，处理不及时的时候，kafka 可以作为 filebeat 与 logstash 之间的缓冲，避免因为 logstash 处理不及时造成数据丢失；
3. 为使用 logstash 集群进行处理提供基础。
### 单机 logstash 升级为logstash 集群 
将原有的一个 logstash 升级为多个 logstash 协同工作，使用并行读取的方式，多个 logstash 同时读取 kafka 中的数据，一方面大大提高了吞吐量；另一方面，当其中某台 logstash 宕机时，其余机器可以保证系统继续运行。
## 难点说明
如果只是单纯的添加 logstash 服务数量，在我们的场景下是不可行的。因为我们的日志是要在 logstash 中进行聚合的，也就是消息是有状态的，上下文有关的。在单机 logstash 环境下，只有一个 logstash 在工作，因而具有相同 transID 的数据一定会进入同一个 logstash，而在分布式 logstash 环境下却并非如此。默认情况下，从 topic 中读取出的数据，会随机进入 logstash 集群中的任意一个。  
要做到使得具有相同的 transID 的数据进入同一个 logstash，我们首先要弄懂 filebeat -> kafka -> logstash 是如何配合工作的。  
### 所有相同 transID 的数据 如何进入同一个 logstash
先看 kafka 与 logstash 集群（这里 logstash 集群使用同一个 consumer name），在 kafka 看来，他们属于同一个 consumer group，下面讨论的情况都是所有 logstash 服务都属于同一个 consumer group 时的情况：
1. 假设从 kafka 读取的 topic 有 3 个 partition，同时有 4 个 logstash，即 num(kafkaPartition) < num(logstash)，此时会有 3 个 logstash 正在工作，分别读取一个 partition 中的数据，还剩一个 logstash 待机中，当 3 个工作中的 logstash 中的任意一个宕机，它会立刻补上。
2. 假设从 kafka 读取的 topic 有 3 个 partition，同时有 3 个 logstash，即 num(kafkaPartition) = num(logstash)，此时所有的 logstash 都在工作中，分别读取一个 partition 中的数据。当 logstash 发生宕机时，宕机 logstash 所负责的 partition 会被其他 logstash 接管，即一个 logstash 处理一个分区，另一个 logstash 处理两个分区。
3. 假设从 kafka 读取的 topic 有 3 个 partition，此时有 2 个 logstash，即 num(kafkaPartition) > num(logstash)，这其实就是上述第二点发生 logstash 宕机时的情况。

由以上可知， kafka 中的一个 partition 的数据，只会发往 consumer group 中的一个 logstash服务。基于这个原理，只要保证具有相同 transID 的数据都能发往同一个 partition，就能保证具有相同 transID 的数据一定会发往同一个 logstash，继而在同一个 logstash 中进行聚合。kafka 与 logstash 集群的关系如图所示：
![](https://tva1.sinaimg.cn/large/0081Kckwly1gmbr9dbes9j312x0m5783.jpg)
### 所有相同 transID 的数据如何进入同一个 kafka partition
再来看 filebeat 与 kafka 的情况。默认情况下，filebeat 的数据发往 partition 时，是使用 random 的方式，任何一个partition 都有可能成为消息的目的地。为了使相同 transID 的数据进入同一个kafka partition，很自然的想到我们可以以 transID 的值作为 key 进行哈希，相同哈希结果的消息进入同一个 kafka partition。查看 filebeat 的源码，提供了对先对消息的 key 计算哈希值，再根据哈希值对分区数取模的方式确定分区：
![](https://tva1.sinaimg.cn/large/0081Kckwly1gmbssplr76j30fj0dnmym.jpg)
![](https://tva1.sinaimg.cn/large/0081Kckwly1gmbsu9t1v8j30ga04kmxe.jpg)

对应的 filebeat 配置如下：
![](https://tva1.sinaimg.cn/large/0081Kckwly1gmbsv5mq96j30yk05edgb.jpg)
如图所示，根据字段 transID 做哈希确定分区。   
这样 filebeat 与kafka partition 之间的关系如下：
![](https://tva1.sinaimg.cn/large/0081Kckwly1gmbt5ftkoyj312y0m6jur.jpg)

### 整体架构
将 kafka 与 logstash 集群的连接与 kafka 与 filebeat 的连接整合在一起，形成的整体架构如下：
![](https://tva1.sinaimg.cn/large/0081Kckwly1gmbtpms840j312x0m5tc8.jpg)

## 创新点说明
利用开源架构的灵活组合，快速搭配整合出一套可以实现日志消息聚合的 filebeat+logstash 的分布式与高可用架构。

## 创新的受益群体
所有使用 ELK 进行日志收集，同时有消息聚合场景需求的项目。

## 创新收益评估
将原本单机无容错的 logstash 聚合服务扩充为分布式高可用的聚合服务。在提高了服务吞吐量的同时，增加了服务的可靠性。
