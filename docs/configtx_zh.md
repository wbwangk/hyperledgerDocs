## 通道配置(configtx)
[原文](http://hyperledger-fabric.readthedocs.io/en/latest/configtx.html)  
Hyperledger Fabric区块链网络的共享配置被保存在一个配置交易集合中，每个通道一个。每个配置交易通常叫做*configtx*。  
通道配置有如下重要属性：
1. 版本(**Versioned**)：配置的所有元素都有一个关联的版本，每次修改都会更新。此外，每次更新配置都会收到一个顺序号。  
2. 权限(**Permissioned**)：配置的每个元素有一个关联的策略，用于管理是否允许对该元素进行修改。任何拥有以前configtx副本（并且没有其他信息）的人都可以根据这些策略验证新配置的有效性。
3. 层次(**Hierarchical**)：根配置组包含子组，每个层次组都有关联的值和策略。这些政策可以利用层次结构从下层策略中推导出一个层次的策略。
### 解剖一个配置
配置被保存在区块的一个类型为`HeaderType_CONFIG`的交易中，该区块中没有其他交易。这些区块被称为**配置区块**，第一个区块被称为**创世区块**(Genesis Block)。  
配置的proto结构存储在`fabric/protos/common/configtx.proto`。封装类型`HeaderType_CONFIG`编码了一个`ConfigEnvelope`消息，作为`Payload``data`字段。`ConfigEnvelope`的proto定义如下：
```golang
message ConfigEnvelope {
    Config config = 1;
    Envelope last_update = 2;
}
```
(上面的`1``2`是go语言结构的属性tag)  
`last_update`字段定义在下面的**Updates to configuration**部分，但仅在验证配置时需要，不会读它。相反，当前提交的配置被存储在`config`字段，其中包含了一个`Config`消息。
```golang
message Config {
    uint64 sequence = 1;
    ConfigGroup channel_group = 2;
}
```
`sequence`数字在每次配置提交后会增长。`channel_group`字段是包含配置的根组(root group)。。`ConfigGroup`结构是递归定义的，构建了一个组数，它可以包含值和策略。它是如下定义的：
```golang
message ConfigGroup {
    uint64 version = 1;
    map<string,ConfigGroup> groups = 2;
    map<string,ConfigValue> values = 3;
    map<string,ConfigPolicy> policies = 4;
    string mod_policy = 5;
}
```
因为`ConfigGroup`是个递归结构，它是按层次整理的。下面的例子是用go语言符号表示的。
```golang
// Assume the following groups are defined
var root, child1, child2, grandChild1, grandChild2, grandChild3 *ConfigGroup

// Set the following values
root.Groups["child1"] = child1
root.Groups["child2"] = child2
child1.Groups["grandChild1"] = grandChild1
child2.Groups["grandChild2"] = grandChild2
child2.Groups["grandChild3"] = grandChild3

// The resulting config structure of groups looks like:
// root:
//     child1:
//         grandChild1
//     child2:
//         grandChild2
//         grandChild3
```
每个组在配置层次上定义了一个级别，每个组都有一组关联的值（由字符串类型的key索引）和策略（也由字符串类型的key索引）。  

Value被下列结构定义：
```golang
message ConfigValue {
    uint64 version = 1;
    bytes value = 2;
    string mod_policy = 3;
}
```
Policy由下列结构定义：
```golang
message ConfigPolicy {
    uint64 version = 1;
    Policy policy = 2;
    string mod_policy = 3;
}
```
请注意，Value、Policy和Group都有一个`version`和一个`mod_policy`。元素的`version`的值会在元素被修改时递增。`mod_policy`用来管理修改元素所需的签名。

对于Group，修改就是向Value、Policy或Group映射中添加或删除元素（或更改`mod_policy`）。对于Value和Policy，修改就是分别改变Value和Policy字段（或改变`mod_policy`）。

每个元素的`mod_policy`都在当前配置级别的上下文中进行评估。考虑在下面例子的`mod_policy`定义在`Channel.Groups["Application"]`（在这里，我们使用golang映射引用语法，所以`Channel.Groups["Application"].Policies["policy1"]`引用了基础`Channel`组的`Application`组的`Policies`映射的`policy1`策略。）

- `policy1`映射到`Channel.Groups["Application"].Policies["policy1"]`  
- `Org1/policy2`映射到`Channel.Groups["Application"].Groups["Org1"].Policies["policy2"]`  
- `/Channel/policy3`映射到`Channel.Policies["policy3"]`  

请注意，如果`mod_policy`引用了一个不存在的策略，则该项目不会被修改。  

### 配置更新
配置更新会递交一个`HeaderType_CONFIG_UPDAT`类型的`Envelope`消息。交易的`Payload``data`是一个marshaled`ConfigUpdateEnvelope`。` ConfigUpdateEnvelope`的定义如下：
```golan
message ConfigUpdateEnvelope {
    bytes config_update = 1;
    repeated ConfigSignature signatures = 2;
}
```
签名字段包含了一组授权配置更新的签名。它的消息定义是：
```golang
message ConfigSignature {
    bytes signature_header = 1;
    bytes signature = 2;
}
```
`signature_header`是标准交易的定义，而`signature`是`signature_header`字节和`ConfigUpdateEnvelope`消息的`config_update`字节串连后的签名。

`ConfigUpdateEnvelope`的`config_update`字节是一个marshaled`ConfigUpdate`消息，该消息定义如下：
```golang
message ConfigUpdate {
    string channel_id = 1;
    ConfigGroup read_set = 2;
    ConfigGroup write_set = 3;
}
```
这`channel_id`是更新要绑定的通道ID，这对于限定此重新配置的签名范围是必要的。

在`read_set`指定了现有的配置的一个子集，它暗示了只有`version`字段必须提供的，而且字段是可选则。字段`ConfigValue``value`或 `ConfigPolicy``policy`不允许出现在`read_set`。`ConfigGroup`可以填充它映射字段的子集，以便引用在配置树更深的元素。例如，要`read_set`中包括`Application`组，则它的母元素（该`Channel`组）也必须包含在`read_set`中，但是，该`Channel`组并不需要填充所有的key，如`Orderer``group`key，或任何`values`或`policies`key。

`write_set`定义了配置的被修改部分。由于配置的层次性，对层次结构中深层元素的写入操作也必须在`write_set`从包含其更高层次的元素。但是，对于`write_set`中任何元素，如果也出现在`read_set`中且版本相同，则会被更新忽略（？）。

例如，对于给定配置：
```
Channel: (version 0)
    Orderer (version 0)
    Appplication (version 3)
       Org1 (version 2)
```
去递交一个改变`Org1`的配置更新，`read_set`会是：
```
Channel: (version 0)
    Application: (version 3)
```
而它的`write_set`会是：
```
Channel: (version 0)
    Application: (version 3)
        Org1 (version 3)
```
当收到`CONFIG_UPDATE`，orderer会按下面的步骤计算结果集`CONFIG`：
1. 检验`channel_id`和`read_set`。所有`read_set`中元素必须存在且版本号相同。  
2. 通过比较存在于`write_set`中而不存在于`read_set`中且版本号相同的元素，得到更新元素集。  
3. 检验更新元素集中的元素版本号，确保只增长了1。  
4. 对更新元素集中的每个元素，检验针对`ConfigUpdateEnvelope`的签名集，确保符合`mod_policy`。  
5. 通过将更新集应用到当前配置，计算出配置的一个新完全版本。  
6. 将新配置写入`ConfigEnvelope`，将`CONFIG_UPDATE`作为`the last_update`字段，并与增长的`sequence`值一起将新配置编码入`config`字段。  
7. 将一个新的`ConfigEnvelope`以类型`CONFIG`写入`Envelope`，最后将这个作为唯一交易写入一个新配置区块。  

当新的peer(或任何其他的`Deliver`接收者)接收到这个配置区块，它会校验这个配置，方法是将`last_update`消息应用到当前配置，确保orderer计算出来的`config`字段包含了正确的新配置。  

### 允许的配置组(group)和值(value)
任何有效的配置都是下面配置的子集。这里我们使用符号`peer.<MSG>`来定义一个`ConfigValue`，它的`value`字段是一个名为`<MSG>`的marshaled proto消息（定义在`fabric/protos/peer/configuration.proto`）。符号`common.<MSG>`、`msp.<MSG>`和`orderer.<MSG>`含义类似，只是它们的消息分别定义在`fabric/protos/common/configuration.proto`、`fabric/protos/msp/mspconfig.proto`和`fabric/protos/orderer/configuration.proto`。  

需要注意的是key`{{org_name}}`和`{{consortium_name}}`可以是任意名字，这表明一个元素可以以任何名字重复出现。  
```golang
&ConfigGroup{
    Groups: map<string, *ConfigGroup> {
        "Application":&ConfigGroup{
            Groups:map<String, *ConfigGroup> {
                {{org_name}}:&ConfigGroup{
                    Values:map<string, *ConfigValue>{
                        "MSP":msp.MSPConfig,
                        "AnchorPeers":peer.AnchorPeers,
                    },
                },
            },
        },
        "Orderer":&ConfigGroup{
            Groups:map<String, *ConfigGroup> {
                {{org_name}}:&ConfigGroup{
                    Values:map<string, *ConfigValue>{
                        "MSP":msp.MSPConfig,
                    },
                },
            },

            Values:map<string, *ConfigValue> {
                "ConsensusType":orderer.ConsensusType,
                "BatchSize":orderer.BatchSize,
                "BatchTimeout":orderer.BatchTimeout,
                "KafkaBrokers":orderer.KafkaBrokers,
            },
        },
        "Consortiums":&ConfigGroup{
            Groups:map<String, *ConfigGroup> {
                {{consortium_name}}:&ConfigGroup{
                    Groups:map<string, *ConfigGroup> {
                        {{org_name}}:&ConfigGroup{
                            Values:map<string, *ConfigValue>{
                                "MSP":msp.MSPConfig,
                            },
                        },
                    },
                    Values:map<string, *ConfigValue> {
                        "ChannelCreationPolicy":common.Policy,
                    }
                },
            },
        },
    },

    Values: map<string, *ConfigValue> {
        "HashingAlgorithm":common.HashingAlgorithm,
        "BlockHashingDataStructure":common.BlockDataHashingStructure,
        "Consortium":common.Consortium,
        "OrdererAddresses":common.OrdererAddresses,
    },
}
```

### 排序系统通道配置
排序系统通道需要定义排序参数，和创建通道的联盟。对一个排序服务必须存在一个排序系统通道，它是创建的第一个通道（更准确地说是引导时创建）。建议不要在排序系统通道的创世配置中定义应用部分（的通道），但可以在测试时这样做。注意，对排序系统通道具有读权限的成员可以看到所有通道（系统的和应用的所有通道）的创建，所以这个通道的访问权限需要严格控制。

排序参数用下面配置的子集来定义：
```golang
&ConfigGroup{
    Groups: map<string, *ConfigGroup> {
        "Orderer":&ConfigGroup{
            Groups:map<String, *ConfigGroup> {
                {{org_name}}:&ConfigGroup{
                    Values:map<string, *ConfigValue>{
                        "MSP":msp.MSPConfig,
                    },
                },
            },

            Values:map<string, *ConfigValue> {
                "ConsensusType":orderer.ConsensusType,
                "BatchSize":orderer.BatchSize,
                "BatchTimeout":orderer.BatchTimeout,
                "KafkaBrokers":orderer.KafkaBrokers,
            },
        },
    },
```
参与排序的每个组织在`Orderer`组下都有一个组元素。这个组定义一个`MSP`参数，它包含了组织的密钥id信息。`Orderer`组的`Values`确定了排序节点的功能。它们存在于每个通道，所以实例的`orderer.BatchTimeout`可以在不同的通道定义不同的值。

启动时，orderer会面对一个包含了多个通道信息的文件系统。orderer根据联盟组通道定义出系统通道。联盟组具有下面的结构：
```golang
&ConfigGroup{
    Groups: map<string, *ConfigGroup> {
        "Consortiums":&ConfigGroup{
            Groups:map<String, *ConfigGroup> {
                {{consortium_name}}:&ConfigGroup{
                    Groups:map<string, *ConfigGroup> {
                        {{org_name}}:&ConfigGroup{
                            Values:map<string, *ConfigValue>{
                                "MSP":msp.MSPConfig,
                            },
                        },
                    },
                    Values:map<string, *ConfigValue> {
                        "ChannelCreationPolicy":common.Policy,
                    }
                },
            },
        },
    },
},
```
注意，每个联盟定义了一组成员，就像排序组织成员一样。每个联盟还定义了一个ChannelCreationPolicy。这个策略用于对通道创建请求进行授权。通常，这个值被设为`ImplicitMetaPolicy`，意思是要求通道的新成员签名以授权频道创建(个人理解：打算创建一个包含N个组织的通道，那么这N个组织都得签名，每个组织都有否决权，有组织不同意，新通道就建不起来)。关于通道创建的更多内容在文章的后面还有。

### 应用通道配置
通道的应用配置被设计用于应用类型的交易。它的定义如下：
```golang
&ConfigGroup{
    Groups: map<string, *ConfigGroup> {
        "Application":&ConfigGroup{
            Groups:map<String, *ConfigGroup> {
                {{org_name}}:&ConfigGroup{
                    Values:map<string, *ConfigValue>{
                        "MSP":msp.MSPConfig,
                        "AnchorPeers":peer.AnchorPeers,
                    },
                },
            },
        },
    },
}
```
很像`Orderer`部分，每个组织被编码为一个组(group)。然而，不仅仅编码了`MSP`id信息，每个组织附加编码了一个`AnchorPeers`列表。这是一个允许不同组织之间通过peer gossip网络互相联系的peer列表（这个列表中的peer才可以被其他组织的peer“看到”）。

应用通道编码了一个orderer组织的副本，和用于确定变更这些参数的共识选项，所以包含了系统通道配置的`Orderer`部分的相同内容。然而，从应用的观点这可以被大大忽略。  

### 通道创建
当排序节点收到一个不存在的通道的`CONFIG_UPDATE`，排序节点就假定这是个通道创建请求，然后执行下列操作：
1. orderer确定发出通道创建请求的联盟身份。它通过查看顶层group的`Consortium`值来做到这一点。  
2. orderer验证包含在`Application`组的组织，确保这些组织是联盟下属(子集)，而且`ApplicationGroup`被设置为`version``1`。  
3. orderer验证联盟是否有成员，从而新通道会有应用成员(创建没有成员的联盟和通道仅用于测试)。  
4. orderer通过从排序系统通道中取得`Orderer`组来创建一个模板配置，用新的成员创建一个`Application`组并指定`mod_policy`为联盟配置里的`ChannelCreationPolicy`。需要注意的是策略是在新配置的上下文中被评估的，所以一个需要`ALL`成员的策略仅需要全部新通道成员的签名，而不是联盟的所有成员签名。  
5. orderer将`CONFIG_UPDATE`作为一个更新应用到这个模板配置。因为这个`CONFIG_UPDATE`应用变更到`Application`组(它的`version`是`1`)，配置代码用`ChannelCreationPolicy`验证这些更新。如果通道创建包含其他变更，如个别组织的锚peer，元素的相应mod policy会被调用。  
6. 为了排序，含有新通道配置的新`CONFIG`交易会被包装和发送到排序系统通道。排序后，通道就创建了。  
