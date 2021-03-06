## 架构解释
Hyperledger Fabric架构具有以下优点：  
- **Chaincode信任灵活性**。该架构将链码（区块链应用）信任假设与排序信任假定分开。换句话说，排序服务可以由一组节点（orderer）提供，并容忍的一些节点的失效或欺诈；并且对于每个链码，背书者可能不同。  
- **可扩展性**。由于背书节点只负责特定链码，与排序节点无关，系统可以比通过相同节点完成这些功能更好地扩展。具体而言，当不同的链码指定无关的背书节点时，这引入了背书节点之间的分区机制从而允许链码的并行运行（背书）。  
- **保密**。该架构便于部署对于其交易的内容和状态更新具有机密性要求的链码。
- **共识模块化**。该体系结构是模块化的，并允许插件式的共识（即排序服务）实现。  
### 1.系统架构
区块链是由多个彼此通信的节点组成的分布式系统。区块链运行程序称为链码，保存状态和总帐数据，并执行交易。链码是链码调用交易操作的核心元素。交易必须被“背书”，只有背书的交易可能会被提交，并对状态产生影响。管理功能和参数存在一个或多个特殊的链码，统称为系统链码。  
#### 1.1 交易
交易可能有两种类型：
- **部署交易**创建新的链码并将一个程序作为参数。当部署交易成功执行时，链码就被安装在区块链上。  
- **调用(Invoke)交易**在先前部署的链码的上下文中执行操作。调用交易是指链码及其提供的某个函数。链码会执行指定的函数 - 这可能涉及修改相应的状态，并返回一个输出。  
  如后面所述，部署交易是调用交易的特殊情况，其中创建新链的部署交易相当于系统链码上的调用交易。  
  **备注**： 这个文件目前假设一个交易要么创建新的链码，要么调用已经部署的链码提供的操作。本文档尚未描述：a）对查询（只读）交易（包含在v1中）的优化，b）对交叉链码交易（post-v1特性）的支持。  
#### 1.2 区块链数据结构
**1.2.1状态**  
区块链的最新状态（或简称为状态）被建模为版本化的键值库（KVS），其中键名称和值是任意的blob。这些条目由运行在区块链上的链码（应用程序）通过`put`和`get` KVS来操纵。状态被持久化存储，状态的更新被日志记录。请注意，采用版本化的KVS作为状态模型，实现可以使用实际的KVS，还可以使用RDBMS或任何其他解决方案。  
**1.2.2账本**  
账本提供了一个在系统运行过程中发生的所有成功状态变化（我们称为有效交易）和不成功的改变状态尝试（我们称为无效交易）的可验证历史。  
账本是由排序服务（参见1.3.3节）构建，是一个由交易区块（有效或无效）组成的有序哈希链。哈希链定义了账本中的总的区块顺序，每个区块都包含一个完全有序的交易数组。这会在所有交易中强加顺序。  
账本保存在所有peer节点，并可保存在部分排序节点(可选)。在排序节点上下文中，我们称账本为`OrdererLedger`，然而在peer上下文中，我们称账本为`PeerLedger`。`PeerLedger`与`OrdererLedger`的区别在于，peer在本地维护一个位掩码(bitmask)，将有效的交易从无效的交易中分离出来（更多细节见第XX章）。  
像第XX节（v1之后的功能）中所述的那样，peer可能会删除`PeerLedger`。排序节点维护`OrdererLedger`是为了容错性和`PeerLedger`的可用性，并且可以决定在什么时候修剪(prune)它，前提是排序服务的属性（参见第1.3.3节）得到维护。  

