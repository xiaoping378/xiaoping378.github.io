---
tags: ["容器", "K8S"]
title: "k8s的各组件认知"
linkTitle: "k8s的各组件认知"
weight: 3
description: >
  主要介绍k8s中的各核心组件，未完...
---

### flannel

* flannel的设计就是为集群中所有节点能重新规划IP地址的使用规则，从而使得不同节点上的容器能够获得“同属一个内网”且”不重复的”IP地址，
并让属于不同节点上的容器能够直接通过内网IP通信。

* 实际上就是给每个节点的docker重新设置容器上可分配的IP段， ``--bip``的妙用。
这恰好迎合了k8s的设计，即一个pod（container）在集群中拥有唯一、可路由到的IP，带来的好处就是减少跨主机容器间通信要port mapping的复杂性。

* 原理
  * flannle需要运行一个叫flanned的agent，其用etcd来存储网络配置、已经分配的子网、和辅助信息（主机IP),如下

  ```bash
  [root@master1 ~]# etcdctl ls /coreos.com/network
  /coreos.com/network/config
  /coreos.com/network/subnets
  [root@master1 ~]#
  [root@master1 ~]# etcdctl get /coreos.com/network/config
  {"Network":"172.16.0.0/16"}
  [root@master1 ~]#
  [root@master1 ~]# etcdctl ls /coreos.com/network/subnets
  /coreos.com/network/subnets/172.16.29.0-24
  /coreos.com/network/subnets/172.16.40.0-24
  /coreos.com/network/subnets/172.16.60.0-24
  [root@master1 ~]#
  [root@master1 ~]# etcdctl get  /coreos.com/network/subnets/172.16.29.0-24
  {"PublicIP":"192.168.1.129"}
    ```

  * flannel0 还负责解封装报文,或者创建路由。
    flannel有多种方式可以完成报文的转发。

      * UDP
      * vxlan
      * host-gw
      * aws-vpc
      * gce
      * alloc

      下图是经典的UDP封装方式数据流图
      ![UDP](/flannel-packet-01.png)
