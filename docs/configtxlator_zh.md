## 用configtxlator重新配置
[原文](http://hyperledger-fabric.readthedocs.io/en/latest/configtxlator.html)  
### 概述
创建`configtxlator`工具的目的是支持不依赖SDK的重新配置。通道配置被当作一个交易存储在通道的配置区块中，可以被直接操作，如在bdd行为测试中。不过，在撰写本文时，SDK本身不支持直接操作配置，因此该configtxlator工具旨在提供API，供任何SDK的使用者与之交互以协助配置更新。

工具名称是configtx和translator的混合，旨在表达该工具只是在两者之间进行转换。它不会生成配置。它不递交或检索配置。它不会自己修改配置，它只是在configtx格式的不同视图之间提供一些双向操作。

标准用法为：

1. SDK检索最新的配置  
2. `configtxlator` 产生人类可读的配置版本  
3. 用户或应用程序编辑配置  
4. `configtxlator`用于计算配置的变更  
5. SDK递交签名和递交配置  

`configtxlator`工具暴露了一个真正的无状态的REST API来与配置元素进行交互。这些REST组件支持将原生配置格式转换为人类可读的JSON格式，或者根据两种配置之间的差异来计算配置变更。

由于`configtxlator`服务故意不包含任何密钥材料或其他秘密信息，因此不包含任何授权或访问控制。预期的典型部署将在本地与应用程序一起作为沙盒容器来运行，因而`configtxlator`是个为消费者提供的专用进程。

### 运行configtxlator
`configtxlator`工具可以与其他Hyperledger Fabric平台相关的二进制文件一起下载。 详细信息查看download-platform-specific-binaries。

该工具可能被配置为侦听不同的端口，您也可以使用`--port`和`--hostname`标志来指定主机名。要查看完整的命令和标志，请运行`configtxlator --help`。

该二进制包将启动一个监听指定端口的http服务器，然后就可以处理请求了。

要启动`configtxlator`服务器：
```
$ configtxlator start
2017-06-21 18:16:58.248 HKT [configtxlator] startServer -> INFO 001 Serving HTTP requests on 0.0.0.0:7059
```
### Proto转换
为了扩展性，并且由于某些字段必须被签名，许多proto字段被存储为字节。这使得无法使用`jsonpb`包来完成原生proto到JSON的翻译。作为代替，`configtxlator`暴露了一个REST组件来做更复杂的翻译。

为了将proto转换为人类可读的JSON等价物，只需将二进制proto发布到REST地址`http://$SERVER:$PORT/protolator/decode/<message.Name>`，其中`<message.Name>`是消息的完全限定的proto名称。

例如，要解码一个配置区块并另存为`configuration_block.pb`，请运行以下命令：:
```
$ curl -X POST --data-binary @configuration_block.pb http://127.0.0.1:7059/protolator/decode/common.Block
```
为了转换人类可读JSON版本的prot消息，只需要把JSON消息发送到`http://$SERVER:$PORT/protolator/encode/<message.Name`，这里`<message.Name>`仍然是消息的完全限定的proto名称。  
例如，重编码区块另存为`configuration_block.json`，运行命令：
```
$ curl -X POST --data-binary @configuration_block.json http://127.0.0.1:7059/protolator/encode/common.Block
```
任何配置相关proto，包括`common.Block`、`common.Envelope`、`common.ConfigEnvelope`、`common.ConfigUpdateEnvelope`
、`common.Config`和`common.ConfigUpdate`都是这些URL有效目标。未来，可能会增加其它proto编码类型，如背书者交易。  

### 配置更新计算
给定两种不同的配置，可以计算在它们之间转换的配置更新。只需将两个`common.Config`proto编码的配置作为`multipart/formdata`、original作为字段`original`和updated作为字段`updated`，发送到`http://$SERVER:$PORT/configtxlator/compute/update-from-configs`。

例如，给定原始配置文件`original_config.pb`和更新的配置文件`updated_config.pb`，对于通道`desiredchannel`：
```
$ curl -X POST -F channel=desiredchannel -F original=@original_config.pb -F updated=@updated_config.pb http://127.0.0.1:7059/configtxlator/compute/update-from-configs
```
### 自举例子
启动`configtxlator`:
```
$ configtxlator start
2017-05-31 12:57:22.499 EDT [configtxlator] main -> INFO 001 Serving HTTP requests on port: 7059
```
首先，为排序系统通道生成一个创世区块：
```
$ configtxgen -outputBlock genesis_block.pb
2017-05-31 14:15:16.634 EDT [common/configtx/tool] main -> INFO 001 Loading configuration
2017-05-31 14:15:16.646 EDT [common/configtx/tool] doOutputBlock -> INFO 002 Generating genesis block
2017-05-31 14:15:16.646 EDT [common/configtx/tool] doOutputBlock -> INFO 003 Writing genesis block
```
将创世区块解码为人类可编辑格式：
```
$ curl -X POST --data-binary @genesis_block.pb http://127.0.0.1:7059/protolator/decode/common.Block > genesis_block.json
```
编辑生成的文件`genesis_block.json`，或者用程序操作它。这里我们使用JOSN CLI工具`jq`。为了简单，我们编辑通道的批大小，因为它是个简单的数字。然而，所有都可以编辑，包括策略和MSP。

首先，让我们建立一个环境变量来保存指向json中属性的路径：
```
$ export MAXBATCHSIZEPATH=".data.data[0].payload.data.config.channel_group.groups.Orderer.values.BatchSize.value.max_message_count"
```
然后，让我们显示这个属性的值：
```
$ jq "$MAXBATCHSIZEPATH" genesis_block.json
10
```
现在，我们设置新的批大小，并显示这个新值：
```
$ jq “$MAXBATCHSIZEPATH = 20” genesis_block.json > updated_genesis_block.json
$ jq “$MAXBATCHSIZEPATH” updated_genesis_block.json
20
```
创世区块现在准备好被重编码为用于自举的原生proto格式：
```
$ curl -X POST --data-binary @updated_genesis_block.json http://127.0.0.1:7059/protolator/encode/common.Block > updated_genesis_block.pb
```
`updated_genesis_block.pb`文件现在可以作为创世区块用于一个排序系统通道的自举。

### 重新配置的例子
*（先用命令删除所有docker容器`docker rm -f $(docker ps -aq)`，否则byfn.sh启动的排序服务也会绑定端口7050）*  
利用另外的终端窗口，使用默认配置启动排序服务，临时自举器会创建一个`testchainid`排序系统通道。
```
ORDERER_GENERAL_LOGLEVEL=debug orderer
```
重新配置一个通道的操作非常类似于改变一个创世配置。

*（现在configtxlator、orderer各占了一个终端窗口，下面需要启动第三个终端窗口）*  
首先，取得配置区块proto：
```
$ peer channel fetch config config_block.pb -o 127.0.0.1:7050 -c testchainid
-o 127.0.0.1:7050 -c testchainid
2017-12-11 09:00:15.658 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2017-12-11 09:00:15.660 UTC [main] main -> INFO 002 Exiting.....
```
*(当前目录下创建了文件`config_block.pb`)*  
然后，发送配置区块到`configtxlator`服务进行解码：
```
$ curl -X POST --data-binary @config_block.pb http://127.0.0.1:7059/protolator/decode/common.Block > config_block.json
```
从区块中提取配置节：
```
$ jq .data.data[0].payload.data.config config_block.json > config.json
```
编辑配置，把它另存为一个新的`updated_config.json`。这里，我们设置批大小为30。
```
$ jq ".channel_group.groups.Orderer.values.BatchSize.value.max_message_count = 30" config.json  > updated_config.json
```
对原始配置和变更的配置进行重编码到proto格式：
```
$ curl -X POST --data-binary @config.json http://127.0.0.1:7059/protolator/encode/common.Config > config.pb
$ curl -X POST --data-binary @updated_config.json http://127.0.0.1:7059/protolator/encode/common.Config > updated_config.pb
```
现在，两个配置都进行适当编码，发送它们到*configtxlator*服务，去计算它们之间的配置变更。
```
$ curl -X POST -F original=@config.pb -F updated=@updated_config.pb http://127.0.0.1:7059/configtxlator/compute/update-from-configs -F channel=testchainid > config_update.pb
```
这时，计算出来的配置变更已经准备好了。传统上，使用一个SDK签名和包装这个消息。然而，为了仅使用peer cli，*configtxlator*也可以用于完成这个任务。

首先，我们对配置变更(ConfigUPdate)进行解码，以便可以用文本方式操作它：
```
$ curl -X POST --data-binary @config_update.pb http://127.0.0.1:7059/protolator/decode/common.ConfigUpdate > config_update.json
```
然后，我们把它包裹进一个信封(envelope)消息：
```
$ echo '{"payload":{"header":{"channel_header":{"channel_id":"testchainid", "type":2}},"data":{"config_update":'$(cat config_update.json)'}}}' > config_update_as_envelope.json
```
接下来，将其转换回完整的配置交易的proto格式：
```
$ curl -X POST --data-binary @config_update_as_envelope.json http://127.0.0.1:7059/protolator/encode/common.Envelope > config_update_as_envelope.pb
```
最后，发送配置变更交易到排序服务去执行一个配置变更。
```
$ peer channel update -f config_update_as_envelope.pb -c testchainid -o 127.0.0.1:7050
```
### 增加一个组织
先启动`configtxlator`：
```
$ configtxlator start
2017-05-31 12:57:22.499 EDT [configtxlator] main -> INFO 001 Serving HTTP requests on port: 7059
```
使用`SampleDevModeSolo`profile选项启动排序服务。
```
$ ORDERER_GENERAL_LOGLEVEL=debug ORDERER_GENERAL_GENESISPROFILE=SampleDevModeSolo orderer
```
增加一个组织的过程与批大小的例子非常类似。然而，不同与设置批大小，一个新组织定义在应用级别。添加一个组织稍微牵扯一点，因为我们必须先创建一个通道，然后修改它的成员组成。  
关于利用configtxlator增加组织的更多内容参考[重新配置首个网络(First-Network)](https://github.com/wbwangk/wbwangk.github.io/wiki/Hyperledger#%E9%87%8D%E6%96%B0%E9%85%8D%E7%BD%AE%E9%A6%96%E4%B8%AA%E7%BD%91%E7%BB%9Cfirst-network)。