账本允许peer重放所有交易的历史记录并重建状态。因此，如1.2.1节所述的状态是可选的数据结构。  
#### 1.3节点
节点是区块链的通信实体。一个“节点”只是一个逻辑功能，也就是不同类型的多个节点可以在同一个物理服务器上运行。重要的是节点如何分组在“信任域”中，并与控制它们的逻辑实体相关联。  
存在三种类型的节点：  
1. **客户端**或**提交客户端**：它发出实际调用交易到背书节点，广播提议交易到排序服务。  
2. **Peer**：它提交交易、维护状态和一个账本副本(参看1.2节)。此外，peer还可以具有背书者角色。  
3. **排序服务节点**或**orderer**：运行通信服务的节点，实现交付担保，如原子或全部排序广播。  

**1.3.1客户端** 
客户代表代表最终用户行事的实体。它必须连接到peer与区块链进行通信。客户端可以连接到其选择的任何peer端。客户端创建并调用交易。  

如第2节详细描述的那样，客户端同时与peer和排序服务通信。  

**1.3.2 Peer**  
peer以区块的形式从排序服务接收顺序的状态更新，并维护状态和账本。  

peer还可以担当**背书peer**的特殊角色(或称**endorser**)。背书peer的特殊功能是针对特定的链码进行的，并且包含在提交前的背书交易中。每个链码可以指定一个背书策略，策略可以指向一组背书peer。该策略为有效的交易背书定义了必要和充分的条件（通常是一组背书签名），如后面的第2和第3节所述。有一种部署新链码的特殊部署交易，它的（部署）背书策略是指定为系统链码的背书政策。  

**1.3.3排序服务节点(Orderer)**  
排序服务(orderer)是一个通信架构，它提供投递担保。排序服务可以用不同的方式实现：从中心式服务（例如在开发和测试中使用）到针对不同网络和节点失效模型的分布式协议。

排序服务为客户和peer提供共享的*通信通道*，为包含交易的消息提供广播服务。客户端连接到该通道，并可以在该通道上广播消息，然后传送给所有peer。该通道支持所有消息的原子交付，也就是全排序交付和（特定实现）可靠性的消息通信。换句话说，通道向所有连接的peer输出相同的消息，并以相同的逻辑顺序输出到所有peer。这种原子通信保证也被称为*全序广播*、*原子广播*或分布式系统下的*共识*。通信消息是包含在区块链状态中的候选交易。  

**分区(排序服务通道)**。排序服务可以支持多通道，类似发布/订阅（pub / sub）消息系统的主题。客户端可以连接到给定的通道，然后可以发送消息并获取到达的消息。通道可以被认为是分区 - 连接到一个通道的客户端不知道其他通道的存在，但客户端可能连接到多个通道。尽管Hyperledger Fabric中包含的一些排序服务实现支持多个通道，但为了简化表示，在本文档的其余部分，我们假设排序服务包含单个通道/主题。  

**排序服务API**。peer通过排序服务提供的接口连接到排序服务提供的通道。排序服务API包含两个基本操作（很常见的*异步事件*）：  
- `broadcast(blob)`: 客户端调用这个广播二进制消息`blob`到通道。这在BFT上下文中也叫做`request(blob)`，当发送请求到一个服务时。  
- `deliver(seqno, prevhash, blob)`：(略)

**账本和区块信息**。账本(参看1.2.2节)包含排序服务输出的所有数据。简而言之，它是一系列`deliver(seqno, prevhash, blob)`事件，根据前面所述的`prevhash`计算形成一个哈希链。  
大多数情况下，出于效率原因，不输出单个交易（blob），排序服务将对blob进行分组(批处理)，并在一个`deliver`事件输出区块。在这种情况下，排序服务必须强制和传达每个块内的blob的确定性排序。区块中的blob数量可以由排序服务实现动态地选择。  

下面为便于表述，我们定义排序服务属性（本小节的其余部分），并解释交易背书的工作流程。假定一个blob产生一个`deliver`事件。一个区块对应一组顺序的`deliver`事件（每个blob一个事件）。区块本身也对应一个`deliver`事件，依靠排序服务，多个区块顺序排列组成区块链。  

