## 背书策略
[原文](http://hyperledger-fabric.readthedocs.io/en/latest/endorsement-policies.html)  
背书策略是用来指导peer如何决定交易是否得到适当的赞同。当peer收到一个交易时，作为交易验证流程的一部分，它调用与交易的链码相关联的VSCC(验证系统链码)来确定交易的有效性。回忆一下，一个交易包含来自任何背书peer的一个或多个背书。VSCC的任务是做出以下决定：
- 所有的背书是有效的（也就是它们是来自有效证书的有效签名）  
- 有恰当数量的签名  
- 背书来自预期的源  

背书策略是指定第二和第三点的一种方式。  

### CLI中的背书策略语法
(本节中的身份都是principal)
在CLI中，策略用一种简单的语言来表达，即基于身份的布尔表达式。  
一个身份通过MSP描述，它的任务是验证签名者身份和签名者角色存在于MSP中。当前，支持两种角色，**member**(成员)和**admin**(管理员)。身份用`MSP`.`ROLE`描述，这里`MSP`是MSP ID(必填)，`ROLE`是`member`或`admin`二选一。验证身份(principal)的例子是`Org0.admin`(`Org0`MSP的任何管理员)或`Org1.member`(`Org1`MSP的任何成员)。  
这个语言的语法是：  
```
EXPR(E[, E...])
```
这里`EXPR`是`AND`或`OR`，表示两个布尔表达式。`E`可以是一个身份或另一个嵌入的`EXPR`。（运算符在前面，这被称为波兰语表示法）   

例如：  
- `AND('Org1.member', 'Org2.member', 'Org3.member')` 要求三位身份分别签名  
- `OR('Org1.member', 'Org2.member')` 两身份中任何一个签名都可以  
- `OR('Org1.member', AND('Org2.member', 'Org3.member'))` 要求`Org1` MSP成员的一个签名，或`Org2`MSP成员和`Org3`MSP成员的分别签名  
### 定义链码的背书策略
使用这个语言，链码部署者可以为某个链码指定特定策略。注意，默认策略是需要`DEFAULT`MSP成员的一个签名。用CLI实例化链码时，没有指定策略就使用默认策略。  
实例化时用`-P`选项指定策略。例如：
```
$ peer chaincode instantiate -C <channelid> -n mycc -P "AND('Org1.member', 'Org2.member')"
```
这个命令部署链码`mycc`，使用的策略是`AND('Org1.member', 'Org2.member')`，即同时需要Org1和Org2的成员签署交易。 
