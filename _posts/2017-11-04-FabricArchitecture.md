---
layout:     post
title:      "Hyperledger Fabric 0.6&1.0架构"
subtitle:   "区块链学习笔记"
date:       2017-11-04
author:     "Liu Ke"
header-img: "img/BC.jpg"
tags:
    - Blockchain区块链
    - Hyperledger Fabric
---

> 参考资料来源：巴比特

## Fabric 0.6总体架构

我们先看看0.6版本的总体架构：

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/171104/Fabric.png)

对应的0.6版本的运行时架构：

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/171104/0.6.png)

### 0.6版本的架构特点

1. 结构简单： 应用-成员管理-Peer的三角形关系，主要业务功能全部集中于Peer节点；
2. 架构问题：由于peer节点承担了太多的功能，所以带来扩展性、可维护性、安全性、业务隔离等方面的诸多问题，所以0.6版本在推出后，并没有大规模被行业使用，只是在一些零星的案例中进行业务验证；

## Fabric 1.0架构

Fabric 1.0版本运行时架构：

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/171104/1.0.png)

>备注：最新的1.0版本中，上图中的Membership服务已经改名为fabric-ca


### 1.0架构要点：

1. 分拆Peer的功能，将Blockchain的数据维护和共识服务进行分离，共识服务从Peer节点中完全分离出来，独立为Orderer节点提供共识服务；
2. 基于新的架构，实现多通道（channel）的结构，实现了更为灵活的业务适应性（业务隔离、安全性等方面）；
3. 支持更强的配置功能和策略管理功能，进一步增强系统的灵活性和适应性；

### 1.0架构目标

从Fabric的新架构设计的建议文档看，1.0版本的设计目标如下：

1. chaincode信任的灵活性：支持多个ordering服务节点，增强共识的容错能力和对抗orderer作恶的能力
2. 扩展性： 将endorsement和ordering进行分离，实现多通道（实际是分区）结构，增强系统的扩展性；同时也将chaincode执行、ledger、state维护等非常消耗系统性能的任务与共识任务分离，保证了关键任务（ordering）的可靠执行
3. 保密性：新架构对于chaincode在数据更新、状态维护等方面提供了新的保密性要求，提高系统的业务、安全方面的能力
4. 共识服务的模块化：支持可插拔的共识结构，支持多种共识服务的接入和服务实现

### 1.0架构特点

Hyperledger fabirc 1.0 版本的在0.6版本基础上，针对安全、保密、部署、维护、实际业务场景需求等方面进行了很多改进，特别是Peer节点的功能分离，给系统架构具备了支持多通道、可插拔的共识的能力，使得Fabric脱离了0.6版本带给人的青涩感（仅仅是个”验证与演示“版的，呵呵），已经接近于工业应用的需求；

## 1.0版本关键架构

### 多链与多通道

Fabric 1.0 的重要特征是支持多chain和多channel.


所谓的chain（链）实际上是包含Peer节点、账本、ordering通道的逻辑结构，它将参与者与数据（包含chaincode在）进行隔离，满足了不同业务场景下的”不同的人访问不同数据“的基本要求。

同时，一个peer节点也可以参与到多个chain中（通过接入多个channel）；如下图所示:

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/171104/multiChain.png)


关于通道：通道是有共识服务（ordering）提供的一种通讯机制，类似于消息系统中的发布-订阅（PUB/SUB)中的topic；基于这种发布-订阅关系，将peer和orderer连接在一起，形成一个个具有保密性的通讯链路（虚拟），实现了业务隔离的要求；通道也与账本（ledger）-状态（worldstate）紧密相关；如下图所示：

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/171104/multichannel.png)


共识服务与（P1、PN）、（P1、P2、P3）、（P2、P3）组成了三个相互独立的通道，加入到不同通道的Peer节点能够维护各个通道对应的账本和状态；也其实也对应现实世界中，不同业务场景下的参与方，例如银行、保险公司；物流企业、生产企业等实体结构；我们可以看到channel机制实际上是的Fabric建模实际业务流程的能力大大增强了，大家可以发挥想象力去找到可能的应用领域。

### 交易（数据）流程说明

在交易发送到网络之前，需要先向背书节点手机足够多的背书支持，同时采用专门的排序节点来负责整个网络中核心的排序环节。

目前，网络中存在以下4中不同种类的服务节点，彼此协作完成整个区块链系统的功能，实现了网络中节点的解耦，这是Fabirc的一大创新。

- **背书节点（Endorse）:**负责对交易的提案（proposal）进行检查和背书，计算交易执行结果。
- **确认节点（Committer）:**负责在接受交易结果前再次检查合法性，接受合法交易对账本的修改，并写入区块链结构。
- **排序节点（Order）:**对所有发往网络中的交易进行排序，将排序后的交易按照配置中的约定整理为区块，之后提交给确认节点进行处理。
- **证书节点（CA）：**负责对网络中所有的证书进行管理，提供标准的PKI服务。

并且，网络支持多通道特性。使用一条独立的系统通道（system channel）负责管理网络中的各种配置信息，并完成对其他应用通道（application channel,提供用户发送交易使用）的创建。

1.0版本总体流程如下图所示：

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/171104/invoke.png)

1. 应用程序通过SDK发送请求道Peer节点（一个或多个）
2. peer节点分别执行交易（通过chaincode），但是并不将执行结果提交到本地的账本中（可以认为是模拟执行，交易处于挂起状态），参与背书的peer将执行结果返回给应用程序（其中包括自身对背书结果的签名）
3. 应用程序 收集背书结果并将结果提交给Ordering服务节点
4. Ordering服务节点执行共识过程并生成block，通过消息通道发布给Peer节点，由peer节点各自验证交易并提交到本地的ledger中（包括state状态的变化）

上述过程对应的执行序列图如下：

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/171104/TransactionFlow.png)

在新的架构中，Peer节点负责维护区块链的账本（ledger）和状态（State），本地的账本称为PeerLedger，其结构如下：

![](https://raw.githubusercontent.com/dugu0808/dugu0808.github.io/master/img/171104/Ledger.png)

我们可以看到，整个区块结构分为文件系统存储的Block结构和数据库维护的State状态，其中state的存储结构是可以替换的，可选的实现包括各种KV数据库（LEVELDB，CouchDB等）；

## Fabric 1.0交易的完整生命周期

1. 客户端创建交易提案（chaincode函数和参数）并发送到Endorse Peer(背书节点)。
2. Endorse Peer节点执行chaincode，基于读取和写入的Key生成读写操作集。
3. Endorse Peer节点向客户端返回提案结果（包含读写操作集）。
4. 客户端把交易提交到 Order服务，交易内容包含来自提案结果的读写操作集。
5. Order服务将排完序的交易封装到区块中。
6. 区块将被发送给 Commit Peer节点。
7. Commit Peer节点执行如下这些操作：
   
	- 运行验证逻辑（VSCC背书第略，MVCC檢查读操作的版本自仿真交易以来未在数据库中被修改）
	- 在区块中指明哪些交易是有效的和无效的
	- 在内存或文件系统上把区块加入区块链，并目将区块内的有效交易写入状态数据库
	- 触发Event肖息，使得客户端通过SDK侦听知道哪些交是有效的或无效的