### 2.交易背书的基本流程
#### 2.1 客户端创建一个交易并将它发送到选择的背书peers  
为了调用一个交易，客户端发送一个`PROPOSE`消息给它所选择的一组背书peer（可能不是同一时间 - 见2.1.2节和2.3节）。对于给定的`chaincodeID`客户端根据背书策略(看第3节)可以获得一组背书peer。例如，根据给定`chiancodeID`交易可以发送给所有背书peer。也就是说，一些背书人可能会离线，其他人可能会反对并选择不赞成交易。发起客户端利用目前可用的背书节点尝试满足策略表达式的要求。  

接下来，我们首先详细描述PROPOSE消息格式，然后讨论提交客户端和背书人之间可能的交互模式。  
**2.1.1 PROPOSE消息格式**  
**2.1.2 消息模式**  

#### 2.2 背书peer模拟一个交易并生成一个背书签名  
从客户端收到`<PROPOSE,tx,[anchor]>`消息后，背书peer`epID`首先验证客户端签名`clientSig`，然后模拟一个交易。如果客户端指定了`anchor`，则背书peer模拟交易的方法是，在本地KVS中读取与版本号`anchor`相匹配的keys。  
模拟一个交易包括背书peer临时性执行一个交易(`txPayload`)（调用交易中`chaincodeID`指定的链码）和背书peer本地保存的状态副本。（用状态副本和临时交易可以得到一个临时的新状态）  
作为执行的结果，背书peer计算*读版本依赖*(`readset`)和*状态更新*(`writeset`)，在数据库语言中也叫*MVCC+postimage info*。  
回想一下状态由键值对组成。所有的键值对条目是版本化的，也就是每个条目都包含有序的版本信息，当通过一个key更新库中的值时版本号会增长。peer解释(模拟执行)交易，记录链码访问的键值对，但不会真的更新状态。进一步来说：
- 在背书peer执行交易前，给定状态`s`，对于交易读取的每个key`k`，键值对`(k,s(k).version)`被添加到`readset`。  
- 此外，对于每个key`k`交易更新为新值`v'`，键值对`(k,v')`被添加到`writeset`。或者，`v'`可能是以前值(`s(k).value`)的新值的增量。  

如果客户端在`PROPOSE`消息中指定了`anchor`，则客户端指定的`anchor`必须等于背书peer模拟交易时产生的`readset`。  
然后，peer将内部`tran-proposal`（即`tx`）转发作为其背书交易逻辑的组成部分，称为**背书逻辑**。默认情况下，peer中的背书逻辑接受`tran-proposal`并简单地签名`tran-proposal`。然而，背书逻辑可能解释任意函数，例如，用`tran-proposal`和`tx`作为输入与遗留系统交互来判断是否批准交易。  
如果背书逻辑决定对一个交易进行背书，它发送`<TRANSACTION-ENDORSED, tid, tran-proposal,epSig>`消息到发起客户端(`tx.clientID`)，这里：
- `tran-proposal := (epID,tid,chaincodeID,txContentBlob,readset,writeset)`，`txContentBlob`是链码/交易指定信息。  
- `epSig`是背书peer在`tran-proposal`上的签名  

另外，如果背书逻辑拒绝对交易签名，背书界面会发送一个`(TRANSACTION-INVALID, tid, REJECTED)`消息到发起客户端。  
注意，背书节点在这一步不会修改状态，因交易模拟而生成的更新不会影响状态。  

#### 2.3 发起客户端收集交易背书并广播到排序服务
发起客户端等待，直到它收到“足够的”消息和签名(`TRANSACTION-ENDORSED, tid, *, *`)，得出交易提议被背书的结论。正如2.1.2节所讨论的，这可能涉及一个或多个与背书人交互的往返。  

“足够的”的确切数量取决于链码背书策略（另见第3节）。如果背书策略得到满足，交易就获得背书; 注意它还没有提交。来自背书peer的签名`TRANSACTION-ENDORSED`消息集合将建立一个背书的交易，称为*背书*(`endorsement`)。

