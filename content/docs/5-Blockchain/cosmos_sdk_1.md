---
title: "解读cosmos-sdk系列(1)"
linkTitle: "解读cosmos-sdk系列(1)"
tags: ["区块链"]
weight: 6
date: 2017-01-05
description: >
  分析解读cosmos跨链方案
---

通过本系列，可以了解tendermint共识和cosmos-sdk架构的设计思想，并学习到如何通过Cosmos-SDK来快速开发自己的区块链应用。

cosmos团队把区块链分成了三层

* 网络层 - p2p负责广播交易
* 共识层 - 对哪些交易打包进块形成共识
* 应用层 - 执行交易，负责交易结果落盘（状态一致）

>这里的应用层可能会有误解，并非是Dapp层，对于SDK底层的Tendermint来说，除p2p网络和打包块共识外，其他都算是应用部分，
拿实现比特币公链的例子来讲，应用部分就是维护账户的UTXO数据库，如果对比以太的话，keystore账户和EVM虚机部分就是应用范畴，所以SDK内置了账户、质押、治理、权限等应用模块，可以帮助我们简单地实现底层链的开发。

可以把这几层简单理解成各节点通过同步交易集（块）日志，实现数据（状态）一致性。数据库的主从模式不也是同步binlog日志，各自执行（replay，回放）日志后，实现数据（状态）最终落盘，区块节点本身同步块的时候，默认就是去下载交易日志，把执行结果按照逻辑链的形式写入本地leveldb的，然后才能对外提供各类RPC服务。

## tendermint共识

为后续更好的利用cosmos-sdk，要先了解下Tendermint。

Tendermint Core 提供了网络和共识层功能，而应用层要通过ABCI协议和Core互通消息msg，简单讲tendermint负责起一个replication engine进程，而应用层要运行一个state macheine进程，进程间通过ABCI消息来通信。

ABCI协议的消息体用protobuf定义在[这里](https://github.com/tendermint/tendermint/blob/master/abci/types/types.proto)，app侧可以响应的request如下：

```golang
message Request {
  oneof value {
    RequestEcho echo = 2;
    RequestFlush flush = 3;
    RequestInfo info = 4;
    RequestSetOption set_option = 5;
    RequestInitChain init_chain = 6;
    RequestQuery query = 7;
    RequestBeginBlock begin_block = 8;
    RequestCheckTx check_tx = 9;
    RequestDeliverTx deliver_tx = 19;
    RequestEndBlock end_block = 11;
    RequestCommit commit = 12;
  }
}
```

ABCI的设计主要有以下几个特点：

* 消息协议
    * 成对出现的消息: `request`/`reponse`
    * tendermint发起Request， app来响应
    * 使用protobuf定义

* server/client
    * tendermint运行client
    * app侧运行server端
    * 可以由TSP（支持checkTx和DeliverTx消息的异步处理）、grpc两种方式实现

* 区块相关
    * abci是面向连接的
    * tendermint会创建三个socket连接来和app通信，分别是`Mempool`, `Cosensus`, `Query`连接
        * `Mempool连接`: 钱包客户端发起交易，会首先进入钱包后台连接的节点的local mempool，该节点通过发送`checkTx`消息来通知app，去检验交易签名是否有效等等，如果OK节点则会p2p广播该交易到其他节点的mempool里。

        * `Cosensus连接`: 出块节点从mempool的交易集里选出一个块提案（proposer），之后会经过3阶段提交（pre-vote, pre-commit， commit）处理，这个块才能说达成共识（上链了），，只有块被commited了，app侧才会更新状态，比如改变某地址余额等等。app更新状态的时候，是通过Core发送`BeginBlock`， `DeliverTx ...`， `EndBlock`, `Commit`消息给app侧来完成的，任何写入操作都是通过此连接完成的。

        * `Query连接`： 主要负责一些和共识无关的查询操作，比如块信息，地址余额等等，主要用到`Info`, `SetOption`, `Query`消息

整个abci的信息流大致如下：
![abci](/abci.png)

关于Tendermint和app间数据流的更多细节见下：

![dataFlow](/tm-transaction-flow.png)

[高清图地址](https://github.com/mobfoundry/hackatom/blob/master/tminfo.pdf)


## 总结

可以理解成，Tendermint主要负责在BFT环境下同步app间的块交易日志，无论任何交易类型，只要交易块的执行结果是确定性的（唯一性），Tendermint就可以为我们形成区块的共识。

    比如说，一个交易的内容在合约里创建一个真随机数，这种交易，tendermint是无法为我们形成共识的，因为多个节点的执行结果是不一样的， 因为这个结果是要在下个区块头的，这样就无法对下个区块形成共识了，所有节点都认为对方在恶意“搞分叉”了。

目前基于tendermint的项目有很多：

我个人看到好的是`BigchainDB`和超级账本`Burrow`项目，更多可以看[这里](https://tendermint.com/ecosystem)


后续源码介绍，如何基于tendermint创建一个区块链