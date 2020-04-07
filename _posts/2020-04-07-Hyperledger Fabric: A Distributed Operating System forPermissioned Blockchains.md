---
layout:     post
title:      Hyperledger Fabric:A Distributed Operating System forPermissioned Blockchains
date:       2020-04-07
author:     Charlie Lin
catalog:    true
tags:
    - hyperledger/fabric
---

# Hyperledger Fabric：一个分布式的联盟链操作系统

> 本文由 Charlie Lin 简单翻译。

## 摘要
Fabric 是一个模块化、可扩展的，用于部署与操作联盟链的开源系统。隶属于 Hyperledger 项目。
Fabric 是第一个真正意义上的可用于运行分布式应用的可扩展区块链系统。支持模块化的共识协议。Fabric 也是第一个用标准的、通用的语言编写的运行分布式应用的区块链系统，不依赖于某些本地的加密货币。这与那些现有的需要使用领域特定语言（DSL，Domain Specific Language）编写智能合约或依赖于加密货币的区块链平台们形成鲜明的对比。Fabric 还引入了全新的区块链设计，使用全新的方式来应对区块链中的不确定性、资源耗尽、性能攻击等问题
Fabric 可以达到 3500笔交易每秒的端到端交易吞吐量，次秒级的延迟，超过 100 个节点的规模量
## 一、介绍
区块链可以定义为一个运行在节点之间互相不信任的分布式网络中的记录交易信息的不可篡改的账本。每个节点维护一个账本的拷贝。节点之间通过某种共识算法进行验证交易、组装区块、将区块加入区块链。这个过程中要通过对交易进行排序来形成账本，这是为了达到一致性所要做的。
在公链中，任何人都可以匿名的加入其中。公链需要引入加密货币并常常需要使用 PoW 共识算法，以及经济刺激。而联盟链的参与者是已知的，需要身份验证的。联盟链对一组需要彼此交流但又缺乏彼此信任的个体提供了一种保护。例如商业中交换基金、商品、信息。依赖于节点的身份，区块链可以使用传统的拜占庭容错公式算法。
区块链可以以智能合约的方式执行任意的可编程的逻辑，例如以太坊。一个智能合约是以受信任的分布式应用的方式运行，并从区块链与底层共识中获得安全性。这与通过状态机复制（state-machine replication，SMR）的方式来构建弹性应用极为相似。不同点在于：
1. 有多个分布式应用同时并发运行
2. 应用可以由任何人动态部署
3. 应用代码是不受信任的，甚至可能是潜在恶意的

以上不同点使得程序有必要进行重新设计。
许多现有的智能合约区块链程序都遵循 SMR 的蓝图实现所谓的动态复制（active replication）：一个共识的或者说原子广播（atomic broadcast）的协议，首先要对交易进行排序并向所有节点广播；然后所有节点接收到广播后，顺序执行每一笔交易。我们称其为排序-执行架构（order-execute architecture）。它要求所有的节点执行每一笔交易并且所有的交易都是确定的。实际上排序-执行架构存在于现有的各种区块链系统中，从公有链的以太坊（使用PoW 共识算法）到联盟链/私有链的 Tendermint，Chain，以及 Quorum。排序-执行架构有个固有的限制条件，即每个节点执行的交易必须是确定性的。  
先前的联盟链系统受限于许多的限制，这些限制通常来自于他们的permissionless relatives 或来自于采用的排序-执行架构。特别是：
* 共识算法采用硬编码的方式嵌入在平台中，这与我们普遍认为的没有一个普适的拜占庭容错共识协议相矛盾。
* 交易验证的信任模型是由共识协议决定的，不能自适应于智能合约的需求
* 智能合约是由某种特定的，非标准的领域特定语言编写，这妨碍了广泛传播或导致程序错误
* 全部节点都顺序执行所有交易，限制了性能
* 交易必须是确定的（例如以太坊使用 solidity 语言，Go 中的 Map iterator 就不满足确定性），这在程序上难以实现（实际上还是为了满足顺序执行的要求）
* 智能合约运行在系统的每一个节点上，这与保密性相冲突，禁止了合约代码在其中一个节点子集上运行。
  
本文描述的 Hyperledger Fabric 克服了以上限制。
Fabric 提出了一种新的区块链架构，着重于有弹性、灵活性、可扩展性与保密性。被设计为一个模块化与可扩展的通用准入区块链系统，区块链是首个支持用标准语言编写的区块链应用，这使得它们能够始终如一的在许多节点上运行，就像运行在一个全球化的分布式区块链计算机上一样。这使得 fabric 成为了第一个联盟链的分布式操作系统。
Fabric 的架构遵循一种新颖的为了在不受信任的环境上分布式执行的不受信任的代码的 execute-order-validate 模型。它把一个交易步骤分成三个部分：
1. 执行一个交易，检查结果，从而为它背书
2. 根据某种共识协议进行排序，不考虑交易语义
3. 交易验证