如果发起客户端没有收集到交易的背书，则放弃该交易，并选择稍后重试。  

对于含有有效背书的交易，我们现在开始使用排序服务。发起客户端使用`broadcast(blob)`调用排序服务，这里`blob=endorsement`。如果客户端没有直接调用排序服务的能力，可以通过自己选择的peer代理它的广播。这样的peer必须被客户信任：不会从`endorsement`删除任何消息，除非交易被认为是无效的。但是请注意，代理peer可能不会编造有效的endorsement。  

#### 2.4 排序服务
当一个事件`deliver(seqno, prevhash, blob)`发生，并且peer已经应用了blob的序列号低于`seqno`的所有的状态更新，peer做下面这些：
- 它根据`blob.tran-proposal.chaincodeID`指向的链码策略检查`blob.endorsement`看是否有效。  
- 在一个典型的情况下，它也验证依赖关系（blob.endorsement.tran-proposal.readset）同时没有被违反。在更复杂的用例中，背书的`tran-proposal`字段可能有所不同，在这种情况下，背书策略（第3部分）指定了状态如何演变。  
  （省略一些）

### 3. 背书策略
#### 3.1 背书策略规范
一个**策略**，是对交易进行背书的条件。区块链peer拥有一套预先设定的背书策略，由安装特定链码的`deploy`交易引入。背书策略可以参数化，这些参数可以由`deploy`交易指定。  

为了保证区块链和安全属性，这套认可策略应该是一组经过验证的策略，确保有限的执行时间（终止），确定性，性能和安全保证。  

不允许动态增加背书策略，日后可以予以支持。  

#### 3.2 对背书策略的交易评估
只有通过策略背书，交易才被宣布有效。链码的调用交易首先必须获得链码策略的背书，否则将不会被提交。这是通过发起客户端和peer之间的交互来进行的，如第2节所述。  

形式上，背书策略是对背书的一个判断，并可能进一步陈述评估为“真”或“假”。对于部署交易，根据系统范围的策略（例如从系统链码）获得背书。

(省略一些)

### 通道

Hyperledger Fabric**通道**是两个或多个特定网络成员之间通信的私有“子网”，用于进行私下交易和保密交易。通道是由成员（组织）、每个成员的锚点、共享账本、链码应用程序和排序服务节点定义的。网络上的每个交易都在一个通道上执行，每个通信方都必须经过认证和授权才能在该通道上进行交易。每个加入通道的节点都有自己的身份，身份由成员服务提供者（MSP）提供，它将每个节点认证给通道节点和服务。

为了创建一个新的通道，客户端SDK调用配置系统链码并引用属性，例如**锚点 peer**和**成员（组织）**。该请求为通道账本创建**初始区块**，存储有关通道策略、成员和锚点的配置信息。在将新成员添加到现有通道时，这个初始区块（还可能有最近的重新配置区块）将与新成员共享。

注意：

*有关配置交易的属性和原始结构的更多详细信息，请参阅[通道配置（configtx）](http://hyperledger-fabric.readthedocs.io/en/latest/configtx.html)部分。*

对通道上每个成员的**领导peer**的选择，决定哪个peer代表成员与排序服务通信。如果没有确定领导者，则可以使用算法来确定领导者。共识服务在一个区块内对交易进行排序，并将区块传递给每个领导peer，该peer再将区块分发自己的成员peer，通过**gossip**协议广播到整个通道。

尽管任何一个锚点peer可以属于多个通道，因此可以维护多个账本，但没有账本数据可以从一个通道传递到另一个通道。按通道划分的账本是通过配置链码、身份成员服务和gossip数据传播协议来定义和实现的。包括交易、账本状态和通道成员信息在内的数据传播仅限于在通道上具有可验证成员资格的peer。通过通道隔离peer和账本数据，允许需要私有和保密交易的网络成员在同一个区块链网络上与商业竞争对手和其他受限制成员共存。
