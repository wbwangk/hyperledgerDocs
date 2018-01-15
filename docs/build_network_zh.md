[原文](http://hyperledger-fabric.readthedocs.io/en/latest/build_network.html)  

## 启动首个网络(first-network)
本节遵循hyperledger官方文档“[Building Your First Network](http://hyperledger-fabric.readthedocs.io/en/latest/build_network.html)  
first-network提供了一个脚本来帮助初学者体验fabric，它就是`byfn.sh`。可以通过帮助命令看看它的功能：
```
$ cd /opt/fabric-samples/first-network
$ ./byfn.sh --help
 byfn.sh -m up|down|restart|generate [-c <channel name>] [-t <timeout>] [-d <delay>] [-f <docker-compose-file>] [-s <dbtype>]
```
### 生成网络工件
generate required certificates and genesis block
fabric的网络和通道具有类似的含义。通道可以视为一种虚拟网络，多个通道就多个虚拟网络。docker支持overlay网络，为同一个peer加入不同的网络创造了底层技术基础（以上认识还没有得到确认）。
```
$ ./byfn.sh -m generate
Generating certs and genesis block for with channel 'mychannel' and CLI timeout of '10000'
Continue (y/n)?y
(后略)
```
`byfn.sh`会在屏幕上有很多输出，通过这些文字也可以知道该脚本的功能：
1. 生成了Orderer的创世区块。在排序节点上fabric维护了一个“系统账本”，保存了整个fabric区块链网络的参数、元数据等。对于区块链来说，第一个块(有时称0号区块)被称为创世区块(Genesis block)。该创世区块对应了一个文件，一般是`genesis.block`。  
2. 创建了一个通道。生成了一个叫`channel.tx`的文件。  
3. 生成了组织Org1MSP和Org2MSP的锚peer。锚peer用于跨组织的通信。  

### 启动网络
```
$ ./byfn.sh -m up
（适当删减）
Channel "mychannel" is created successfully =====================
PEER0 joined on the channel "mychannel" =====================
PEER1 joined on the channel "mychannel" =====================
PEER2 joined on the channel "mychannel" =====================
PEER3 joined on the channel "mychannel" =====================
Anchor peers for org "Org1MSP" on "mychannel" is updated successfully =====================
Anchor peers for org "Org2MSP" on "mychannel" is updated successfully =====================
Chaincode is installed on remote peer PEER0 =====================
Chaincode is installed on remote peer PEER2 =====================
Chaincode Instantiation on PEER2 on channel 'mychannel' is successful =====================
Querying on PEER0 on channel 'mychannel'... =====================
Invoke transaction on PEER0 on channel 'mychannel' is successful =====================
Chaincode is installed on remote peer PEER3 =====================
Querying on PEER3 on channel 'mychannel'... =====================
========= All GOOD, BYFN execution completed ===========
```
启动后屏幕仍被锁定为日志输出，可以打开另外的终端窗口进行后续的操作。如，查看容器清单：
```
$ docker ps
```
可以看到容器有：
1. cli(fabric-tools)，是fabric的命令行工具  
2. peer0.org1.example.com等(fabric-peer)，是peer节点的进程容器  
3. orderer.example.com(fabric-orderer)，是orderer节点的进程容器  
4. dev-peer0.org1.example.com-mycc-1.0-xxxx， 是链码容器
在其他的fabric环境中(如生产环境下)，还可能看到ca-server的容器、couchdb的容器等。  

### 停止网络
```
$ ./byfn.sh -m down
$ docker ps
```
网络停止后，通过docker ps命令可以看到所有的容器都消失了。

### 密钥生成器(Crypto Generator)
我们用`cryptogen`工具为各种的网络实体生成密码学文件(x509证书和签名密钥)。这些证书表达身份，对实体间通信和事务认证进行签名和验证。  

#### 它是如何工作的？
Cryptogen的配置文件是`crypto-config.yaml`，该文件包括网络拓扑，允许我们为组织以及属于组织的组件生成一系列证书和密钥。执行示范：
```
$ cryptogen generate --config=./crypto-config.yaml
```
运行后会自动创建一个`crypto-config`目录，里面有很多密码学文件。在生成的文件中，每个组织都会分配一个根证书(`ca-cert`)，该证书绑定特殊组件(peer和orderer)到组织。假定每个组织都有一个唯一的CA证书，我们模仿了一个典型的网络，其中每个[成员](http://hyperledger-fabric.readthedocs.io/en/latest/glossary.html#member)拥有自己的CA。在Hyperledger Fabric中，实体使用自己的私钥(`keystore`)对事务和通信进行签名，并用对方的公钥(`signcerts`)验证签名。  
配置文件中`Template`有个`count`变量，我们用它来指定每个组织下的peer数量；在我们示例中，每个组织下有两个peer。`Users`下也有`count`变量，它表示创建的用户数量。示例如下：
```yaml
  - Name: Org2
    Domain: org2.example.com
    Template:
      Count: 2
    Users:
      Count: 1
```

我们在本文不会详述[X509证书和PKI](https://en.wikipedia.org/wiki/Public_key_infrastructure)。  
在`crypto-config.yaml`文件中，注意`OrdererOrgs`之下的“Name”, “Domain” and “Specs”参数。网络实体的命名约定是：`{{.Hostname}}.{{.Domain}}`。例如，排序节点的名称是`orderer.example.com`，这关联了一个MSP ID `Orderer`，关于MSP的更多细节参考[成员服务提供者(MSP)](https://github.com/wbwangk/wbwangk.github.io/wiki/Fabric%E8%BF%90%E7%BB%B4#%E6%88%90%E5%91%98%E6%9C%8D%E5%8A%A1%E6%8F%90%E4%BE%9B%E8%80%85msp)文档。  
在`cryptogen`生成的加密材料中有管理员的，这里有篇文章[寻找管理员的证书和私钥](https://github.com/wbwangk/wbwangk.github.io/wiki/Fabric%E7%AC%94%E8%AE%B0#%E5%AF%BB%E6%89%BE%E7%AE%A1%E7%90%86%E5%91%98%E7%9A%84%E8%AF%81%E4%B9%A6%E5%92%8C%E7%A7%81%E9%92%A5)，专门研究了`cryptogen`生成的管理员的加密材料。  
 
### 配置事务生成器
工具`configtxgen`用于生成四个配置工件：
- 排序器(orderer)`genesis block`
- 通道`configuration transaction`  
- 两个`anchor peer transactions`，每个组织生成一个  
关于`configtxgen`更多细节参考[Channel Configuration (configtxgen)](http://hyperledger-fabric.readthedocs.io/en/latest/configtxgen.html)。  

生成的四个工件中的`genesis block`是排序服务的[创世区块](#创世区块)。[通道](#通道)的`configuration transaction`文件在通道创建时被广播到orderer。至于`anchor peer transactions`，顾名思义，定义组织在这个通道的[锚点peer](#锚点peer)。  

#### 工作原理

Configtxgen会根据文件`configtx.yaml`定义的配置运行，文件包含了示范网络的定义。其中有三个成员：一个排序器组织(`OrdererOrg`)和两个Peer组织(`Org1`和`Org2`)，每个Peer组织管理和维护了两个peer节点。这个文件还定义了一个联盟(`SampleConsortium`)，联盟包含了两个peer组织。特别需要注意文件开头的“Profiles”小节。你应该注意到了它有两个唯一的头。一个是排序器创世区块(`TwoOrgsOrdererGenesis`)，另一个是通道(`TwoOrgsChannel`)。  

这些头很重要，当生成工件时，它们将作为参数发送。  
*注释：我们的`SampleConsortium`联盟定义在系统级profile，然后被通道集profile引用。通道将存在于整个联盟范围。*  

这个文件还包含了两个值得注意的附加内容。首先，为每个peer组织(`peer0.org1.example.com`和`peer0.org2.example.com`)定义了锚点peer。其次，定义了每个成员的MSP目录地址，这让我们在排序器创世区块中保存了每个组织的根证书。这是一个关键概念。现在与排序服务通信的任何网络实体可以被验证其数字签名了。  

### 运行工具
你可以使用`configtxgen`和`cryptogen`工具手工生成证书/密钥和不同的配置工件。或者，你也可以修改`byfn.sh`脚本来达到你的目的。  

#### 手工生成工件
你可以参考byfn.sh脚本中的generateCerts函数，里面的命令可以根据`crypto-config.yaml`文件中定义的网络配置生成证书。  

首先，让我们运行`cryptogen`工具。它位于`bin`目录，所以要使用相对路径了执行它（或者修改操作系统的PATH）:
```
$ ../bin/cryptogen generate --config=./crypto-config.yaml
org1.example.com
org2.example.com
```
证书和密钥(即MSP文书)会被输出到`crypto-config`目录（位于`first-network`目录之下）。  
生成的加密材料主要是管理员的身份证明文件，其中：
- 系统管理员:`crypto-config/ordererOrganizations`目录下
- 组织管理员：org1组织管理员位于`crypto-config/peerOrganizations/org1.example.com`目录下
- peer管理员：org1的peer0管理员位于`crypto-config/peerOrganizations/org1.example.com/peerspeer0.org1.example.com`目录下

下一步，我们需要告诉`configtxgen`工具到哪里去寻找`configtx.yaml`文件。需要通过下面的环境变量告诉它($PWD表示当前目录)：
```
$ export FABRIC_CFG_PATH=$PWD
```
然后，我们调用`configtxgen`工具去生成排序器创世区块：
```
$ ../bin/configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block
```
需要先手工创建目录`channel-artifacts`，否则上述命令会出错。  
#### 创建一个通道配置事务
（在Fabric中通道配置信息也保存在区块链中，而区块链的内容是靠事务写入的，所以要新建通道需要创建一个事务。）  
下一步，我们需要创建通道事务工件。确保替换`$CHANNEL_NAME`或设置`CHANNEL_NAME`为环境变量，然后执行下列指令：
```
$ export CHANNEL_NAME=mychannel  
$ ../bin/configtxgen -profile TwoOrgsChannel \
 -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME
```
下一步，我们将在刚刚创建的通道上定义Org1的锚点peer。同样，确保覆盖`$CHANNEL_NAME`或设置为下面的命令设置环境变量。
```
$ ../bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate \
./channel-artifacts/Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP
2017-12-07 08:59:42.756 UTC [common/tools/configtxgen] main -> INFO 001 Loading configuration
2017-12-07 08:59:42.762 UTC [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 002 Generating anchor peer update
2017-12-07 08:59:42.762 UTC [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 003 Writing anchor peer update
```
现在，我们将在同一个通道中定义Org2的锚点peer：
```
$ ../bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate \
./channel-artifacts/Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP
```
总结一下本节，执行了上述一系列命令后，`crypto-config`目录生成了一些证书和密钥；`channel-artifacts`下生成了一个创世区块文件和3个事务文件：
```
$ ls ./channel-artifacts
channel.tx  genesis.block  Org1MSPanchors.tx  Org2MSPanchors.tx
```
（执行到这里的时候，就相当于执行了`byfn.sh -m generate`）  

### 启动网络
我们将使用docker-compose脚本启动我们的网络。docker-compose会使用我们之前下载的docker镜像，用我们之前生成`genesis.block`(创世区块)引导排序器(orderer)。  
在启动网络前，打开`docker-compose-cli.yaml`文件，注释掉CLI容器的`script.sh`。让你的`docker-compose-cli.yaml`文件变成这样：
```
working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
# command: /bin/bash -c './scripts/script.sh ${CHANNEL_NAME}; sleep $TIMEOUT'
volumes
```
如果不注释掉这一样，脚本就会利用CLI命令把网络启动起来了。然而，我们的目的是手工执行这些命令，以便解释这些调用的语法和功能。  
CLI默认超时是10000秒。如果你需要容器存在的更久，需要通过设置`TIMEOUT`环境变量覆盖这一默认值。  

启动网络(确保用命令`docker ps -a`看不到任何容器)：
```
$ TIMEOUT=10000 CHANNEL_NAME=$CHANNEL_NAME docker-compose -f docker-compose-cli.yaml up -d
```
如果你想看到网络的实时日志，就不要加上`-d`标志。如果不加`-d`表示，你需要另外打开一个终端窗口了执行CLI。  

#### 环境变量
为了通过CLI命令让`peer0.org1.example.com`工作起来，我们需要准备4个环境变量。这些变量之前被“烧入”了`peer0.org1.example.com`CLI容器中，所以我们不用输入它们就能操作。但如果你需要调用其他peer或orderer，则需要正确的设置这些变量值。查看`docker-compose-base.yaml`文件可以看到这四个变量的值：
```
# Environment variables for PEER0

CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
CORE_PEER_ADDRESS=peer0.org1.example.com:7051
CORE_PEER_LOCALMSPID="Org1MSP"
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
```
#### 创建和加入通道
回想一下上面我们在[创建一个通道配置事务](#创建一个通道配置事务)一节中使用`configtxgen`工具创建通道配置事务。你可以重复那个过程来创建另外的通道配置事务，使用相同或不同的profile参数(在`configtx.yaml`中定义)传递给`configtxgen`。你可以重复这一过程，用来在网络中创建其他通道。  
用下列命令进入CLI容器：
```
$ docker exec -it cli bash
root@0d78bb69300d:/opt/gopath/src/github.com/hyperledger/fabric/peer#
```
首先进入的是在`docker-compose.ymal`中定义的`working_dir`目录。下面用`$$`表示在容器内的命令行操作。  
在之前[创建一个通道配置事务](#创建一个通道配置事务)一节中我们创建了通道配置事务工件("channel.txt")，下面我们把它作为创建通道请求的一部分发给orderer。  
我们用`-c`标志指定通道名称，用`-f`标志指定通道配置事务。在这里它叫`channel.txt`，然而你可以用其他名字来挂载你自己的通道配置事务。我们又一次在CLI容器内设置`CHANNEL_NAME`环境变量，所以不用显式地传递这个参数。  
```
$$ export CHANNEL_NAME=mychannel
$$ export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
$$ peer channel create -o orderer.example.com:7050 -c $CHANNEL_NAME -f \
./channel-artifacts/channel.tx --tls --cafile $ORDERER_CA
```
注意此命令行中的`--cafile`参数，它是一个指向orderer根CA证书的本地路径，用于验证TLS握手。  
命令会返回一个创世区块(`<channel-ID.block>`)，我们用它加入通道。它里面包含了在`channel.tx`中定义的配置信息，如果你没有改变过通道名称，该命令将返回一个叫`mychannel.block`的proto。在当前目录下可以看到这个`mychannel.block`文件。  
现在，让我们把`peer0.org1.example.com`节点加入通道：
```
$$ peer channel join -b mychannel.block
```
你还可以将其他peer加入通道，方法是修改之前在[环境变量](#环境变量)一节中提到的4个环境变量。如果你用`env`命令查看一下环境变量，会发现那4个环境变量的值都是`peer0.org1.example.com`对应的。  
我们不将每个peer都加入网络，而是将`peer0.org2.example.com`加入网络，这样我们可以修改通道的锚点peer定义。我们用下面的命令将预先烧制在CLI容器中的4个环境变量替换掉:
```
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp CORE_PEER_ADDRESS=peer0.org2.example.com:7051 CORE_PEER_LOCALMSPID="Org2MSP" CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt peer channel join -b mychannel.block
```
#### 更新锚点peer 
下面的命令是通道变更，他们会广播到通道定义。本质上，我们会在通道创始区块的上面追加配置信息。注意，我们没有改变创始区块，只是向链中添加了定义锚点peer的delta。  
变更通道定义，将`peer0.org1.example.com`定义为Org1的锚点peer。
```
$$ peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile $ORDERER_CA
```
现在，变更通道定义，将`peer0.org2.example.com`定义为Org2的锚点peer。注意命令中的4个环境变量用于覆盖默认的`peer0.org1.example.com`的相关值。
```
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp CORE_PEER_ADDRESS=peer0.org2.example.com:7051 CORE_PEER_LOCALMSPID="Org2MSP" CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org2MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```
#### 安装和实例化链码
*注意，我们简单实用了一个已有链码。如果想学习链码开发，请参考[链码开发](链码教程:链码开发)一章。*  
应用通过链码与区块链账本交互。我们需要安装链码到那些执行和为事务背书的peer上，并在通道上实例化链码。  
首先，安装Go语言示范链码到4个peer节点的某个上。这些命令会将源码放到指定peer的文件系统上。  
*注意，对于每个链码名称和版本，你只能安装一个版本的源码。源码以链码的名称和版本号为上下文存在于peer的文件系统中，它不关注语言。*  
```
$$ peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/
```
下一步在通道上实例化链码。这将在通道上初始化链码、为链码设置背书策略和为目标peer启动一个链码容器。注意`-P`参数。这是链码的背书策略，用于对链码事务进行验证。  
在下面的命令中，你注意到了我们将策略设置为`-P "OR ('Org0MSP.member','Org1MSP.member')"`。这意味着我们需要从Org1或Org2的peer上获得一个背书。如果把`OR`改成`AND`，则表示我们需要两个背书。  
```
$$ peer chaincode instantiate -o orderer.example.com:7050 --tls --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "OR ('Org1MSP.member','Org2MSP.member')"
```
关于背书策略的细节可以看到[背书策略](http://hyperledger-fabric.readthedocs.io/en/latest/endorsement-policies.html)。  
如果你想更多的peer与账本交互，你需要将它们加入通道，安装同样名字、版本和语言的链码到相应peer的文件系统。一个链码容器被在peer上启动，然后就可以与相应链码交互了。需要知道的是，Node.js镜像的编译相对较慢。  
当链码在通道上被实例化后，我们只需传入通道id和链码名称来访问它。  
#### 查询
让我们查询一下键`a`的值，以便确认链码已经实例化和状态数据库已经填充。查询语句如下：
```
$$ peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
Query Result: 100    (其它信息省略)
```
#### 调用(Inoke)
现在让我们把`a`的10个给`b`（即a减少10，b增加10）。这个事务会切割一个新区块并更新状态数据库。调用的语法如下：
```
$$ peer chaincode invoke -o orderer.example.com:7050  --tls --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc -c '{"Args":["invoke","a","b","10"]}'
``` 
然后分别查询一下`a`和`b`的值：
```
$$ peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
Query Result: 90    (其它信息省略)
$$ peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","b"]}'
Query Result: 210    (其它信息省略)
```
#### 这演示了什么？
为了对账本进行读写操作，peer必须安装链码。此外，链码容器并没有启动，直到对链码进行初始化或执行读写事务(如查询`a`的值)。这些事务促使容器启动。而且，通道中的所有peer会维持一个账本的完全副本，其中包括不可修改、区块中的顺序记录，以及状态数据库(其中维护了当前状态的快照)。这包含没有安装链码的peer（就像上面例子中的`peer1.org1.example.com`peer)。 最终，链码在安装后可以访问(就像上面例子中的`peer1.org2.example.com`)，因为它已经被实例化。  
#### 怎么看到这些事务？
检查CLI docker容器的日志(需要先通过exit命令先退出容器，返回到宿主操作系统):
```
$ docker logs -f cli
```
但，我的cli容器的日志是空的！原因不明
#### 怎么看到链码日志？
用`docker logs`命令查看不同链码容器的日志，来分别查看各个事务的日志。需要先用`docker ps`命令找到链码容器id。下面是刚刚测试的链码容器日志：
```
docker logs 3daea3abfab2
ex02 Init
Aval = 100, Bval = 200
ex02 Invoke
Query Response:{"Name":"a","Amount":"100"}
ex02 Invoke
Aval = 90, Bval = 210
ex02 Invoke
Query Response:{"Name":"a","Amount":"90"}
ex02 Invoke
Query Response:{"Name":"b","Amount":"210"}
```
### 理解docker-comopse拓扑
BYFN范例提供了两种风格的Docker Compose文件，都是从`docker-compose-base.yaml`(位于`base`目录)扩展而来。第一种风格是`docker-compose-cli.yaml`，提供了一个CLI容器,以及一个orderer和4个peer。我们在这个文章中主要使用这个文件。  

*注释：本文剩余部分的内容主要讲一个为了SDK设计的docker-compose文件。更多细节参考[Node SDK库](https://github.com/hyperledger/fabric-sdk-node)。*
  
第二种风格的是`docker-compose-e2e.yaml`，这个用于使用Node.js SDK进行的端到端测试。为了使用SDK，它的主要不同是包含一个运行fabric-ca服务器的容器。因此，我们可以发送REST请求到组织的CA，用来进行用户的登记(registration)和注册(enrollment)。  
如果你想使用`docker-compose-e2e.yaml`而不运行`byfn.sh`脚本，需要进行4个小修改。我们需要指出组织CA的私钥。你可以指出私钥在crypto-config目录中的位置。例如，Org1的私钥是路径`crypto-config/peerOrganizations/org1.example.com/ca/`。这个私钥的文件名是一个以`_sk`结尾的长哈希值。Org2的私钥路径是`crypto-config/peerOrganizations/org2.example.com/ca/`。  
在`docker-compose-e2e.yaml`中为ca0和ca1修改`FABRIC_CA_SERVER_TLS_KEYFILE`变量。你还需要修改启动ca服务器的命令路径。你需要为每个CA容器提供同样的私钥两次。  
### 使用CouchDB
状态数据库可以从默认(goleveldb)切换到CouchDB。同样的链码函数可以用于CouchDB，然而，当把链码数据建模为JSON后，还可以对状态数据库的数据内容执行丰富而复杂的查询。  
使用CouchDB代替默认数据库(goleveldb)，与之前描述相同步骤生成工件，除了启动网络时使用`docker-compose-couch.yaml`：
```
CHANNEL_NAME=$CHANNEL_NAME TIMEOUT=<pick_a_value> docker-compose -f docker-compose-cli.yaml -f docker-compose-couch.yaml up -d
```
下面的链码**chaincode_example02**将使用CouchDB。  

*注释：如果你选择了将fabric-couchdb容器的端口映射到主机端口，请确保端口的远程访问是安全的。在开发环境下映射端口使CouchDB REST API可用，并使通过CoutchDB web接口(Fauxton)使数据库可见。在生产环境下进行端口映射要慎重，需要限制从外部访问CouchDB容器的端口。*   

你可以用**chaincode_example02**链码访问CouchDB状态数据库，就像前面讲的那样。但为了执行CouchDB特性的查询，你需要使用数据模型为JSON的链码(如**marbles02**)。你可以在`fabric/examples/chaincode/go`目录找到**marbles02**链码。
我们可以使用相同的步骤创建和加入通道，就像前面在[创建和加入通道](https://github.com/wbwangk/wbwangk.github.io/wiki/Hyperledger#创建和加入通道)一节中讲的那样。一旦将peer加入了通道，使用下面的步骤与**marbles02**链码交互：
1. 在`peer0.org1.example.com`上安装和实例化链码:  
```
$$ peer chaincode install -n marbles -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/marbles02
$$ peer chaincode instantiate -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -v 1.0 -c '{"Args":["init"]}' -P "OR ('Org0MSP.member','Org1MSP.member')"
```
2. 创建一些弹珠并移动它们：
```
$$ peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -c '{"Args":["initMarble","marble1","blue","35","tom"]}'
$$ peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -c '{"Args":["initMarble","marble2","red","50","tom"]}'
$$ peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -c '{"Args":["initMarble","marble3","blue","70","tom"]}'
$$ peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -c '{"Args":["transferMarble","marble2","jerry"]}'
$$ peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -c '{"Args":["transferMarblesBasedOnColor","blue","jerry"]}'
$$ peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -c '{"Args":["delete","marble1"]}'
```
3. 如果你选择在docker-compse中映射CouchDB端口，你现在就可以通过CouchDB的web接口(Fauxton)查询状态数据库，方法是打开一个浏览器并导航到下列URL：
```
http://localhost:5984/_utils
```
你可以看到一个叫`mychannel`的数据库(或你自己定义的通道名称)和里面的文档数据。

*注释：你需要更新$CHANNEL_NAME为合适的值*   

你可以通过CLI运行一般查询(如读`marble2`):
```
$$ peer chaincode query -C $CHANNEL_NAME -n marbles -c '{"Args":["readMarble","marble2"]}'
Query Result: {"color":"red","docType":"marble","name":"marble2","owner":"jerry","size":50}
```
你可以查询一个特定弹珠的历史，如`marble1`:
```
$$ peer chaincode query -C $CHANNEL_NAME -n marbles -c '{"Args":["getHistoryForMarble","marble1"]}'
Query Result: [{"TxId":"1c3d3caf124c89f91a4c0f353723ac736c58155325f02890adebaa15e16e6464", "Valu
```
你还可以对数据内容执行富文本查询，如查询拥有者`jerry`的弹珠字段：
```
peer chaincode query -C $CHANNEL_NAME -n marbles -c '{"Args":["queryMarblesByOwner","jerry"]}'
Query Result: [{"Key":"marble2", "Record":{"color":"red","docType":"marble","name":"marble2","owner"
```
### 为什么是CouchDB？
CouchDB是一种NoSQL解决方案。它是一个面向文档的数据库，其中文档字段被存储为键值对。字段可以是简单的键/值对，列表或映射。除了LevelDB支持的keyed/composite-key/key-range查询外，CouchDB还支持完整的富文本查询功能，例如对整个区块链数据的非键查询，因为其数据内容以JSON格式存储，完全可查询。因此，CouchDB可以满足不受LevelDB支持的许多用例的链码，审计和报告要求。

CouchDB还可以增强区块链中合规性和数据保护的安全性。因为它能够通过过滤和屏蔽事务内的属性来实现字段级别的安全性，并且在需要时授权只读权限。  

另外，CouchDB属于CAP定理的AP类型（Availability和Partition Tolerance）。它使用主 - 主复制模型。有关更多信息，请参阅CouchDB文档的“[ 最终一致性](http://docs.couchdb.org/en/latest/intro/consistency.html)”页面。但是，在每个Fabric peer下，不存在数据库副本，写入数据库将保证一致性和持久性（非`Eventual Consistency`）。

CouchDB是Fabric的第一个外部可插入状态数据库，可能也会有其他外部数据库选项。例如，IBM为关系数据库启用区块链。而CP型（一致性和分区容忍）数据库也可能是需要的，以便在没有应用级保证的情况下实现数据一致性。  

### 一个关于数据持久化的备注
如果在peer容器或CouchDB容器上需要数据持久性，有一种选择是将docker宿主机中的目录挂载到容器中的相关目录中。例如，您可以在`docker-compose-base.yaml`文件中的peer容器定义中添加以下两行：
```
volumes:
 - /var/hyperledger/peer0:/var/hyperledger/production
```
对于CouchDB容器，您可以在CouchDB容器定义中添加以下两行：、
```
volumes:
 - /var/hyperledger/couchdb0:/opt/couchdb/data
```
### Troubleshooting
- 总是干净地启动网络。使用下列命令删除共建、密钥、容器和链码镜像：
```
./byfn.sh -m down
```
*注释：如果不删除旧的容器和镜像会报错。* 
 
- 如果你看到Docker错误，首先检查docker版本，然后重启docker进程。docker问题往往不好识别。例如，你可能看到的错误是不能找到挂在到容器的加密材料。  
如果你想删除镜像重新开始：
```
$ docker rm -f $(docker ps -aq)
$ docker rmi -f $(docker images -q)
```
- 如果你在创建、实例化、调用或查询命令中看到错误，确保你的通道名称和链码名称正确。
- 如果你看到下面的错误：
```
Error: Error endorsing chaincode: rpc error: code = 2 desc = Error installing chaincode code mycc:1.0(chaincode /var/hyperledger/production/chaincodes/mycc.1.0 exits)
```
看来你已经有了一个链码镜像（例如`dev-peer1.org2.example.com-mycc-1.0`或`dev-peer0.org1.example.com-mycc-1.0`)已经在运行。删除它们重试。
```
$ docker rmi -f $(docker images | grep peer[0-9]-peer[0-9] | awk '{print $3}')
```
- 如果你看到类似下面的内容：
```
Error connecting: rpc error: code = 14 desc = grpc: RPC failed fast due to transport failure
Error: rpc error: code = 14 desc = grpc: RPC failed fast due to transport failure
```
确保你正在运行的网络是“1.0.0”镜像并且tag是“latest”。  

- 如果你看到下面的错误：
```
[configtx/tool/localconfig] Load -> CRIT 002 Error reading configuration: Unsupported Config Type ""
panic: Error reading configuration: Unsupported Config Type ""
```
说明你没有正确设置环境变量`FABRIC_CFG_PATH`。configtxgen工具需要这个变量来定位configtx.yaml。返回和执行`export FABRIC_CFG_PATH=$PWD`，然后重建你的通道工件。  
- 使用`down`选项清理网络：
```
$ ./byfn.sh -m down
```
- 如果你看到一个错误说你仍有活动的端点，则需要prune你的docker网络。这将清除之前的网络，重新启动一个干净的环境：
```
$ docker network prune
```
你会看到下列信息：
```
WARNING! This will remove all networks not used by at least one container.
Are you sure you want to continue? [y/N]
```
选择`y`。

### 理解Fabric网络
应用通过API调用智能合约。智能合约托管在网络中，靠名称和版本号识别。例如，智能合约容器的名称是`dev-peer0.org1.example.com-fabcar-1.0`，其中`fabcar`是智能合约名称，`1.0`是智能合约版本号，而`dev-peer0.org1.example.com`是peer名称。  
API可以把SDK访问。SDK封装了应用与智能合约通信的接口，如查询或接收账本更新。这些API使用几个不同的网络地址，接收一些输入参数。智能合约由peer管理员安装，然后按照链码的策略被实例化到通道中。智能合约的实例化流程与普通调用的事务流程相同，背书、排序、生效、提交，之后才能与链码容器交互（智能合约实例化就是链码容器启动）。   
#### 查询
查询是最简单的调用：一个请求和响应。最常见的查询是向状态数据库查询一个key的当前值(`GetState`)。然而，[链码shim接口](https://github.com/hyperledger/fabric/blob/release/core/chaincode/shim/interfaces.go)允许不同的Get请求，如`GetHistoryForKey`或`GetCreator`。  
创建查询需要指定一个peer、一个链码、一个通道和一系列输入(如key)和一个可用的链码函数，然后通过API`chain.queryByChaincode`发送查询到peer。相应的响应值会返回给应用客户端。  
#### 更新
账本更新开始于应用创建一个事务提议。类似于查询，创建事务请求需要指定一个peer、链码、通道、函数和一系列输入。程序之后会调用API`channel.SendTransactionProposal`发送事务提议到peer寻求背书。  
网络(也就是背书peer(可能多个))会返回一个提议响应，应用使用该响应来创建和签署事务请求。通过调用API`channel.sendTransaction`，这个事务请求被发送到排序服务。排序服务将事务捆绑入一个区块，并将它发送到通道中的所有peer以求生效(Fabcar网络只有一个peer和一个通道)。  
最后应用使用两个事件处理器API：用`eh.setPeerAddr`连接到peer的事件监听者端口，用`eh.registerTxEvent`和一个特定事务ID去注册事件。`eh.registerTxEvent`API使应用可以收到事务结果通知（就是是否生效）。  
事务流程图示参考本文的[共识过程](https://github.com/wbwangk/wbwangk.github.io/wiki/Hyperledger#%E5%85%B1%E8%AF%86%E8%BF%87%E7%A8%8B)一节。  
关于事务流程的更多细节参考[Transaction Flow](http://hyperledger-fabric.readthedocs.io/en/latest/txflow.html)。  
开始链码编程参考[Chaincode for Developers](http://hyperledger-fabric.readthedocs.io/en/latest/chaincode4ade.html)。  
更多背书策略参考[Endorsement policies](http://hyperledger-fabric.readthedocs.io/en/latest/endorsement-policies.html)。  
更多fabric架构信息参考[Architecture Explained](http://hyperledger-fabric.readthedocs.io/en/latest/arch-deep-dive.html)。 
