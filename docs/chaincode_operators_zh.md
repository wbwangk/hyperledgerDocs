
[原文](http://hyperledger-fabric.readthedocs.io/en/latest/chaincode4noah.html)  

## 链码运维
[原文](http://hyperledger-fabric.readthedocs.io/en/latest/chaincode4noah.html)  
链码是一个程序，用Go、Node.js编写(未来会支持其他语言，如Java)，实现了一个规定的接口。链码运行在安全的Docker容器中，隔离于背书peer过程。链码通过应用提交的事务来初始化和管理账本状态。  
链码处理网络成员都同意的业务逻辑，所以它可以被认为是“智能合约”。链码创建的状态是不能直接被其他链码访问的（scoped）。但在同一个网络内（一般指通道？），通过适当的授权，一个链码可以调用其它链码而从访问它的状态。  
本章假定了一个叫诺亚的运维工程师，通过他的视角关注链码。根据诺亚的喜好，我们专注于链码的全生命周期维护，即包装、安装、实例化和升级链码的过程。  
### 链码生命周期
Hyperledger Fabric API允许与区块链网络中的不同节点(peer、orderer和MSP)交互，它还允许在背书peer节点上打包、安装、实例化和升级链码。Hyperledger Fabric各语言SDK对Hyperledger Fabric API进行抽象以利于应用开发，所以它可以用于管理链码生命周期。此外，Hyperledger Fabric API还可以通过CLI直接访问，这在本章我们会用到。  
我们提供了四个命令去管理链码生命周期：`package`、`install`、`instantiate`和`upgrade`。在未来的版本中，我们正考虑增加`stop`和`start`事务去禁用和重新启用链码，而不用实际卸载它。在链码被成功安装和实例化后，链码是活跃状态(运行中)，可以通过 `invoke`事务处理事务。链码可以在安装后多次升级（版本更新）。  
### 打包
链码包由三部分组成：
- 链码，就象在`ChaincodeDeploymentSpec`(简称CDS)中定义的。CDS通过code和其它属性(如名称和版本)来定义链码包  
- 一个可选的实例化策略，这有时被称为背书策略  
- 链码“拥有者”（实体）的一组数字签名  

签名用于以下目的：  
- 建立链码的所有权  
- 允许验证包裹的内容  
- 允许检测包裹篡改  
链码实例化事务的创建者需要通过链码的实例化策略的验证。  

#### 建包
有两个方法对链码打包，复杂的和简单的。当链码具有多个拥有者时，它需要被多个身份签名。这需要我们首先建立一个签名的链码包(`SignedCDS`)，然后发给其他拥有者进行签名。这是一个复杂流程。
简化流程是，当你部署的SignedCDS只有一个签名，而且签名者就是安装事务的发起者。（即安装一个自己签名的包到自己的peer）  
先讲复杂流程。  
创建一个签名的链码包，适用下列命令：  
```
$ peer chaincode package -n mycc -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02 -v 0 -s -S -i "AND('OrgA.admin')" ccpack.out
```
`-s`选项表示创建一个多拥有者签名的包，如果不加就简单创建一个纯CDS。当指定了`-s`选项，如果有其他拥有者需要签名，则`-S` 选项必须设置。否则，会创建一个仅包含实例化策略的SignedCDS。  
`-S`选项使处理流程使用 `core.yaml`文件中`localMspid`属性下定义的MSP身份对包进行签名。  
`-S`选项使可选的。但如果一个包没有签名，它就不能被其他拥有者使用`signpackage`命令进行签名。  
`-i`选项用于为链码指定实例化策略。实例化策略与背书策略的格式相同，都是指定哪些身份可以实例化这个链码。在上面的例子中，只有`OrgA`的管理员(admin)可以实例化这个链码。如果没有设置策略，会使用默认策略，则只允许peer的MSP的管理员身份去实例化链码。  
#### 包签名
一个创建时签名的链码包可以被移交给其他拥有者查看和签名。流程支持out-of-band对链码包签名。  

[ChaincodeDeploymentSpec](https://github.com/hyperledger/fabric/blob/master/protos/peer/chaincode.proto#L78)可以选择被集体拥有者签名，而从创建一个[SignedChaincodeDeploymentSpec](https://github.com/hyperledger/fabric/blob/master/protos/peer/signed_cc_dep_spec.proto#L26)(或叫SignedCDS)。SignedCDS包含3个元素：  
 1. CDS包含的链码源码、名称和版本号。  
 2. 一个链码的实例化策略，表述为背书策略。  
 3. 链码拥有者列表，通过[背书](https://github.com/hyperledger/fabric/blob/master/protos/peer/proposal_response.proto#L111)定义。    

*【注意】：当链码在一些通道实例化时，这个背书策略通过out-of-band确定MSP身份。如果实例化策略没有指定，默认策略是通道的任何MSP管理员。*  

每个拥有者都对`ChaincodeDeploymentSpec`进行背书，背书方法是对CDS与拥有者身份（如证书）的组合结果进行签名(算法：sign(ProposalResponse.payload + endorser))。  
一个链码拥有者使用下面的命令对以前创建的签名包进行签名：
```
$ peer chaincode signpackage ccpack.out signedccpack.out
```
`ccpack.out`和`signedccpack.out`分别是输入包和输出包。`signedccpack.out`中包含了一个对包的新增签名，签名使用了本地MSP。  

#### 安装链码
安装(`install`)事务按规定格式对链码的源码进行打包，这个格式称为`ChaincodeDeploymentSpec`（或称CDS），该事务将链码安装在将来要运行它的peer节点上。  

*【注意】：你必须将链码安装在要运行链码的通道的每个背书peer节点上。*  

当`install`API简单给予了一个`ChaincodeDeploymentSpec`，它将使用默认实例化策略和包含一个空的拥有者列表。  

*【注意】：为了保证链码逻辑对网络上的其他成员保密，链码只安装在链码拥有者的背书peer节点上（可能存在一个或多个拥有者）。哪些没有链码的成员，不能是链码事务的背书者；也就是说，他们不能执行链码。然而，他们仍然可以生效和提交事务到账本。*  

为了安装链码，发送一个[SignedProposal](https://github.com/hyperledger/fabric/blob/master/protos/peer/proposal.proto#L104)到`lifecycle system chaincode`(LSCC)(LSCC会在[系统链码](https://github.com/wbwangk/wbwangk.github.io/wiki/Hyperledger#%E7%B3%BB%E7%BB%9F%E9%93%BE%E7%A0%81)一节中描述)。例如，使用CLI安装**sacc**示范链码（前文在“链码教程:链码开发-[调试与测试](https://github.com/wbwangk/wbwangk.github.io/wiki/Hyperledger#%E8%B0%83%E8%AF%95%E4%B8%8E%E6%B5%8B%E8%AF%95)”一节中描述过）的命令如下：
```
$ peer chaincode install -p chaincodedev/chaincode/sacc -n mycc -v 0
```
CLI内部为**sacc**创建一个`SignedChaincodeDeploymentSpec`，并发送它到本地peer，peer调用LSCC上的`Install`方法。`-p`选项指定了链码的路径，它必须位于用户`GOPATH`的源码树上，如`$GOPATH/src/sacc`。[CLI](https://github.com/wbwangk/wbwangk.github.io/wiki/Hyperledger#cli)一节有这个命令选项的详细描述。  
请注意，为了安装在peer上，SignedProposal的签名必须来自peer的本地MSP管理员之一。  

#### 实例化
`instantiate`事务调用`lifecycle System Chaincode`(LSCC)在一个通道上创建和实例化某个链码。这是一个链码-通道绑定过程：一个链码可以绑定到任意数量的通道，独立和互不依赖地运行在每个通道上。换句话说，无论链码在多少个其他通道上安装和实例化，对于提交事务的通道状态是隔离的。  
`instantiate`事务的创建者必须满足包含在SignedCDS中的链码实例化策略，必须还是通道的写入者（这是通道创建时的配置之一）。这对于通道安全很重要，可以阻止恶意实体部署链码和欺骗成员执行非绑定通道的链码。  
例如，回想一下，默认实例化策略是任何通道MSP管理员，因此链码实例化事务的创建者必须是通道管理员的成员。事务提议到达背书者时，会根据实例化策略验证创建者的签名。在提交到账本之前，在事务生效期间再次执行此操作。  
实例化事务还为通道上的链码建立了背书策略。背书策略描述了事务结果可以被通道成员接受的证据需求。  
例如，使用CLI实例化**sacc**链码和用`john`和`0`初始化状态，命令如下：
```
$ peer chaincode instantiate -n sacc -v 1.0 -c '{"Args":["john","0"]}' -P "OR ('Org1.member','Org2.member')"
```
*【注意】上面的背书策略(CLI使用波兰语表示法)，所有的**sacc**事务需要一个Org1成员或Org2成员的背书。就是说，为了事务生效，Org1或Org2需要对调用(Invoke)**sacc**的执行结果签名。*   
实例化成功后，通道中的链码进入活动状态，准备好处理任意[ENDORSER_TRANSACTION](https://github.com/hyperledger/fabric/blob/master/protos/common/common.proto#L42)类型的事务提议。当事务到达背书peer时，它们会被并发处理。  
#### 版本更新
链码可以在任何时间更新版本，版本是SignedCDS的组成部分。SignedCDS的其它部分，如拥有者和实例化策略是可选项。然而，链码名称必须相同，否则它会被视为完全不同的链码。  
版本更新前，链码的新版本必须已经在背书者节点上安装。更新是一个类似于实例化的事务，它绑定新版本的链码到通道。绑定链码旧版本的通道仍然运行旧版本。话句话说，`upgrade`事务仅影响提交了更新事务的通道。  

*【注意】，由于链码的多个版本可能同时有效，更新过程不会自动删除就版本，因此用户必须临时管理它。*  

更新事务还是与`instantiate`事务由细微的不同：`upgrade`事务检查当前链码实例化策略，不是新策略(如果指定了策略)。这确保了只有在当前实例化策略中存在的成员才可以更新链码。  

*【注意】，在更新时，链码的`Init`函数将被调用去执行相关数据更新或重新初始化，所以链码更新时要小心避免重置状态。*  

#### 停止和启动
注意`stop`和`start`生命周期事务还没有被实现。然而，你可以手工停止链码，办法是从每个背书者peer删除链码容器和SingedCDS包。在每个运行背书peer节点的主机或虚机上删除链码容器，然后删除SignedCDS。  
```
(注意，为了从peer节点删除CDS，你需要先进入peer节点的容器。我们提供了干这个的工具脚本)
$ docker rm -f <container id>
$ rm /var/hyperledger/production/chaincodes/<ccname>:<ccversion>
```
停止在用于以受控方式进行升级的工作流程中是有用的，其中链码可以在发布升级之前在所有peer的信道上停止。  
#### CLI

*【注意】：我们正在评估是否发布平台专属Hyperledger Fabric peer二进制包。在此之前，你可以在一个docker容器中简单调用命令。*  

为了显示当前可用的CLI命令，在运行中的`fabric-peer`Docker容器中执行下列命令：
```
$ docker run -it hyperledger/fabric-peer bash
(peer chaincode --help)
```
它将显示类似的以下输出：
```
Usage:
  peer chaincode [command]

Available Commands:
  install     Package the specified chaincode into a deployment spec and save it on the peer's path.
  instantiate Deploy the specified chaincode to the network.
  invoke      Invoke the specified chaincode.
  list        Get the instantiated chaincodes on a channel or installed chaincodes on a peer.
  package     Package the specified chaincode into a deployment spec.
  query       Query using the specified chaincode.
  signpackage Sign the specified chaincode package
  upgrade     Upgrade chaincode.

Flags:
    --cafile string      Path to file containing PEM-encoded trusted certificate(s) for the ordering endpoint
-h, --help               help for chaincode
-o, --orderer string     Ordering service endpoint
    --tls                Use TLS when communicating with the orderer endpoint
    --transient string   Transient map of arguments in JSON encoding
```
为了方便在脚本式应用中使用，`peer`命令在失败事件中总是产生非零的返回码。  
链码命令的例子：  
```
peer chaincode install -n mycc -v 0 -p path/to/my/chaincode/v0
peer chaincode instantiate -n mycc -v 0 -c '{"Args":["a", "b", "c"]}' -C mychannel
peer chaincode install -n mycc -v 1 -p path/to/my/chaincode/v1
peer chaincode upgrade -n mycc -v 1 -c '{"Args":["d", "e", "f"]}' -C mychannel
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","e"]}'
peer chaincode invoke -o orderer.example.com:7050  --tls --cafile $ORDERER_CA -C mychannel -n mycc -c '{"Args":["invoke","a","b","10"]}'
```
### 系统链码
系统链码与普通链码具有相同的编程模型，知识它运行在peer进程中，而不是在隔离的容器中。因此，系统链码构建在peer可执行文件中，它不会遵循上述同样的生命周期。特别是，安装、实例化和版本更新不会用在系统链码上。  
系统链码的目的是减少peer和链码间gRPC通信成本，和管理灵活性的折中。例如，系统链码只能与peer程序一起更新。它必须以一套固定参数注册，且不能有背书策略或背书策略函数。  
Hyperledger Fabric中使用系统链代码来实现许多系统行为，以便系统集成商可以根据需要替换或修改它们。  
当前系统链码的列表：  
1. [LSCC](https://github.com/hyperledger/fabric/tree/master/core/scc/lscc) 生命周期系统链码，处理上述的生命周期请求。  
2. [CSCC](https://github.com/hyperledger/fabric/tree/master/core/scc/cscc) 配置系统链码，处理peer端的通道配置。  
3. [QSCC](https://github.com/hyperledger/fabric/tree/master/core/scc/qscc) 查询系统链码，提供账本查询API，例如获取区块和事务。  
4. [ESCC](https://github.com/hyperledger/fabric/tree/master/core/scc/escc) 背书系统链码，通过签署事务提议响应来处理背书。  
5. [VSCC](https://github.com/hyperledger/fabric/tree/master/core/scc/vscc) 生效系统链码，处理事务生效，包括检查背书策略和多版本并发控制。  

更改或覆盖这些系统链码要小心，特别是LSCC、ESCC和VSCC，因为它们处在主事务的运行路径上。值得注意的是，VSCC将区块将提交到账本之前的生效验证，通道中的所有peer计算相同的生效验证以避免账本分歧（非确定性）是很重要的。因此，如果修改或更换VSCC，需要特别小心。  












