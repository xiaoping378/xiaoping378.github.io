---
tags: ["k8s"]
title: "使用kind本地启动多集群"
linkTitle: "kind本地启动多集群"
weight: 40
description: >
  在本地利用kind搭建多个集群，验证多集群纳管... 
---

{{% pageinfo %}}
我本地4c/8G的小本儿，跑了两个集群，组建了多集群环境，还行，能玩动...
{{% /pageinfo %}}


## 环境篇

1. kind[安装](https://kind.sigs.k8s.io/docs/user/quick-start/)

2. 镜像准备

    视网络情况，可以把依赖镜像`kindest/node`提起pull到本地

3. docker的data-root目录

    尽量不要放到/var目录下，kind起的集群容器会占用比较大的空间

## 实操

1. 创建集群

执行完如下命令后，docker ps可以看到本地启动了两个容器，一个容器对应一个集群。
```bash
kind create cluster --image kindest/node:v1.19.16 --name host
kind create cluster --image kindest/node:v1.19.16 --name member
```

`kubectl config use-context [kind-host | kind-member]`，可以切换kubecl执行的上下文

2. 安装kubesphere

分别在两个集群各自安装ks组件
```bash
# 集群1安装
kubectl config use-context kind-host
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.2.1/kubesphere-installer.yaml   
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.2.1/cluster-configuration.yaml

# 集群2安装
kubectl config use-context kind-member
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.2.1/kubesphere-installer.yaml   
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.2.1/cluster-configuration.yaml
```


3. 纳管集群

可以在上面的初始化阶段直接改好主和成员集群的关系，这里参考[官文](https://kubesphere.com.cn/docs/multicluster-management/enable-multicluster/direct-connection/)即可

host集群的UI地址，可以通过`host容器IP:30880`来访问，主集群的容器ip，可以如下获取：
```bash
docker inspect --format '{{ .NetworkSettings.Networks.kind.IPAddress }}' host-control-plane
```

实操`添加`集群时，需要member集群的kubeconfig，可以用如下命令获取到
```bash
kind get kubeconfig --name member
```
记得把kubeconfig中的`server`地址中改成`member容器ip:6443`，这样host集群才能访问到member集群

## 总结

验证功能、测试开发，挺方便的，可以视本地资源紧张情况停掉监控的ns。

现在kind启动的集群默认使用了containerd的runtime，若想进一步调试查看集群内的情况，可以内部集成的`crictl`代替熟悉的docker工具。