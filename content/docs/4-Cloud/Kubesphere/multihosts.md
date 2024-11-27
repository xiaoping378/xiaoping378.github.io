---
tags: ["k8s"]
title: "论Kubesphere的异地多活方案"
linkTitle: "论Kubesphere的异地多活方案"
weight: 41
description: >
  混合容器云管理平台Kubesphere的异地多活方案... 
---

{{% pageinfo %}}
遇到这样一个场景，在同一套环境中需要存在多个host控制面集群...bulabula... 因此想探索下kubesphere的异地多活混合容器云管理方案
{{% /pageinfo %}}

## 集群角色介绍

一个兼容原生的k8s集群，可通过`ks-installer`来初始化完成安装，成为一个QKE集群。QKE集群分为多种角色，默认是none角色（standalone模式），开启多集群功能时，可以设置为host或者member角色。

![多集群](/images/kcp-多集群.png)

- none角色，是最小化安装的默认模式，会安装必要的ks-apiserver, ks-controller-manager, ks-console和其他组件
  - ks-apiserver, kcp的API网关，包含审计、认证、权限校验等功能
  - ks-controller, 各类自定义crd的控制器和平台管理逻辑的实现
  - ks-console, 前端界面UI
  - ks-installer, 初始化安装和变更QKE集群的工具，由shell-operator触发ansible-playbooks来工作。