使用这种设计，Fabric 可以在达到最终一致之前先执行交易。两种复制方法，被动式与主动式：
1. Fabric 首先使用在分布式数据库中常用的 **主从复制（primary-backup replication）**，但是由于基于中间件的不对称更新处理已经移至到了一个不受信任的拜占庭容错环境。Fabric 中的每一笔交易都是由节点中的一部分子集来执行（背书），这就允许了交易的并行执行，以及解决潜在的不确定性问题。一个弹性的背书策略指定了哪些节点来担保给定智能合约的正确执行。
2. Fabric 引入主动复制机制，交易的副作用对账本状态的影响只能在共识打成之后才写入，在确定性验证这一步被每一个节点独立执行。进一步的，负责对状态更新排序达成共识的功能被赋予一个独立的模块组件，这些组件是无状态的，逻辑上与负责执行（背书）与维护账本的节点解耦的。既然共识模块是组件化的，它就可以根据不同的新人假设进行个性化设计。

为了实现上述的价格，Fabric 做了如下的模块设计：
* 排序服务（ordering service）：广播状态更新信息到各个节点并按交易的顺序确立共识
* 成员服务提供者（membership service provider）：负责联系节点间的加密身份，它维护着 Fabric 的本质。
* 可选的点到点的 gossip 服务：把排序服务输出的 block 广播到各个节点
* Fabric 中的智能合约运行在独立的容器环境中。他们是由普通的编程语言编写，但是不能直接访问账本状态信息
* 每个节点都在本地以 append-only 的区块链形式保存一份账本，并在 kv 数据库中保存最近的状态快照。

