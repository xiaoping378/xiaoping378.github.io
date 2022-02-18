---
title: "解读Consul"
linkTitle: "解读Consul"
weight: 10
description: >
  解读Consul和背后的设计原理。 
---

## 概述

介绍现在的Consul前，有必要先介绍下`Service Mesh`的概念，

**Service Mesh**

Service Mesh是一种面向服务间安全通信的基础设施层，俗称代理服务的东西向流量。 典型的实现包含控制面板和数据面。

- 控制面：跟踪记录所有的服务和访问地址，提供服务发现、健康检查、策略管理等。
- 数据面：通过sidcar机制代理服务间的通信。

_传统 vs 微服务优势 :_
- 单体 -> 微服务架构
- 函数调用 -> 服务发现（熔断、健康检查）
- 配置文件 -> 配置中心
- 防火墙 -> 访问策略
- DMZ、生产区 -> 零信任 

**Consul**

Consul是Service mesh中的控制面的实现，主要有一下几大特色：

- 服务发现：分为注册服务、发现服务、健康检查三大功能。
- K-V存储：[BoltDB](https://github.com/etcd-io/bbolt)，支持 ACID 事务、无锁并发事务 MVCC，提供 B+Tree 索引。
- 多数据中心：多中心Gossip数据同步
- Servie Mesh: 利用sidecar机制实现了端到端的认证和加密通信

## 部署架构

下图为经典的多中心consul集群配置。

![](/images/解读consu-01-2022-02-16-15-57-53.png)

- 每个datacenter，都有N个Server和M个Client节点。因为网络延迟对Consul分布式一致性的性能影响，会出现节点越多，共识越慢的现象，server节点一般推荐为3~5个，但client节点没有限制。
- Client主要充当DNS server、负载均衡和健康检查的作用。
- Server实现数据的分布式存储一致性，一般用来存储节点状态、注册的服务信息、应用配置、通信策略等。
- Consul在选主和数据事务中都使用了Raft算法，但是其WAN(多个数据中心间)、LAN(节点间)通信、故障广播使用了Gossip算法（流行病协议）。
- 在单集群内，Consul server默认的Follower节点收到读写请求后，会转发给Leader处理
- 若查询的服务不在本集群，本地的Leader转发给远程集群的Leader处理。

## 运维高可用管理

目前分布式系统的高可靠实现，均受限于**CAP**（一致性（Consistency）、可用性（Availability）和分区容错性（Partition tolerance））理论，根据不同业务场景的会采取三者之间的平衡取舍。本文的Consul是采用Raft算法来实现的，采用此算法的还有EverDB、RocketDB、Etcd、Kafka3等系统。

**Raft算法状态机**：

![](/images/解读consu-01-2022-02-16-14-50-29.png)

主要有三种状态`Follower`、`Candidate`和`Leader`，其中值得关注有以下几个方面：

- 第一次启动后，默认进入为`Follower`，经过随机时间（Election Time）后，进入`Candidate`状态，发起投票流程。
- 收到大多数节点投票，可以成为`Leader`
- 非拜占庭环境，每进行一次选举，选举周期Term就回自加1，默认大家都遵从term高的节点
- 选主成功后，Leader会发起心跳保活
- Follower可以根据CAP相性调整，一般是收到请求后转发给Leader处理。

详情可查看[动画](http://thesecretlivesofdata.com/raft/)展示。

从Raft算法原理可以得出，Leader选举和日志同步都只需要大多数的节点正常互联即可，所以少量节点故障或网络异常不会影响系统的可用性。

即使Leader故障，在选举超时后，剩余节点会自发选举新Leader，无需人工干预，RTO时间极小。如果网络发生脑裂，待网络恢复后，也会保证数据最终一致性。

**Gossip通信**

Gossip是Consul节点间状态同步的，下图为形象表示：

![](/images/gossip流行病.gif)

可以理解为`一传十、十传百`，随机选择几个节点发出消息，收到信息的节点在向其他节点传播，这种方式在数据量大、环境复杂的情况下比`一传百`更具备可扩展性和可靠性。Consul默认有两个Gossip池，LAN池和WAN池。

**LAN池** :Consul中的每个数据中心有一个LAN池，它包含了这个数据中心的所有成员，包括clients和servers。LAN池用于以下几个目的:

- 成员关系信息允许client自动发现server, 免去如何发现server端的配置。
- 分布式失败检测机制使得由整个集群来做失败检测这件事， 而不是集中到几台机器上。
- gossip池使得类似Leader选举这样的事件，可以更可靠、迅速的传达。

**WAN池**: WAN池是全局唯一的，因为所有的server都应该加入到WAN池中，无论它位于哪个数据中心。由WAN池提供的成员关系信息允许server做一些跨数据中心的请求。一体化的失败检测机制允许Consul优雅地去处理：整个数据中心失去连接， 或者仅仅是别的数据中心的某一台失去了连接。

## 数据读写一致性

![](/images/解读consu-01-2022-02-16-15-30-35.png)

数据写入必须走Leader，经过两阶段提交（propose、prewrite(append、replicate)、commited、apply）过程，详情可以参考[这里](/docs/3-devops/tidb/tidb-初体验/#多副本raft一致性)，读取数据Consul有[三个](https://www.consul.io/docs/architecture/consensus)级别可以配置。

- consistent： Leader处理数据前，要和所有节点确认自己还是Leader，每次请求都会额外的网络请求
- default: 引入Leader Lease机制，一定时间窗口内，Learder利用状态缓存,直接处理请求。
- stale: 所有server节点都可以直接回应请求，会产生Follower节点反应过久的数据。

## ACL授权

TODO.

**client说明：**

configmap中的字段：
client的配置datacenter、encrypt

- `-encrypt`：和sever段保持一致，可用`consul keygen`生成，只在启动后加载一次，之后便不需要了，用于加密consul的gossip协议通信。
- `datacenter`: 和要接入的server端保持一致，代表consul集群名称。
- `acl.tokens.agent`: acl申请的token，代表了该client可以操作服务、配置的权限

deployment的字段：
- `CONSUL_RETRY_JOIN_ADDR`里的值，会填充到 `join`和 `--retry-join`参数中，代表要接入的consul集群，前者加入一次不成功会失败，后者会重复尝试直到成功。
- `CONSUL_NODE_NAME`，会填充到`-node`参数，默认是hostname，要求全局唯一，此处代表要接入注册中心的系统的名字。
- `AGENT_TOKEN`：acl里申请的token

**server说明**：
