
[原文](http://hyperledger-fabric.readthedocs.io/en/latest/write_first_app.html)  

## 编写首个应用
在实践本章前，请按[启动首个网络(first-network)](https://github.com/wbwangk/wbwangk.github.io/wiki/Hyperledger#%E5%90%AF%E5%8A%A8%E9%A6%96%E4%B8%AA%E7%BD%91%E7%BB%9Cfirst-network)一章的描述准备好环境。  
本章原文是官网的[Writing Your First Application](http://hyperledger-fabric.readthedocs.io/en/latest/write_first_app.html)  
本章的工作目录是`/opt/fabric-samples/fabcar`。  
在本章中，首先访问CA生成注册证书(ECert)，然后利用生成的身份(用户对象)查询和更新账本。  
### 建立开发环境
可以通过执行下列命令停止“first-network”一章中的各种容器：
```
$ cd /opt/fabric-samples/fabcar
$ ../first-network/byfn.sh -m down
```
或使用docker命令删除所有容器(可以与上面的命令混合使用)：
```
$ docker rm -f $(docker ps -aq)
```
删除所有缓存网络：
```
$ docker network prune
```
#### 安装客户端和启动网络
`/opt/fabric-samples/fabcar`目录下有个`package.json`文件，定义了示范应用的依赖包，通过执行下列命令安装依赖（与java的maven类似）：
```
$ npm install
```
使用脚本`startFabric.sh`启动网络。这个命令会启动几个Fabric实体和一个智能合约容器(链码)。
```
$ ./startFabric.sh
$ docker ps
```
用上述docker命令可以看到启动的容器清单。  

### 注册用户
#### 注册管理员用户
前面的`startFabric.sh`命令会启动一个CA容器，可以通过下列命令查看CA容器的日志：
```
$ docker logs -f ca.example.com
```
执行下列命令来注册admin用户：
```
$ node enrollAdmin.js
```
上述代码会向创建目录`hfc-key-store`，创建私钥，向CA发送证书签名请求(CSR)，然后把返回的CA签名证书(eCert)存放在`hfc-key-store`目录。  
#### 注册用户user1
admin用户是运维人员用的管理员证书。在一般的业务场景下，应用程序使用“普通用户”(非管理员)来访问Fabric网络。需要说明的是，向Fabric网络注册普通用户需要使用管理员身份，如现在就是用admin身份来注册`user1`用户。  
```
$ node registerUser.js
```
与注册admin用户类似，代码会创建user1用户私钥，向CA发送证书签名请求(CSR)，并将返回的CA签名证书(eCert)存放在`hfc-key-store`目录。  
### 查询账本
用`docker ps`命令可以看到一个`hyperledger/fabric-couchdb`的容器，这是一个保存“世界状态”的key-value数据库(couchdb)。查询账本实际上就是在查询key-value数据库中的数据（数据库变化日志保存在区块链中）。查询参数一般是一个或几个key，也可以用json串当参数进行复杂查询。  
演示查询的代码文件是`query.js`，其中可以看到下面的代码，说明应用使用`user1`作为签名实体。
```
fabric_client.getUserContext('user1', true);
```
user1的身份证明材料已经放在了`hfc-key-store`目录下，我们简单地告诉应用去获取身份。  
```
$ node query.js
```
会返回一个json串，里面是全部10辆车的信息。  
观察一下`query.js`源码。在应用的初始化区段，定义了几个变量，如通道名称、证书库地址和网络端点。
```js
var channel = fabric_client.newChannel('mychannel');
var peer = fabric_client.newPeer('grpc://localhost:7051');
channel.addPeer(peer);

var member_user = null;
var store_path = path.join(__dirname, 'hfc-key-store');
console.log('Store path:'+store_path);
var tx_id = null;
```
下面是执行查询的代码：
```js
// queryCar chaincode function - requires 1 argument, ex: args: ['CAR4'],
// queryAllCars chaincode function - requires no arguments , ex: args: [''],
const request = {
  //targets : --- letting this default to the peers assigned to the channel
  chaincodeId: 'fabcar',
  fcn: 'queryAllCars',
  args: ['']
};
```
应用执行时，它调用了peer的`fabcar`链码，执行其中的`queryAllCars`函数。 
如果想看看有哪些链码函数可以调用，可以打开文件`../chaincode/fabcar/go/fabcar.go`来看。可以看到下列函数可以调用：`initLedger`, `queryCar`, `queryAllCars`, `createCar`和`changeCarOwner`。  
下图图示了应用、智能合约和账本的关系：  
![](http://hyperledger-fabric.readthedocs.io/en/latest/_images/RunningtheSample.png)  
下面示范一下调用链码的`queryCar`函数来查询某一辆车的信息。将`query.js`中的`queryAllCars`函数换成`quieryCar`，参数key是`CAR4`。
```js
const request = {
  //targets : --- letting this default to the peers assigned to the channel
  chaincodeId: 'fabcar',
  fcn: 'queryCar',
  args: ['CAR4']
};
```
保存`query.js`，并执行，可以看到返回了`CAR4`的车辆信息：
```
$ node query.js
{"colour":"black","make":"Tesla","model":"S","owner":"Adriana"}
```
#### 更新账本
更新账本的过程：先提议，后背书，结果返回到应用，然后发送给排序节点，再写到各个peer的账本。  
![](http://hyperledger-fabric.readthedocs.io/en/latest/_images/UpdatingtheLedger.png)  
js文件`invoke.js`是更新账本的程序示范。首先可以看到程序中构造请求的代码：
```js
var request = {
  //targets: let default to the peer assigned to the client
  chaincodeId: 'fabcar',
  fcn: 'createCar',
  args: ['CAR10', 'Chevy', 'Volt', 'Red', 'Nick'],
  chainId: 'mychannel',
  txId: tx_id
};
```
在代码中通过调用链码`fabcar`的`createCar`函数，试图创建一个10号车(key是`CAR10`)。执行`invoke.js`：
```
$ node invoke.js
Store path:/opt/fabric-samples/fabcar/hfc-key-store
Successfully loaded user1 from persistence
Assigning transaction_id:  1adb925d24db20816bcfc97f0216f3b094f3291778af775c1d07d0d3179a3031
Transaction proposal was good
Successfully sent Proposal and received ProposalResponse: Status - 200, message - "OK"
info: [EventHub.js]: _connect - options {"grpc.max_receive_message_length":-1,"grpc.max_send_message_length":-1}
The transaction has been committed on peer localhost:7053
Send transaction promise and event listener promise have completed
Successfully sent transaction to the orderer.
Successfully committed the change to the ledger by the peer
```
从上面可以看到关于`ProposalResponse`的终端输出和promise(一个异步调用标准)。上文的`The transaction has been committed on peer localhost:7053`表示事务已经成功写到peer。下面可以回到`query.js`，将查询条件由`CAR4`改成`CAR10`：
```js
const request = {
  //targets : --- letting this default to the peers assigned to the channel
  chaincodeId: 'fabcar',
  fcn: 'queryCar',
  args: ['CAR10']
};
```
重新执行`query.js`可以查询到最新添加的10号车的信息：
```
$ node query.js
Response is  {"colour":"Red","make":"Chevy","model":"Volt","owner":"Nick"}
```
下面重新修改代码`invoke.js`。将函数`createCar`改成`changeCarOwner`，目的是把10号车的车主改成`Dave`：
```js
var request = {
  //targets: let default to the peer assigned to the client
  chaincodeId: 'fabcar',
  fcn: 'changeCarOwner',
  args: ['CAR10', 'Dave'],
  chainId: 'mychannel',
  txId: tx_id
};
```
重新执行`invoke.js`和`query.js`，可以看到车主信息被从`Nick`改成了`Dave`。
```
$ node invoke.js
$ node query.js
Response is  {"colour":"Red","make":"Chevy","model":"Volt","owner":"Dave"}
```
本章主要讲应用开发，后面的章节会讲链码的开发。  
