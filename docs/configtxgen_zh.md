## 通道配置(configtxgen)
本文档介绍了`configtxgen`的用法，它是操作Hyperledger Fabric通道配置的实用程序。

目前，该工具主要侧重于生成用于引导orderer的创世区块，但是将来要增强生成新的通道配置以及重新配置现有通道。  

### 配置profile
为`configtxgen`工具提供配置参数的主要供给是文件`configtx.yaml`。这个位于fabric.git库的`fabric/sampleconfig/configtx.yaml`位置。

这个配置文件主要分为三部分：

1. `Profiles`部分。默认情况下，这个部分包含一些可以用于开发和测试场景示范配置，使用了fabric.git树中的密钥材料。这些profile对于组织一个实际的部署profile是个很好的起点。`configtxgen`工具允许你用`-profile`标志来指定profile。profile可以明确地表明所有配置，但通常从第3部分(默认部分)继承配置。  
2. `Organizations`部分。默认情况下，这部分包含指向sampleconfig MSP定义的一个简单引用。对于生产部署，示范组织会被移除，网络成员的MSP定义会被引用和定义。`Organizations`部分的每个元素都可以打上一个锚点标签如`&orgName`，这允许这个定义被`Profiles`部分引用。  
3. 默认部分。这里有`Orderer`和`Application`部分的默认配置，包括象`BatchTimeout`这样的属性和通常用作profile的基本继承值的属性。  

这个配置文件可以被编辑，或者单个属性可能被设置环境变量覆盖，比如`CONFIGTX_ORDERER_ORDERERTYPE=kafka`。请注意，`Profiles` 元素和profile名称不需要指定。

### 引导orderer
创建一个期望的配置profile，简单调用：
```
$ configtxgen -profile <profile_name> -outputBlock orderer_genesisblock.pb
```
将在当面目录下创建一个`orderer_genesisblock.pb`文件。这个创世区块被用于引导排序系统通道，该通道被orderer用于授权和编排其他通道的创建。默认情况下，由`configtxgen`生成，编码进创世区块的通道ID是`testchainid`。建议你修改这个值为某个全局唯一的值。  

为了利用生成的创世区块，在启动orderer之前，需要设定环境变量：
```
ORDERER_GENERAL_GENESISMETHOD=file
ORDERER_GENERAL_GENESISFILE=$PWD/orderer_genesisblock.pb
```
或者修改配置文件orderer.yaml，将上述值包含进文件中。  

### 创建一个通道
这个工具可以通过执行下列命令输出一个通道创建tx：
```
$ configtxgen -profile <profile_name> -channelID <channel_name> -outputCreateChannelTx <tx_filename>
```
这会输出一个可以广播出去创建通道的marshaled `Envelope`消息。