- member角色，承载工作负载的业务集群，和none模式的组件安装情况一致
- host角色，整个混合云管理平台的控制面，会在none的基础上，再额外安装tower，kubefed-controller-manager， kubefed-admission-webhook等组件
  - tower，代理业务集群通信的server端，常用于不能直连member集群api-server的情况
  - kubefed-controller-manager，社区的[kubefed](https://github.com/kubernetes-sigs/kubefed)联邦资源的控制器
  - kubefed-admission-webhook， 社区的kubefed联邦资源的动态准入校验器

## 多集群管理原理

上段提到QKE有3种角色，可通过修改`cc`配置文件的`clusterRole`来使能, ks-installer监听到配置变化的事件，会初始化对应集群角色的功能。
```bash
kubectl edit cc ks-installer -n kubesphere-system
```
> 角色不要改来改去，会出现莫名问题，主要是背后ansible维护的逻辑有疏漏，没闭环

### host集群

host角色的主集群会被创建25种联邦资源类型Kind，如下命令可查看，还会额外安装kubefed stack组件。
``` bash
➜  kubectl get FederatedTypeConfig  -A
```

此外api-server被重启后，会根据配置内容的变化，做两件事，注册多集群相关的路由和缓存同步部分联邦资源。
 - 添加url里包含`clusters/{cluster}`路径的agent路由和转发的功能，要访问业务集群的信息，这样可以直接转发过去。
 - cacheSync，缓存同步联邦资源，这里是个同步的操作。
> 当开启多集群后，如果某个member出现异常导致不可通信，那host的api-server此时遇到故障要重启，会卡在cacheSync这一步，导致无法启动，进而整个平台无法访问。

controller-manager被重启后，同样会根据配置的变化，把部分资源类型自动转化成联邦资源的逻辑，也就是说，在host集群创建的这部分资源会自动同步到所有成员集群，实际的多集群同步靠kubefed-controller-manager来执行。以下资源会被自动创建联邦资源下发：
  - users.iam.kubesphere.io -> federatedusers.types.kubefed.io
  - workspacetemplates.tenant.kubesphere.io -> federatedworkspaces.types.kubefed.io
  - workspaceroles.iam.kubesphere.io -> federatedworkspaceroles.types.kubefed.io
  - workspacerolebindings.iam.kubesphere.io -> federatedworkspacerolebindings.types.kubefed.io

此外还会启动cluster、group和一些globalRole*相关资源的控制器逻辑，同上也会通过kubefed自动下发到所有集群，`clusters.cluster.kubesphere.io`资源除外。

> 如果以上资源包含了`kubefed.io/managed: false`标签，kubefed就不会再做下发同步，而host集群下发完以上资源后，都会自动加上该标签，防止进入死循环

### member集群

修改为member集群时，需要cc中的**jwtSecret**与host集群的保持一致(若该值为空的话，ks-installer默认会随机生成)，提取host集群的该值时，需要去cm里找，如下：
```bash
kubectl -n kubesphere-system get cm kubesphere-config -o yaml | grep -v "apiVersion" | grep jwtSecret
```
> jwtSecret要保持一致，主要是为了在host集群**签发**的用户token，在用户访问业务集群时token**校验**能通过。

### 添加集群

本文只关注`直接连接`这种情况，当填好成员集群的kubeconfig信息，点击`添加`集群后,会做如下校验：
- 通过kubeconfig信息先校验下是否会添加已存在的重复集群
- 校验成员集群的网络连通性
- 校验成员集群是否安装了ks-apiserver
- 校验成员集群的`jwtSecret`是否和主集群的一致

> 写稿时，此处有个问题，需要修复，如果kubeconfig使用了`insecure-skip-tls-verify: true`会导致该集群添加失败，经定位主要是kubefed 空指针panic了，后续有时间我会去fix一下，已提[issue](https://github.com/kubesphere/kubesphere/issues/4891)。

校验完必要信息后，就执行实质动作`joinFederation`加入联邦，kubesphere多集群纳管，实质上是先组成联邦集群:
- 在成员集群创建ns kube-federation-system
- 在上面的命名空间中创建serviceAccount [clusterName]-kubesphere, 并绑定最高权限
- 在主集群的kube-federation-system的命名空间创建`kubefedclusters.core.kubefed.io`，由kubefed stack驱动联邦的建立
- 加入联邦后，主集群的联邦资源会通过kubefed stack同步过去
> 上述一顿操作，等效于 `kubefedctl join member-cluster --cluster-context member-cluster  --host-cluster-context host-cluster`


## 异地多活方案设计

异地多活的方案主要是多个主集群能同时存在，且保证数据双向同步，经过上面的原理分析，可知多个主集群是可以同时存在的，也就是一个成员集群可以和多个主集群组成联邦。整体方案示意图设计如下：

![](/images/kcp-multi-hostclusters.png)

> 以下操作假设本地已具备三个QKE集群，如果不具备的可按照[此处](/docs/4-cloud/kubesphere/kind-multicluster-dev/)快速搭建`host、host2、member`3个集群

大致实现逻辑的前提介绍：

1. 三个集群的`jwtSecret`得保持一致
2. 两个主集群都去`添加`纳管同一个member集群
3. 利用`etcdctl make-mirror`实现双向同步

### 验证下可行性

实操双活前，先验证下可行性

**实验1：**

在两边创建一个同名用户，用户所有信息一致，可以添加成功，然后再修改一边的用户信息，使两边不一致

可以看到member集群的用户xxp，一直会被两边不断的更新...
```bash
root@member-control-plane:/# kubectl get user xxp -w
NAME   EMAIL         STATUS
xxp    xxp@163.com   Active
xxp    xxp-2@163.com   Active
xxp    xxp@163.com    Active
xxp    xxp-2@163.com   Active

... 周而复始 ...
```
这个实验，即使在创建用户时，页面表单上两边信息填的都一样，也会出现互相刷新覆盖的情况，因为yaml里的uid和time信息不一致

**实验2：**

在两边添加一个同名用户，但两边用户信息（用户角色）不一致，可以创建成功，但后创建者的kube-federa会同步失败, 到这里还能接受，毕竟有冲突直接就同步失败了

但member集群上该用户的关联角色会出现上文的情况，被两边的主集群持续反复地修改...


**实验3：**

在一侧的主集群上尝试修复冲突资源，即删除有冲突的用户资源，可以删除成功，但对应的联邦资源会出现删失败的情况
```
➜  ~ kubectl get users.iam.kubesphere.io
NAME    EMAIL                 STATUS
admin   admin@kubesphere.io   Active
xxp3    xxp3@163.com          Active
➜  ~
➜  ~ kubectl get federatedusers.types.kubefed.io
NAME    AGE
admin   5h33m
xxp     65m #这里是个删不掉的资源，fed controller会重复做失败尝试
xxp3    61m
```

这样就会出现，两个主集群：一个要删，一个要同步，member集群上：持续上演“一会儿消失，一会儿又出现了”的奇观。

### 总结

两个主集群可以同时工作，一旦出现同名冲突资源，处理起来会非常麻烦，尤其是背后的Dependent附属资源出现冲突时，往往问题点隐藏的更深，修复起来也棘手...

后来调研也发现：目前的社区方案make-mirror只支持单向同步，适合用来做灾备方案。

所以容器云平台的双活，除非具备跨AZ的etcd集群，否则需要二次开发改造类make-mirror方案来支持了。我最开始要考虑的问题答案也就显而易见了：如果要多个host集群共存，必须考虑通过行政管理手段，来尽量避免同名资源冲突。