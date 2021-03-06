
| 原文 | 作者 | 审核修正 |
| --- | --- | --- |
| [原文](http://hyperledger-fabric.readthedocs.io/en/latest/glossary.html) | Linsheng Yu | Baohua Yang |


Terminology is important, so that all Fabric users and developers agree on what we mean by each specific term. What is chaincode, for example. So we’ll point you there, whenever you want to reassure yourself. Of course, feel free to read the entire thing in one sitting if you like, it’s pretty enlightening!

专业术语很重要，所以所有”Fabric”项目用户和开发人员同意我们所说的每个特定术语的含义，举个例子：如什么是链码，因此我们将引导你到术语说明，让你随时可以消除对术语理解的疑虑，当然，如果你愿意的话可以自由的阅读整个文档，非常有启发！

## Anchor Peer - 锚节点

A peer node on a channel that all other peers can discover and communicate with. Each [Member](#Member) on a channel has an anchor peer (or multiple anchor peers to prevent single point of failure), allowing for peers belonging to different Members to discover all existing peers on a channel.

锚节点是通道中能被所有对等节点探测、并能与之进行通信的一种对等节点。通道中的每个成员都有一个（或多个，以防单点故障）锚节点，允许属于不同成员身份的节点来发现通道中存在的其它节点。

## Block - 区块

An ordered set of transactions that is cryptographically linked to the preceding block(s) on a channel.

在一个通道上，（区块是）一组有序交易的集合。区块往往通过密码学手段（Hash 值）连接到前导区块。

**Zhu Jiang：区块是一组有序的交易集合，在通道中经过加密（哈希加密）后与前序区块连接。**

## Chain - 链

The ledger’s chain is a transaction log structured as hash-linked blocks of transactions. Peers receive blocks of transactions from the ordering service, mark the block’s transactions as valid or invalid based on endorsement policies and concurrency violations, and append the block to the hash chain on the peer’s file system.

chain就是block之间以hash连接为结构的交易日志。peer从order service接收交易block，并根据背书策略和并发冲突标记block上的交易是否有效，然后将该block追加到peer文件系统中的hash chain上。

Zhu Jiang:账本的链是一个交易区块经过“哈希连接”结构化的交易日志。对等节点从排序服务收到交易区块，基于背书策略和并发冲突来标注区块的交易为有效或者无效状态，并且将区块追加到对等节点文件系统的哈希链中。

## Chaincode - 链码

Chaincode is software, running on a ledger, to encode assets and the transaction instructions (business logic) for modifying the assets.

链码是一个运行在账本上的软件，它可以对资产进行编码，其中的交易指令（或者叫业务逻辑）也可以用来修改资产。

## Channel - 通道

A channel is a private blockchain overlay on a Fabric network, allowing for data isolation and confidentiality. A channel-specific ledger is shared across the peers in the channel, and transacting parties must be properly authenticated to a channel in order to interact with it. Channels are defined by a [Configuration-Block](#Configuration-Block).

通道是构建在“Fabric”网络上的私有区块链，实现了数据的隔离和保密。通道特定的账本在通道中是与所有对等节点共享的，并且交易方必须通过该通道的正确验证才能与账本进行交互。通道是由一个“配置块”来定义的。

## Commitment - 提交

Each [Peer](#Peer) on a channel validates ordered blocks of transactions and then commits (writes-appends) the blocks to its replica of the channel [Ledger](#Ledger). Peers also mark each transaction in each block as valid or invalid.

一个通道中的每个对等节点都会验证交易的有序区块，然后将区块提交（写或追加）至该通道上账本的各个副本。对等节点也会标记每个区块中的每笔交易的状态是有效或者无效。

## Concurrency Control Version Check - 并发控制版本检查（CCVC）

Concurrency Control Version Check is a method of keeping state in sync across peers on a channel. Peers execute transactions in parallel, and before commitment to the ledger, peers check that the data read at execution time has not changed. If the data read for the transaction has changed between execution time and commitment time, then a Concurrency Control Version Check violation has occurred, and the transaction is marked as invalid on the ledger and values are not updated in the state database.

CCVC是保持通道中各对等节点间状态同步的一种方法。对等节点并行的执行交易，在交易提交至账本之前，对等节点会检查交易在执行期间读到的数据是否被修改。如果读取的数据在执行和提交之间被改变，就会引发CCVC冲突，该交易就会在账本中被标记为无效，而且值不会更新到状态数据库中。

## Configuration Block - 配置区块

Contains the configuration data defining members and policies for a system chain (ordering service) or channel. Any configuration modifications to a channel or overall network (e.g. a member leaving or joining) will result in a new configuration block being appended to the appropriate chain. This block will contain the contents of the genesis block, plus the delta.

包含为系统链（排序服务）或通道定义成员和策略的配置数据。对某个通道或整个网络的配置修改（比如，成员离开或加入）都将导致生成一个新的配置区块并追加到适当的链上。这个配置区块会包含创始区块的内容加上增量。

## Consensus - 共识

A broader term overarching the entire transactional flow, which serves to generate an agreement on the order and to confirm the correctness of the set of transactions constituting a block.

共识是贯穿整个交易流程的广义术语，其用于产生一个对于排序的同意书和确认构成区块的交易集的正确性。

## Current State - 当前状态

The current state of the ledger represents the latest values for all keys ever included in its chain transaction log. Peers commit the latest values to ledger current state for each valid transaction included in a processed block. Since current state represents all latest key values known to the channel, it is sometimes referred to as World State. Chaincode executes transaction proposals against current state data.

ledger的current state表示其chain交易log中所有key的最新值。peer会将处理过的block中的每个交易对应的修改value提交到ledger的current state，由于current state表示channel所知的所有最新的k-v，所以current state也被称为World State。Chaincode执行交易proposal就是针对的current state。

## Dynamic Membership - 动态成员

Fabric supports the addition-removal of members, peers, and ordering service nodes, without compromising the operationality of the overall network. Dynamic membership is critical when business relationships adjust and entities need to be added-removed for various reasons.

Fabric支持动态添加-移除members、peers和ordering服务节点，而不会影响整个网络的操作性。当业务关系调整或因各种原因需添加-移除实体时，Dynamic Membership至关重要。

## Endorsement - 背书

Refers to the process where specific peer nodes execute a transaction and return a `YES-NO` response to the client application that generated the transaction proposal. Chaincode applications have corresponding endorsement policies, in which the endorsing peers are specified.

Endorsement 是指一个peer执行一个交易并返回`YES-NO`给生成交易proposal的client app 的过程。chaincode具有相应的endorsement policies，其中指定了endorsing peer。

## Endorsement policy - 背书策略

Defines the peer nodes on a channel that must execute transactions attached to a specific chaincode application, and the required combination of responses (endorsements). A policy could require that a transaction be endorsed by a minimum number of endorsing peers, a minimum percentage of endorsing peers, or by all endorsing peers that are assigned to a specific chaincode application. Policies can be curated based on the application and the desired level of resilience against misbehavior (deliberate or not) by the endorsing peers. A distinct endorsement policy for install and instantiate transactions is also required.

Endorsement policy定义了依赖于特定chaincode执行交易的channel上的peer和响应结果（endorsements）的必要组合条件（即返回Yes或No的条件）。Endorsement policy可指定对于某一chaincode，可以对交易背书的最小背书节点数或者最小背书节点百分比。背书策略由背书节点基于应用程序和对抵御不良行为的期望水平来组织管理。在install和instantiate Chaincode（deploy tx）时需要指定背书策略。

## Fabric-ca

Fabric-ca is the default Certificate Authority component, which issues PKI-based certificates to network member organizations and their users. The CA issues one root certificate (rootCert) to each member, one enrollment certificate (eCert) to each authorized user, and a number of transaction certificates (tCerts) for each eCert.

Fabric-ca是默认的证书管理组件，它向网络成员及其用户颁发基于PKI的证书。CA为每个成员颁发一个根证书（rootCert），为每个授权用户颁发一个注册证书（eCert），为每个注册证书颁发大量交易证书（tCerts）。

## Genesis Block - 初始区块

The configuration block that initializes a blockchain network or channel, and also serves as the first block on a chain.

Genesis Block是初始化区块链网络或channel的配置区块，也是链上的第一个区块。

## Gossip Protocol - Gossip协议

The gossip data dissemination protocol performs three functions: 1) manages peer discovery and channel membership; 2) disseminates ledger data across all peers on the channel; 3) syncs ledger state across all peers on the channel. Refer to the [Gossip](http:--hyperledger-fabric.readthedocs.io-en-latest-gossip.html) topic for more details.

Gossip数据传输协议有三项功能：1）管理peer发现和channel成员；2）channel上的所有peer间广播账本数据；3）channel上的所有peer间同步账本数据。

## Initialize - 初始化

A method to initialize a chaincode application.

一个初始化chaincode程序的方法。

## Install - 安装

The process of placing a chaincode on a peer’s file system.

将chaincode放到peer的文件系统的过程。*（译注：即将ChaincodeDeploymentSpec信息存到chaincodeInstallPath-chaincodeName.chainVersion文件中）*

## Instantiate - 实例化

The process of starting a chaincode container.

启动chaincode容器的过程。*（译注：在lccc中将ChaincodeData保存到state中，然后deploy Chaincode并执行Init方法）*

## Invoke - 调用

Used to call chaincode functions. Invocations are captured as transaction proposals, which then pass through a modular flow of endorsement, ordering, validation, committal. The structure of invoke is a function and an array of arguments.

用于调用chaincode内的函数。Chaincode invoke就是一个交易proposal，然后执行模块化的流程（背书、共识、 验证、 提交）。invoke的结构就是一个函数和一个参数数组。

## Leading Peer - 主导节点

Each [Member](#Member) can own multiple peers on each channel that it subscribes to. One of these peers is serves as the leading peer for the channel, in order to communicate with the network ordering service on behalf of the member. The ordering service “delivers” blocks to the leading peer(s) on a channel, who then distribute them to other peers within the same member cluster.

每一个Member在其订阅的channel上可以拥有多个peer，其中一个peer会作为channel的leading peer代表该Member与ordering service通信。ordering service将block传递给leading peer，该peer再将此block分发给同一member下的其他peer。

## Ledger - 账本

A ledger is a channel’s chain and current state data which is maintained by each peer on the channel.

Ledger是个channel的chain和由channel中每个peer维护的world state。*（这个解释有点怪）*

## Member - 成员

A legally separate entity that owns a unique root certificate for the network. Network components such as peer nodes and application clients will be linked to a member.

拥有网络唯一根证书的合法独立实体。像peer节点和app client这样的网络组件会链接到一个Member。

## Membership Service Provider - MSP

The Membership Service Provider (MSP) refers to an abstract component of the system that provides credentials to clients, and peers for them to participate in a Hyperledger Fabric network. Clients use these credentials to authenticate their transactions, and peers use these credentials to authenticate transaction processing results (endorsements). While strongly connected to the transaction processing components of the systems, this interface aims to have membership services components defined, in such a way that alternate implementations of this can be smoothly plugged in without modifying the core of transaction processing components of the system.

MSP是指为client和peer提供证书的系统抽象组件。Client用证书来认证他们的交易；peer用证书认证其交易背书。该接口与系统的交易处理组件密切相关，旨在使已定义的成员身份服务组件以这种方式顺利插入而不会修改系统的交易处理组件的核心。

## Membership Services - 成员服务

Membership Services authenticates, authorizes, and manages identities on a permissioned blockchain network. The membership services code that runs in peers and orderers both authenticates and authorizes blockchain operations. It is a PKI-based implementation of the Membership Services Provider (MSP) abstraction.

成员服务在许可的区块链网络上认证、授权和管理身份。在peer和order中运行的成员服务的代码都会认证和授权区块链操作。它是基于PKI的MSP实现。

The `fabric-ca` component is an implementation of membership services to manage identities. In particular, it handles the issuance and revocation of enrollment certificates and transaction certificates.

`fabric-ca`组件实现了成员服务，来管理身份。特别的，它处理ECert和TCert的颁发和撤销。

An enrollment certificate is a long-term identity credential; a transaction certificate is a short-term identity credential which is both anonymous and un-linkable.

ECert是长期的身份凭证；TCert是短期的身份凭证，是匿名和不可链接的。

## Ordering Service - 排序服务或共识服务

A defined collective of nodes that orders transactions into a block. The ordering service exists independent of the peer processes and orders transactions on a first-come-first-serve basis for all channel’s on the network. The ordering service is designed to support pluggable implementations beyond the out-of-the-box SOLO and Kafka varieties. The ordering service is a common binding for the overall network; it contains the cryptographic identity material tied to each [Member](#Member).

将交易排序放入block的节点的集合。ordering service独立于peer流程之外，并以先到先得的方式为网络上所有的channel作交易排序。ordering service支持可插拔实现，目前默认实现了SOLO和Kafka。ordering service是整个网络的公用binding，包含与每个Member相关的加密材料。

## Peer - 节点

A network entity that maintains a ledger and runs chaincode containers in order to perform read-write operations to the ledger. Peers are owned and maintained by members.

一个网络实体，维护ledger并运行Chaincode容器来对ledger执行read-write操作。peer由Member拥有和维护。

## Policy - 策略

There are policies for endorsement, validation, block committal, chaincode management and network-channel management.

有背书策略，校验策略，区块提交策略，Chaincode管理策略和网络-通道管理策略。

## Proposal - 提案

A request for endorsement that is aimed at specific peers on a channel. Each proposal is either an instantiate or an invoke (read-write) request.

一种针对channel中某peer的背书请求。每个proposal要么是Chaincode instantiate要么是Chaincode invoke。

## Query - 查询

A query requests the value of a key(s) against the current state.

对于current state中某个key的value的查询请求。

## Software Development Kit - SDK

The Hyperledger Fabric client SDK provides a structured environment of libraries for developers to write and test chaincode applications. The SDK is fully configurable and extensible through a standard interface. Components, including cryptographic algorithms for signatures, logging frameworks and state stores, are easily swapped in and out of the SDK. The SDK API uses protocol buffers over gRPC for transaction processing, membership services, node traversal and event handling applications to communicate across the fabric. The SDK comes in multiple flavors - Node.js, Java. and Python.

SDK为开发人员提供了一个结构化的库环境，用于编写和测试链码应用程序。SDK完全可以通过标准接口实现配置和扩展，像签名的加密算法、日志框架和state存储这样的组件都可以轻松地实现替换。SDK API使用gRPC进行交易处理，成员服务、节点遍历以及事件处理都是据此与fabric通信。目前SDK支持Node.js、Java和Python。

## State Database - stateDB

Current state data is stored in a state database for efficient reads and queries from chaincode. These databases include levelDB and couchDB.

为了从Chaincode中高效的读写，Current state 数据存储在stateDB中，包括levelDB和couchDB。

## System Chain - 系统链

Contains a configuration block defining the network at a system level. The system chain lives within the ordering service, and similar to a channel, has an initial configuration containing information such as: MSP information, policies, and configuration details. Any change to the overall network (e.g. a new org joining or a new ordering node being added) will result in a new configuration block being added to the system chain.

包含在系统级定义网络的配置区块。系统链存在于ordering service中，与channel类似，具有包含以下信息的初始配置：MSP信息、策略和信息配置。对整个网络的任何变化（例如新的Org加入或者添加新的Ordering节点）将导致新的配置区块被添加到系统链。

The system chain can be thought of as the common binding for a channel or group of channels. For instance, a collection of financial institutions may form a consortium (represented through the system chain), and then proceed to create channels relative to their aligned and varying business agendas.

系统链可看做是一个channel或一组channel的公用binding。例如，金融机构的集合可以形成一个财团（以system chain表示），然后根据其相同或不同的业务创建channel。

## Transaction - 交易

An invoke or instantiate operation. Invokes are requests to read-write data from the ledger. Instantiate is a request to start a chaincode container on a peer.

Chaincode的invoke或instantiate操作。Invoke是从ledger中请求read-write set；Instantiate是请求在peer上启动Chaincode容器。


# 另一个翻译
## 术语
[原文](http://hyperledger-fabric.readthedocs.io/en/latest/glossary.html)  

#### 锚点Peer
它是通道中的一个peer节点，其他peer可以发现它并与它通信。通道中的每个[成员](#成员)都一个锚点peer(或者多个锚点防范单点失效)，用于通道中属于不同成员的peer可以发现全部peer。  

#### 区块
一组有序的事务，通过哈希链接到通道中的前一个区块。

#### 链
账本的链式一个事务日志，组织成多个哈希值链接的区块。peer从排序服务接收事务区块，根据背书策略和并发违规将区块中的事务标记为生效和失效，而且在peer的文件系统中将区块追加到哈希链。  

#### 链码
链码是运行在一个账本上的软件，编码了资产和事务指令（业务逻辑）来修改资产。  

#### 通道
通道是一个私有区块链，它允许数据隔离和保密。通道的账本被通道中所有peer共享，与通道交互的事务参与方必须被通道正确地认证。通道被[配置区块](#配置区块)定义。  

#### 提交(Commitemnt)
通道中的每个[Peer](#Peer)生效事务的有序区块，然后提交(写/追加)区块到它的通道[账本](#账本)副本。Peer会标记每个区块的每个事务为生效或失效。  

#### 并发控制版本检查
这是一个保持通道内多个peer之间状态同步的方法。peer并行执行事务，在提交到账本前，peer用读检测数据在执行的时候没有被改变。如果事务的读数据在执行时和提交时之间被改变了，则发生了并发控制版本检查违规，事务在账本中被标记为失效，不会改变状态数据库的数据。  

#### 配置区块
为系统链(排序服务)或通道容纳成员定义和策略配置数据。任何对一个通道或全部网络(如成员离开或加入)的任何配置修改都导致一个新的配置区块追加到相应链中。配置区块包含创世区块，再加上delta。

#### 共识
贯穿整个事务流程的一个更广泛的术语，用来按顺序生成一致性并确认按一组事务构组成区块的正确性。  

#### 当前状态
账本的当前状态表示包含在它的链式事务日志中的所有密钥的最新值。针对包含在已处理区块中的每个有效事务，peer将最新值提交到账本的当前状态。由于当前状态代表通道已知的所有最新键值，因此有时称为世界状态。链码针对当前状态数据执行事务提议。  

#### 动态成员资格
Hyperledger Fabric在不影响这个网络的可靠性的前提下，支持增减会员、peer和排序服务节点。当业务关系调整和引各种原因需要增减实体时，动态成员资格是很需要的。   

#### 背书
指一个过程：特定peer节点执行一个链码事务，返回提议响应到客户端应用。提议响应包含链码执行相应信息、结果(read set和write set)、事件和一个签名（证明peer执行了链码）。链码应用程序遵照背书策略，其中规定了背书peer。  

#### 背书策略
定义了通道中的哪些peer节点必须执行附加在链码应用中的事务，和需要的响应(背书)组合。需要这样一个策略，定义一个事务至少得到几个peer的同意，或最少多少比例的peer同意，或全部peer的同意，这个策略会附加到一个特定链码应用上。策略可以根据应用程序和期望水平来反对不当行为（有意或无意）。一个事务在被提交peer标记为生效前，必须符合背书策略。对部署、实例化事务也需要一个明确的背书政策。  

#### Hyperledger Fabric CA
Hyperledger Fabric CA是向网络成员组织和它们的用户发放PKI证书的默认CA组件。CA向每个成员发放一个根证书(rootCert)，象每个授权用户发放一个登记证书(ECert)。  

#### 创世区块
初始化区块链网络或通道的配置区块，也作为链上的第一个区块。  

#### Gossip协议
gossip数据广播协议完成三个功能：1）管理peer发现和通道成员资格；2）在通道中所有peer之间广播账本数据；3）在通道中所有peer之间同步账本状态。更多细节参考[Gossip](https://hyperledger-fabric.readthedocs.io/en/latest/gossip.html)主题。  

#### 初始化
一个初始化链码应用的方法。  

#### 安装
将链码放置到peer的文件系统的过程。

#### 调用(invoke)
用于调用链码函数。一个客户端应用通过发送事务提议到一个peer来调用链码。peer会执行链码和返回一个背书提议响应到客户端应用。客户端应用收集足够的提议响应以便满足背书策略，然后递交事务结果以便排序、生效和提交。客户端应用程序也可以选择不去递交事务结果。例如查询账本，客户端应用程序通常不会递交这个只读事务，除非是为了审计的目的想把读账本的行为记录日志。调用包括通道id、调用的链码函数和参数数组。  

#### 领导peer
每个[成员](#成员)在它订阅的每个通道中都可以有多个peer。其中一个peer可以作为该通道的领导peer，代表成员与网络排序服务进行通信。排序服务“递送”区块到一个通道的领导peer(可能多个)，它(或它们)再分发区块到成员集群的其他peer。  

#### 账本
账本是一个通道链和被通道中所有peer共同维护的当前状态数据。  

#### 成员
一个在网络中拥有特定根证书的独立法律实体。象peer和应用客户端这样的网络组件会被关联到成员。  

#### 成员服务提供商
成员服务提供商（MSP）是指为客户端提供（匿名）凭证的系统的抽象组件，以及参与Hyperledger/fabric网络的peer。客户端使用这些凭证对其事务进行身份验证，peer使用这些凭据来验证事务处理结果（背书）。在与系统的事务处理组件紧密连接的同时，这个接口的目标是定义成员服务组件，这样可以顺利地插入替代的实现，而无需修改系统的事务处理组件的核心。  

#### 成员服务
成员服务在授权区块链网络上认证、授权和管理身份。在peer和orderer节点上运行的成员服务代码认证和授权区块链操作。它是个成员服务提供商抽象的一个基于PKI的实现。

#### 排序服务
一个将事务排序进入区块的节点集合。排序独立于peer过程而存在，并以先来先服务的原则为网络上的所有通道进行事务排序。
一个集中或非几种的服务，对区块中事务进行排序。您可以选择“排序”功能的不同实现方式 - 例如：简化和测试的“solo”，用于碰撞容错的Kafka，或用于拜占庭容错的sBFT/PBFT。您也可以开发自己的协议来插入服务。排序服务在开箱即用的SOLO和Kafa实现的之外支持插件式实现。排序服务一般绑定到整个网络；它存放了每个[成员](#成员)的密钥身份材料。    

#### Peer
peer是一个网络实体，负责维护账本和为了读写账本而运行链码容器。peer被成员拥有和维护。  

#### 策略
是背书、生效、链码管理和网络/通道管理的策略。  

#### 提议
针对通道中的某个peer发出的背书请求。提议要么是个实例化请求，要么是个调用（读或写）请求。

#### 查询
查询是一个链码调用，它读账本当前状态但不会写账本。链码函数可以对账本按key查询，或按一组key查询。由于查询不会修改账本状态，客户端应用程序通常不会为了排序、验证和提交而递交只读事务。偶尔，客户端应用选择为了排序、验证和提交而递交只读事务，例如如果客户端需要账本在某个时间点的当前状态的审计证明。  

#### SDK
Hyperledger Fabric客户端SDK是为了方便开发者编写和测试链码应用而提供的一个结构化库环境。通过标准接口，SDK是完全可配置和可扩展的。包括签名算法、日志框架和状态存储，组件可以轻松进出SDK。SDK为事务处理、成员服务、节点遍历和事件处理提供了API。提供了多种风格的SDK：Node.js、Java和Python。

#### 状态数据库
当前状态数据被保存一个状态数据库中，以便链码可以方便地读和查询。支持的数据库包括levelDB和couchDB。

#### 系统链
包含一个在系统级别定义网络的配置区块。系统链存在于排序服务中，与通道类似，具有包含以下信息的初始配置：参与者组织的根证书和排序服务节点、策略、OSN(排序服务节点)监听地址以及配置详细信息。对整个网络的任何改变（例如一个新的组织加入或新增一个排序节点）将导致新的配置区块被添加到系统链中。  
系统链可以被认为是通道或通道组的通用绑定。例如，一系列金融机构可以组成一个联盟（通过系统链表示），然后再根据联盟或业务议程创建通道。  

#### 事务
为了排序、验证和提交而被递交的调用或实例化结果。调用是一个为了从账本读或写数据的请求。实例化是一个为了在通道中启动和实例化链码的请求。应用客户端收集从背书peer返回的调用或实例化响应，包装结果集和背书为一个事务，事务为了排序、验证和提交而被递交。

#### 排序者(Orderer)
构成排序服务的网络实体之一。一个排序服务节点（OSN）的集合，用来将事务排序进区块。在“独奏”的情况下，只需要一个OSN。事务被“广播”给排序者，然后作为区块“交付”到适当的通道。  

#### 背书者(Endorser)
一个特定的peer角色，背书peer负责模拟事务，并且防止不稳定或不确定的事务通过网络。事务以事务提议的形式发送给背书者。所有的背书peer同时也是committer peer（即他们写账本）。  

#### 提交者(Committer)  
一个特定的peer角色，负责将有效事务写入某通道的账本。一个peer可以同时充当背书者和提交者，当很多时候只是充当提交者。  