### 显示一个配置
除了生成配置以外，`configtxgen`工具还有查看配置的能力。  
它支持查看配置区块和配置交易。可以分别使用查看标志`-inspectBlock`和`-inspectChannelCreateTx`并在后面附加文件路径来输出一个JSON串来显示配置信息。  
还可以对查看标志进行组合，例如：
```json
$ configtxgen -channelID foo -outputBlock foo_genesisblock.pb -inspectBlock foo_genesisblock.pb
2017-11-02 17:56:04.489 EDT [common/tools/configtxgen] main -> INFO 001 Loading configuration
2017-11-02 17:56:04.564 EDT [common/tools/configtxgen] doOutputBlock -> INFO 002 Generating genesis block
2017-11-02 17:56:04.564 EDT [common/tools/configtxgen] doOutputBlock -> INFO 003 Writing genesis block
2017-11-02 17:56:04.564 EDT [common/tools/configtxgen] doInspectBlock -> INFO 004 Inspecting block
2017-11-02 17:56:04.564 EDT [common/tools/configtxgen] doInspectBlock -> INFO 005 Parsing genesis block
{
  "data": {
    "data": [
      {
        "payload": {
          "data": {
            "config": {
              "channel_group": {
                "groups": {
                  "Consortiums": {
                    "groups": {
                      "SampleConsortium": {
                        "mod_policy": "/Channel/Orderer/Admins",
                        "values": {
                          "ChannelCreationPolicy": {
                            "mod_policy": "/Channel/Orderer/Admins",
                            "value": {
                              "type": 3,
                              "value": {
                                "rule": "ANY",
                                "sub_policy": "Admins"
                              }
                            },
                            "version": "0"
                          }
                        },
                        "version": "0"
                      }
                    },
                    "mod_policy": "/Channel/Orderer/Admins",
                    "policies": {
                      "Admins": {
                        "mod_policy": "/Channel/Orderer/Admins",
                        "policy": {
                          "type": 1,
                          "value": {
                            "rule": {
                              "n_out_of": {
                                "n": 0
                              }
                            },
                            "version": 0
                          }
                        },
                        "version": "0"
                      }
                    },
                    "version": "0"
                  },
                  "Orderer": {
                    "mod_policy": "Admins",
                    "policies": {
                      "Admins": {
                        "mod_policy": "Admins",
                        "policy": {
                          "type": 3,
                          "value": {
                            "rule": "MAJORITY",
                            "sub_policy": "Admins"
                          }
                        },
                        "version": "0"
                      },
                      "BlockValidation": {
                        "mod_policy": "Admins",
                        "policy": {
                          "type": 3,
                          "value": {
                            "rule": "ANY",
                            "sub_policy": "Writers"
                          }
                        },
                        "version": "0"
                      },
                      "Readers": {
                        "mod_policy": "Admins",
                        "policy": {
                          "type": 3,
                          "value": {
                            "rule": "ANY",
                            "sub_policy": "Readers"
                          }
                        },
                        "version": "0"
                      },
                      "Writers": {
                        "mod_policy": "Admins",
                        "policy": {
                          "type": 3,
                          "value": {
                            "rule": "ANY",
                            "sub_policy": "Writers"
                          }
                        },
                        "version": "0"
                      }
                    },
                    "values": {
                      "BatchSize": {
                        "mod_policy": "Admins",
                        "value": {
                          "absolute_max_bytes": 10485760,
                          "max_message_count": 10,
                          "preferred_max_bytes": 524288
                        },
                        "version": "0"
                      },
                      "BatchTimeout": {
                        "mod_policy": "Admins",
                        "value": {
                          "timeout": "2s"
                        },
                        "version": "0"
                      },
                      "ChannelRestrictions": {
                        "mod_policy": "Admins",
                        "version": "0"
                      },
                      "ConsensusType": {
                        "mod_policy": "Admins",
                        "value": {
                          "type": "solo"
                        },
                        "version": "0"
                      }
                    },
                    "version": "0"
                  }
                },
                "mod_policy": "Admins",
                "policies": {
                  "Admins": {
                    "mod_policy": "Admins",
                    "policy": {
                      "type": 3,
                      "value": {
                        "rule": "MAJORITY",
                        "sub_policy": "Admins"
                      }
                    },
                    "version": "0"
                  },
                  "Readers": {
                    "mod_policy": "Admins",
                    "policy": {
                      "type": 3,
                      "value": {
                        "rule": "ANY",
                        "sub_policy": "Readers"
                      }
                    },
                    "version": "0"
                  },
                  "Writers": {
                    "mod_policy": "Admins",
                    "policy": {
                      "type": 3,
                      "value": {
                        "rule": "ANY",
                        "sub_policy": "Writers"
                      }
                    },
                    "version": "0"
                  }
                },
                "values": {
                  "BlockDataHashingStructure": {
                    "mod_policy": "Admins",
                    "value": {
                      "width": 4294967295
                    },
                    "version": "0"
                  },
                  "HashingAlgorithm": {
                    "mod_policy": "Admins",
                    "value": {
                      "name": "SHA256"
                    },
                    "version": "0"
                  },
                  "OrdererAddresses": {
                    "mod_policy": "/Channel/Orderer/Admins",
                    "value": {
                      "addresses": [
                        "127.0.0.1:7050"
                      ]
                    },
                    "version": "0"
                  }
                },
                "version": "0"
              },
              "sequence": "0",
              "type": 0
            }
          },
          "header": {
            "channel_header": {
              "channel_id": "foo",
              "epoch": "0",
              "timestamp": "2017-11-02T21:56:04.000Z",
              "tx_id": "6acfe1257c23a4f844cc299cbf53acc7bf8fa8bcf8aae8d049193098fe982eab",
              "type": 1,
              "version": 1
            },
            "signature_header": {
              "nonce": "eZOKru6jmeiWykBtSDwnkGjyQt69GwuS"
            }
          }
        }
      }
    ]
  },
  "header": {
    "data_hash": "/86I/7NScbH/bHcDcYG0/9qTmVPWVoVVfSN8NKMARKI=",
    "number": "0"
  },
  "metadata": {
    "metadata": [
      "",
      "",
      "",
      ""
    ]
  }
}
```
上述命令先生成区块，再显示它。  
