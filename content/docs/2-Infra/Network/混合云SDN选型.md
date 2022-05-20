---
title: "混合云网络SDN"
linkTitle: "混合云网络SDN"
weight: 3
description: >
  目前云上的SDN网络已相对成熟，本文为初步介绍，未完... 
---

* TODO.

### 为什么需要SDN

* 网络可编程
* VPC（Virtual Private Cloud）

### 现有SDN方案

* 硬件方案（软件定义，硬件实现）
  * 主流网络设备厂商有各自实现

* 软件方案（NFV）
  * VMWare NSX, Juniper OpenContrail, OpenStack DVR...

### 业务需求

* 用户网络隔离 - 多租户
* 保证中等流量规模的高性能低延迟
* 适应复杂异构的基础架构（混合云-- kubernetes，虚机，裸机）
* 端点迁移，IP不变
* 负载均衡（L2/L3）
* 端到端流量精细ACL
* 可API控制
* 运维监控（包，字节流）

### 方案选型
* 成本
* 设备依赖

### 开源方案

各开源SDN方案对比：
* flannel vxlan:
  不具备网络隔离功能。

* OpenShift SDN:

  基于vxlan利用ovs-multienant可实现基于项目的网络隔离，和flannel vxlan相比，其使用的ovs-subnet插件，数据流场景大体一致，容器向外网发包也使用的NAT。

* Calico:

  支持混合云，安全加密，
  纯3层的路由实现保证了性能和低延迟
  支持了网络隔离和ACL
  但存在目前只支持TCP、UDP、ICMP、ICMPv6协议，四层协议不支持。

* OpenStack Neutron

  支持网络隔离
  性能和低延迟 -- 需要优化
  支持多租户
  基于ML2支持混合云方案 -- kubernetes的支持需要第三方的kubestack项目
  虚机迁移，IP可不变，容器迁移，IP不变 -- 需要开发
  支持负载均衡LBaaS
  支持精细级的ACL
  API ?
  可运维监控基本数据

* OpenContrail

  **完全满足我们的网络需求**, 值得深入研究

  Juniper开源的SDN & NFV方案

  已经集成支持OpenStack, VMware, Docker 和 Kubernetes.

### 各大厂商公开资料

* 网易蜂巢： VxLAN, 基于Openstack Neutron
* 待补充...