## 二、背景
###  2.1 区块链的Order-Execute 架构
先前的区块链系统，无论是公链还是联盟链，都是遵循 order-execute 架构。意外着系统要先对交易排序，再通过某种共识算法达成一致后，按顺序在各个节点中执行。
![](https://tva1.sinaimg.cn/large/006tNbRwgy1gb4c0trd3yj30m5082gmj.jpg)
例如，使用 PoW 共识机制的以太坊是按如下步骤达成共识并执行的：
1. 所有节点组装有效的交易信息（为了确认有效性，该节点会先做预执行）
2. 各个节点开始解PoW题
3. 某个节点解出谜题后，把它的 block 通过 gossip 协议传播到其他节点
4. 收到 block 的其他节点验证 PoW 谜题的正确性，并确认其中的交易，然后顺序执行交易

现有的区块链，例如 Tendermint，Chain，Quorum都是使用 BFT 共识，PBFT 或其他原子广播的协议。都遵循 order-execute 方式并实现经典的主动 SMR。
### 2.2 Order-Execute 的限制
1. 顺序执行。在所有的节点上都顺序执行交易限制了区块链系统的吞吐量。
2. 非确定性代码带来非确定性的交易。在获得共识之后在主动 SMR 中执行的操作必须是确定性的，否则不同的节点就会有不同的状态。通常，使用 DDL 语言编码可以解决这个问题（例如以太坊中的 Solidity）。
3. 保密执行。智能合约在所有节点上运行。
### 2.3 现有架构的其他限制
1. Fixed trust model。大多数的联盟链都是依赖异步 BFT 复制协议来确定共识。这些算法都有一个基本的安全假设：在一个 n > 3f 的节点中，可以允许最多 f 个恶意或故障节点。这些节点也在同样的安全假设下运行程序。然而，这种一视同仁的安全假设，不考虑节点在系统中的角色，可能不符合智能合约的执行。也就是不同类型的节点应更采用不同的安全策略。
2. hard-coded consensus。Fabric 是第一个实现了插件化共识机制的区块链系统。没有完全通用的共识机制。

## 三、架构
### 3.1 Fabric 概述
![](https://tva1.sinaimg.cn/large/006tNbRwgy1gb4c1f6q81j30mn0bcdhi.jpg)
Fabric 是一个联盟链的分布式操作系统，运行由通用语言（Go，Java，Node.js）编写的分布式程序。它安全地在一个 append-only 的自我复制的账本数据结构上记录执行历史，没有内建的加密货币。
Fabric 引入了 execute-order-validate 区块链架构。简而言之，一个 Fabric 上的分布式应用有两部分组成：
1. 智能合约——chaincode。实现了应用逻辑，在执行阶段运行。chaincode 是 Fabric 的核心，可以由不受信任的开发者编写。有一些特殊的负责系统管理与维护的 chaincode，统称为 system chaincode。
2. 背书策略。在 validation 阶段进行评估。背书策略不能由不受信任的开发者选择或修改。背书策略扮演的是交易验证的静态库角色，可以由 chaincode 参数化。只有管理员有权限对背书策略做出修改。

* 背书阶段：客户端发送交易信息到背书策略指定的节点。这些节点执行所有的交易，并记录输出。这一步骤也叫做背书。
* 排序阶段（ordering phase）：使用可插拔的共识机制对区块中的所有经过背书的交易进行排序，生成一个完全排序的区块。这些区块会被广播到所有节点。不同于标准的动态复制（active replication）对输入进行排序，Fabric 是对背书后的输出连同他们的状态依赖进行排序。
* 验证阶段（validation phase）：每个节点安装背书策略验证经过背书的交易的状态更新情况以及一致性情况。所有节点都按照同样的、确定的顺讯进行验证。
  
Fabric 在拜占庭模型中引入了创新的复合复制（replication）机制，结合了 passive replication（pre-consensus computation of state updates） 与 active replication（the post-consensus computation of state updates）。

Fabric 区块链的网络节点图如下：
![](https://tva1.sinaimg.cn/large/006tNbRwgy1gb5cso2mofj30ns0gidil.jpg)
由于 Fabric 是联盟链，因而其中所有的节点都有由 MSP（Membership Service Provider）提供的身份。Fabric 中的每个节点扮演如下三个角色的中一个：
* 客户端 Clients：提交交易提案
* 背书节点 endorsing peers，endorsers：不是所有的节点都执行交易，只有背书策略制定的节点才执行交易
* 排序服务节点 Ordering Service Nodes（OSN），orderers。确定交易的最终顺序。每个交易还包含着状态更新以及计算过程中的依赖，以及背书节点的签名。排序节点不知道应用的状态，也不参与交易的背书与验证。
> Fabric 网络支持多个区块链连接到同一个排序服务（排序服务是复用的），每一条区块链都是一个 channel，每个 channel 会有不同的节点与参与者。
### 3.2 Execution Phase
客户端签到并发送交易提案到一个或多个背书节点执行。提案包含了提交客户端的 ID。交易的payload 是由这些组成：交易操作、参数、chaincode 的标识符、随机数 nonce，交易标识符。
![](https://tva1.sinaimg.cn/large/006tNbRwgy1gb5e8vr00uj30ne0f9jta.jpg)
endorsers 通过在指定的 chaincode 上执行操作来模拟提案，chaincode 安装在区块链上，运行在 docker 环境中，与背书进程隔离开了。
提案仅依靠背书节点的本地区块链状态，不与其他节点同步。更进一步来说，背书节点不会将模拟背书结果保存到账本状态。由 chaincode 产生的状态仅在本地 chaincode 范围内有效，不能被其他 chaincode 所访问到。注意：chaincode 不应该维护程序代码中的本地状态，它只维护被GetState，PutState，DelState 操作所访问的对象。
经过模拟，背书节点生成`writeset`，包含模拟过程中发生的状态变化；还有 `readset`，包含了本次模拟的版本依赖。再将`writeset`与`readset`经过加密签名，发回客户端。
设计选择讨论：既然背书节点在模拟提案的时候没有和其他背书节点同步，那么就有可能两个节点在不同的账本环境下执行，从而产生不同的输出。对标准的背书策略而言，需要不同的背书节点产生相同的结果。这也意味着在高竞争的访问同一个 key 的情况下，客户端可能没法满足背书策略。与通过中间件同步复制数据库的主从备份相比，这是一个考虑点：区块链中没有一个节点的执行结果是可信的。
我们最终考虑采用这种设计，是因为它能显著地简化架构，并且适用于区块链应用。如Bitcoin的方法所示，分布式应用可以被表述为，在正常情况下，访问相同状态的竞争操作将会减少或完全消除（例如在比特币中，两个操作同时修改一个对象是不允许的，也就是双花攻击）。在未来，我们计划主键增强 Fabric 在竞争条件下的活性语义。
在排序阶段前先执行交易是容许非确定性 chaincodes 存在的关键。一个Fabric 中的非确定交易只能危害它操作的活性，例如，客户端可能收集不到足够数量的背书结果。这相较于 order-execute 架构是个重大优势。因为不确定性的操作将导致节点状态的不一致。
最后，容许不确定性的执行也解决了来自不受信任 chaincode 的 DoS 攻击，因为背书节点可以简单地中止执行，只要它根据本地策略怀疑遭受了 DoS 攻击。
### 3.3 Ordering Phase
当客户端收集到足够数量的某个提案的背书确认后，会组装交易信息并提交到排序服务。交易信息包括：交易负载（payload，例如包含参数的 chaincode 操作），交易元数据，一组背书信息。排序阶段确定各个 channel 提交过来的交易的最终顺序。换句话说，排序自动广播背书信息，从而建立共识，尽管有错误的排序服务节点。此外，排序服务将多个交易信息批量转换成区块，输出一个包含交易信息的哈希链表。
从更高层次来看，排序服务的接口仅支持两种操作：
* broadcast(tx): 客户端调用该操作来广播某个任意的交易 tx，通常包含交易负载（payload）与一个用于传播的客户端签名。
* B <- deliver(s)：客户端调用此方法召回区块 B，通过指定非负序列号 `s`。区块包含一系列的交易[tx~1~, tx~2~, tx~3~,...,tx~k~]，以及表示序号 s-1 区块哈希值的 `h`，例如，B = ([tx~1~, tx~2~, tx~3~,...,tx~k~]，h)。该方法具有幂等性，多次调用返回相同的区块 B。

排序服务保证在同一个 channel 中递送的区块都是完全排序的。更具体的说，排序服务保证每个 channel 的`safety properties`如下：
* Agreement：一致性、幂等性。在任何正确的节点上调用 deliver(s)，都能得到相同的区块 B
* Hash chain integrity：哈希链完整性。如果某个**正确**节点找回序号 s 的区块 B，另一个**正确**节点召回序号 s+1 的区块$B'=([tx_1,...,tx_k], h')$，那么$H(B)=h'$，其中 $H(·)$ 表示加密哈希函数。
* No skipping：如果一个正确的节点 p 收到区块 B，通过某个非负整数 s。那么对于区块[0, s-1]，p 也肯定已经收到了
* No creation：如果某个正确的节点收到区块 B，那么对于 $tx \in B$，必定有某个客户端广播了 tx。
  
排序服务支持如下`liveness properties`
* Validity：如果有某个客户端调用了 broadcast(tx)，那么最终所有的正确节点必然会收到一个包含 tx 的区块 B。

> 这里说明一下什么是 `safety properties` 与 `liveness properties`
> safety properties：保证不会有坏的情况发生，promise nothing bad happens，换句话说，保证什么样的事件不会发生
> liveness properties：保证好的事情最终会发生，promise something good eventually happens，换句话说，保证什么样的事件最终会发生

区块链中有许多的节点，但是只有其中一小部分实现了排序服务。Fabric 可以使用配置的方式使用内置的 gossip 服务来完成排序服务节点到所有节点间的传播。gossip 服务的实现是可扩展的，且对排序服务节点来说是透明的，因而不论排序服务使用 CFT 还是 BFT，都能保证 Fabric 的模块化。
### 3.4 Validation Phase
区块通过排序服务节点或 gossip 协议传播到各个节点。传播到 validation 阶段的区块由如下三步顺序构成：
1. 区块内的所有交易的背书策略评估（endorsement policy evaluation）都是并行发生的。该评估是所谓 VSCC（validation system chaincode）的任务，是区块链配置的一部分静态库，负责验证关于配置的 chaincode 背书策略的背书。如果背书验证未通过，交易将被标记为无效的，产生的影响将会被忽略。
2. 读写冲突检查是按先后顺序在区块中进行。对每一笔交易，先比较 readset 中的 keys 与节点本地保存的账本当前状态中的 keys 的版本是否一致。如果不一致，则交易被标记为无效，产生的影响将被忽略。
3. 账本状态更新操作最后进行，在这个阶段中区块会被追加到本地储存的账本中，区块链整体会被更新。特别地，当区块链被加到账本中时，前两步的操作将被持久化。并且，所有状态的更新是通过写 key-value 对到 本地的writeset中来实现的。
   
**设计选择讨论**：Fabric 的账本包含所有交易信息，包括那些被认为不合法的交易。这是因为排序服务是无状态的，以及验证阶段是在共识达成之后才进行的。这些特性在某些需要在后续审计过程中追踪不合法交易的用例中是很有用的，这与那些只记录合法交易的账本系统行程对比（例如：比特币与以太坊）。另外，由于 Fabric 是需要准入认证的，因而检测客户端的 DoS 攻击会很容易。例如可以使用黑名单，或者使用交易费的形式，使得 DoS 攻击的代价变得巨大。
### 3.5 Trust and Fault Model
Fabric 可以适应灵活的信任与故障假设。通常来说，每个客户端都被认为是恶意的或拜占庭错误的。节点被分为不同的组织，每个组织形成一个信任域，从而每个节点信任它所有组织下的其他所有节点，但不信任其他组织的节点。
Fabric 网络的完整性依赖于排序服务的一致性。排序服务的信任模型直接依赖于它的实现。
需要注意的是，Fabric 将应用的信任模型与共识机制的信任模型解耦开来。即，一个分布式应用可以定义自己的信任假说，通过背书策略来表达，与排序服务实现的共识机制独立。
## 四、Fabric 组件
Fabric 由 Go 语言编写，使用 gRPC 架构在客户端、节点、排序服务之间通信。接下来将讨论一些重要组件的具体细节。
![](https://tva1.sinaimg.cn/large/006tNbRwgy1gbfw5mgmh4j30vo0ko0uz.jpg)
### 4.1 Membership Service
Membership Service Provider（MSP）维护系统中所有节点（客户端，节点，OSNs排序服务节点）的身份，同时负责分发用于身份认证与授权的节点证书。由于 Fabric 是需要准入允许的，节点之前彼此的交流信息都是经过身份认证的，通常是经过数字签名的。MSP 在每个节点上都包含一个组件，负责交易身份的验证，交易完整性的校验，签署与验证背书，以及对其他区块链操作的验证。key 管理工具与节点的注册工具同样也是 MSP 的一部分。
MSP 是一个抽象概念，可以在此基础上有不同的实例化。MSP 在 Fabric 上的的默认实现是使用标准的基于数字签名验证的 PKI 方法，能够适应商业的认证机构（certification authorities，CAs）。Fabric 同样也提供独立部署的 CA，叫做 Fabric-CA。
Fabric 允许两种建立区块链网络的方式：
* offline mode：证书与 CA生成，通过带外（**out-of-band**）的方式分发到所有节点。peers 与 orderers 只能通过 offline 的方式注册。
* online mode：对于参与的客户端，Fabric-CA 提供该方式提供加密的证书。
### 4.2 Ordering Service
排序服务管理着许多 channel。对每一个 channel，都提供如下服务：
1. 原子广播（atomic broadcast），帮助确立交易顺序，实现 `broadcast` 和 `deliver` 方法。
2. 重新配置一个 channel，当它的成员通过广播更新配置交易修改了 channel。
3. 可选的，访问控制，在那些排序服务扮演可信任实体的配置中，限制交易的传播以及限制接受来自指定客户端或节点的区块。

交易服务由一个创世区块引导，该区块携带着配置交易（configuration Transaction），配置交易定义了配置服务的属性。
实际上，原子广播方法是由 Kafka 实例实现的，它提供了一个可扩展的发布-订阅模式以及基于作呕keeper 的在节点故障情况下的强一致性消息队列。Kafka 可以运行在于 OSNs 物理隔离的节点上。OSNs 作为 peers 与 Kafka 之间的代理。
一个 OSN 收到一个新的交易立即转发到 atomic broadcast（例如，Kafka broker）。OSNs 将 atomic broadcast 中的信息分块并组成区块。当满足以下三个条件之一时，一个区块立即被切割：
1. 区块包含指定的最大数量的交易
2. 区块达到最大字节数
3. 距离收到上一笔交易已经超过某个指定的时间段
这个批处理是确定性的，因此所有节点都会产生相同的区块。很容易看出，给定来自 atomic broadcast的交易流，前两个条件是非常确定的。为了确保在第三种情况下能够得到相同的区块，节点从收到来自 atomic broadcast 的第一条交易起就开始计时。如果当计时到期后区块还没有被切割，OSN 将会发送一条特殊的 time-to-cut 交易到 channel 中，里面包含待切区块的序列号。另一方面，在收到第一条 time-to-cut 交易请求后，OSN 会立即切割出一个新的区块。由于这个交易是自动的发送给所有的 OSNs，他们都将包含相同的交易列表。OSNs 会持久化最新收到的一些区块到本地文件系统中，一遍能够响应 peers 的 deliver 请求。
基于 Kafka 的排序服务是目前可用的三中实现方式之一。一种中心化的排序服务节点，称作 Solo，运行在一个节点上，目前用于开发环境。还提供另一种 基于 BFT-SMaRt 的 proof-of-concept排序服务。它保证了原子广播服务，但暂时没有重新配置与访问控制。
### 4.3 Peer Gossip 
execution，ordering，validation 分类的一个优势就是，他们可以独立地进行扩展。然而，由于大多数的共识算法（在 CFT 与 BFT 模型中）都是受限于带宽的，排序服务节点的吞吐量受限于节点的网络能力。共识算法不能通过增加节点来扩展，相反，吞吐量会降低。由于 ordering 与 validation 已经解耦，我们更感兴趣如何高效地将 execution 的结果在经过排序后，发往所有的 peers 进行验证。这与如新增 peers 与断线重连 peers 的状态转移（state transfer）问题一起，都是 gossip 组件需要解决的重要问题。Fabric Gossip 利用 epidemic multicast 以达成此目的。区块都经由排序服务签名，这意味着一个 peer 有能力通过接受所有的区块，独立地组装区块链并验证其完整性。
这个 gossip 的通信层是基于 gRPC 并使用具有相互认证的 TLS，这使得每一方都能够将 TLS 证书绑定到远程 peer 的身份。gossip 组件维护一份最新的系统中在线peer 的 membership view。所有节点都通过周期性传播的 membership 数据独立地构建一份本地membership view。而且，在peer 宕机后或网络中断后，能够重新连接到这个 membership view。
Fabric gossip 采用两个阶段进行消息传播：
1. 在 push 阶段，每一个节点从 membership view 中随机选择一个活跃的相邻节点集，并向它们转发消息；
2. 在 pull 阶段，每个 peer 随机选择一些 peers 并请求丢失的消息。
已经有研究表明，将这两种方法联合使用，对于优化利用可用带宽以及确保各个 peers 能够以高概率接受所有消息至关重要。为了减少从排序节点向网络发送区块的负载，协议还选举了一个 leader peer 来代表它们从排序服务拉取区块和开始 gossip 分发。
### 4.4 Ldger
每个 peer 上的账本（Ledger）组件 维护持久化存储的账本及其状态，并支持 **simulation**，**validation**，以及 **ledger-update** 阶段。通常来说，它由一个区块存储（block store）以及一个节点交易管理（peer Transaction manager）
账本区块存储系统（Ledger block store）持久化交易区块并以一个 append-only 文件系统的方式实现。由于区块是不可变的且以一个既定的顺序到达，一个 append-only 的结构能发挥最大的性能。作为补充，block store 维护一个随机访问区块以及区块内交易的索引。
节点交易管理（peer Transaction Manager，PTM）维护带版本号的 key-value存储的最新状态。它以（key，val，version）元组的形式为每一个 chaincode 的 entry key 存储数据，包含它最新的 value 值 val 以及最新的版本号 ver。版本号有区块序列号以及区块内的交易序列号组成。这使得版本号会全局唯一并单调递增。PTM 使用本地的 KV 存储来实现版本变化，例如 LevelDB 或 Apache CouchDB。
在 simulation 阶段，PTM 提供一个稳定的最新状态的快照给交易。正如 3.2 节提到的，PTM 会为每一条通过 GetState 访问的条目在 readset 中一个元组(key, ver)，以及为每一条由交易通过 PutState 访问的条目在 writeset 中更新元组(key, ver)。此外，PTM 支持范围查找，它计算查询结果（一组(key,value)元组）的哈希值，并将查询的字符串本身及其哈希结果加入到 `readset` 中。
对于交易验证，PTM 按顺序验证 block 中的每一笔交易。验证是否交易与之前的交易（在同一个区块中或先前的区块中）冲突。对于 readset 中的每一个 key，如果它的版本号与最新状态（假设先前所有的合法交易都已经提交）中的不同，则 PTM 会将该交易标记为非法。对于范围查询，PTM 会重新执行该查询，计算结果哈希值并与 readset 中的做比较，来确保没有**幽灵读**的发生。
账本组件能容许 peer 在账本更新期间宕机，如下所示：在收到一个新的区块之后，PTM 已经通过 3.2 节提到的位图对区块里的交易执行验证并标记交易为合法/不合法。然后账户组件把区块写入到 ledger block store 中，写入磁盘，随后更新 block store 的索引。然后 PTM 将 writeset 中的所有状态变化应用到本地版本存储。最终，计算并持久化得到一个值——savepoint，表示为目前最大的成功的区块序列号。savepoint 用与节点从宕机后恢复时，还原索引与最新的状态。
### 4.5 Chaincode Execution 
Chaincode是在一个松散耦合的环境中执行的，该环境支持用于添加新的Chaincode编程语言的插件。目前支持 Go，Java以及 Node。
每一个用户级或应用 chaincode 都运行在一个独立地 Docker 进行环境中。该环境能够隔离其他的 chaincode 以及其他的 peer。这能够接话 chaincode 的生命周期管理（例如启动，停止，终止）。 chaincode 与 peer 通过 gRPC 交流。通过这种松耦合方式， peer 无法得知 chaincode 实际上是用何种语言实现的。
与应用chaincode 不同，系统 chaincode 直接运行在peer 的进程中。系统 chaincode 能实现 Fabric 需要的指定功能，并可以在那些用户 chaincode 被过分限制的情况下使用。
### 4.6 Configuration and System Chaincodes
Fabric 可以通过 channel 配置文件以及特殊 chaincode——系统 chaincode 进行个性化配置。
回忆一下：Fabric 中的每一条 channel 组成了一个逻辑区块链。一个 channel 的配置是由一个保存在特殊的配置区块（configuration block）中的元数组来维护。每个配置区块都包含全部 channel 的配置且不包含任何其他交易。每一个区块链都开始于一个配置区块——创世区块（genesis block）——用来引导一个 channel。channel 配置包括：
1. MSP（Membership Service Provider） 关于参与节点的定义
2. OSNs（Ordering Service Nodes）的网络地址
3. 共识算法实现与排序服务的共享配置，包括 batch size，timeouts 等
4. 控制对排序服务操作的访问规则，例如 broadcast 与 deliver
5. 控制如何修改channel配置的每个部分的规则

Channel 的配置可以通过 channel 配置更新交易（channel configuration update Transaction） 来进行更新。此交易包含要对配置进行的更改的表示形式，以及一组签名。OSN 通过现有的配置来验证修改项是否是使用签名认证过的，以此来评估更新是否为合法的。然后排序服务节点生成一个新的配置区块，包含新的配置以及配置更新交易。peers 收到该区块后，基于现有配置验证配置更新是否经过授权，如果合法，更新现有的配置。
应用 chaincode 的部署参考背书系统 chaincode（endorsement system chaincode，ESCC） 以及验证系统 chaincode（validation system chaincode，VSCC）。ESCC 的输出可能作为 VSCC 的一部分输入进行验证。ESCC 以提案和提案模拟结果作为输入。如果结果是令人满意的，ESCC 会产生一个包含结果与背书的响应。对于默认的 ESCC，背书只是由 peer 本地登录 ID 的一个简单的签名。VSCC 以一个交易作为输入，并输出交易是否合法。对于默认的 VSCC，背书结果会被收集并根据chaincode 指定的配属策略进行评估。进一步的，系统 chaincode 实现其他支持功能，例如 chaincode 的生命周期。
## 五、评估
尽管 Fabric 还没经过性能调优与优化，我们在这节还是报告一些初步的性能数字。Fabric 是一个复杂的分布式系统，它的性能依赖于许多参数，包括：分布式应用的选择以及交易大大小，排序服务与共识算法的实现以及它们的参数，网络参数以及网络中的节点拓扑结构，节点运行的硬件，节点与 channel 的数量，更近一步，配置参数以及网络动态。因而 Fabric 的深度性能评估还有待后续安排进行。
缺少区块链的标准性能指标，我们使用最重要的区块链应用来评估 Fabric，一个简单的使用比特币的数据模型的 authority-minted 加密货币——Fabric coin（下文简称 Fabcoin）。这使得我们能够将 Fabric 的性能置于其他联盟链（Bitcoint，Ethereum）的环境中进行讨论。
在接下来的章节中，我们将讨论 Fabcoin，以及证明如何定制 validation 阶段与 endorsement policy。
### 5.1 Fabric Coin (Fabcoin)
UTXO 加密货币。由比特币引入的数据模型，通常被称为 Unspent transaction output 或者 UTXO，也被其他许多加密货币及分布式应用所使用。UTXO 描述了一个数据对象作为账本上独立的原子状态所演化的每一步。这个状态由一个交易创建，由另一个随后产生的唯一的交易所销毁（或消费）。**每个给定的交易都会销毁许多 input states 并创建一个或多个 output states。**比特币中的一个“币”是由一个 coinbase 交易创建的，并奖励给该区块的“矿工（miner）”。这在账本显示为一个指定该矿工为 owner 的coin state。**任意一个币都是可以被消费（spent），这意味着，coin 是通过交易指定给一个新的 owner，该交易销毁了指定前owner的当前 coin state，并创建了表示另一个代表新 owner 的 coin state。**
我们捕获在 ke-value store（KVS）中的 Fabric UTXO 如下。每一个 UTXO 状态相当于一个唯一的 KVS 条目，该条目创建一次（coin 状态为“unspent”），销毁一次（coin 状态为“使用”）。 同样，每个状态可以在 KVS 条目中如下表示：当它创建后，逻辑版本为 0，当它销毁后，逻辑版本为 1。任何的条目都不允许有并发更新（例如，尝试用不同的方式更新一个 coin 状态相当于“双花”）。
UTXO 中的值都是通过指向一些输入状态的交易传输的，这些交易属于发起这些交易的实体。一个实体拥有一个状态是因为实体的公钥也包含在它自己的状态中。每一个交易都会在 KVS 中生成一个或多个的输出状态来表示新的拥有者，删除 KVS 中的输入状态，确保输入状态中的值的总和等于输出状态中值的总和。也有一个策略决定了输入输出状态中的值是如何创建或销毁的。
_Fabcon 的实现_。每一个 Fabcoin 中的状态都是一组(key, val) = (txid.j, (amount, owner, label))，的形式，表示该 coin state 是一个标识符为 `txid` 的交易的第 `j` 个输出，分配 `amount` 数量个单位、使用 `label` 进行标签的的“货币”给以 `owner` 作为公钥的实体。其中 `label`是一个用来标志货币类型的字符串（例如‘USD’，‘EUR’，‘FBC’）。交易 ID 是 short 类型，能够唯一的标识所有的 Fabric 交易。Fabcoin 的实现由以下三个部分组成：
1. 客户端钱包
每个 Fabric 客户端维护一个 Fabcoin 钱包，该钱包在本地存储一组允许用来消费 coin 的加密密钥。例如创建一比交易一个或多个 coin 的 `SPEND` 交易，客户钱包会创建一个 Fabcoin 请求，request = (inputs, outputs, sigs)，其中：(1) 一组input coin states，inputs = [in, ...]，制定了客户想要消费的 coin 的状态；(2) 一组 output coin 状态，outputs = [(amount, owner, label), ...]。(3) 客户端钱包使用与input coin状态对应的私钥对Fabcoin请求和nonce(每个Fabric transaction的一部分)进行签名，并将签名添加到sigs set中。当输入 coin 状态中的 amounts 数量之和不少于输出 coin 状态中的 amount 之和，且输出 coin 状态中的 amount 之和为正的时，一笔 `SPEND` 交易才是合法的。对于一个创建新 coin 的 `MINT` 交易而言，inputs 只包含一个叫做 Central Bank(CB)的特殊实体的标识符（例如，一个公钥的引用），而 outpus 包含任意数量的 coin 状态。要被认为是合法的，MINT（铸币） 交易在 sigs 中的签名必须是使用 CB 的公钥对 Fabcoin 请求已经前文提到的 nonce 的签名。最后，客户端钱包将 Fabcoin 请求包含到一个交易中，并发送给它选定的 peer。
2.  Fabcoin chaincode
在一个节点上运行 Fabcoin 的 chaincode，模拟交易以及创建 readset 与 writeset。简而言之，以 `SPEND` 交易为例，对于每一个输入状态 $in \in inputs$，chaincode 先执行 $GetState(in)$；该操作将 $in$ 以及它当前的版本（4.4 节）一起纳入到 readsets 中。然后 chaincode 执行 $DelState(in)$ ，该操作会将每一个 $in$ 将入到 writesets 中，并高效的标记 coin 状态为“spent”。最后，对于 $j = 1,..., |outputs|$，chaincode 执行 $PutState(txid.j, out)$，其中，第 j 个输出 $out = (amount, owner, lable)$。除此之外，peer 还可以会运行Fabcon 的 VSCC 步骤中描述的交易验证代码。这是非必须的，因为 Fabcoin VSCC 实际上验证了交易，但是它允许（正确的）节点来过滤掉潜在的畸形交易。在我们的实现中，chaincode  在没有验证签名的情况下运行 Fabcoin VSCC
3.  实现了 Fabcoin 背书策略的自定义 VSCC（Validation System Chaincode）。
最后，每个节点使用定制化的 VSCC 来验证 Fabcoin 交易。使用对应的公钥校验了 sigs 中的签名，以及如下验证。对于一个铸币（MINT）交易，检查输出状态是否是基于对应的交易标识符（txid）创建的，并且输出 amount 的总和是否是大于零。对于 `SPEND` 交易，VSCC 额外验证：(1)对于所有的输入状态，都在  readset 中创建一个条目，并加入到 writeset 中，标记为已删除。(2) 输入 amount 之和等于输出 amount 之和。(3) input 与 output 状态的 label 想匹配。这里 VSCC 通过检索追溯输入 coin 在账本中的当前值来获取它们的 amount。
注意，Fabcoin VSCC 不对交易的双花攻击做检查，因为这是在客制化的 VSCC 之后运行的 Fabcoin 标准验证中进行。特别的，如果两个交易尝试分配同一个未消费的 coin 给一个新的 owner，两个交易都会通过 VSCC 逻辑的验证，但是会在后续由 PTM 执行的读写冲突检查中被发现。根据 3.4 节与 4.4 节，PTM 验证账本中的当前的版本号与 readset 中的是否一致；因此，经过第一笔交易后，coin 状态的版本号已被修改，后续的交易就会被认为是非法的。
 勘定感慨干景丹怪爱国 kjkgjakgjkag
 交定金怪安静干饭
 看分开精加工
 决定感慨就拐角
 精精加a工