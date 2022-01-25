---
tags: ["容器", "K8S"]
title: "k3s实践-01"
linkTitle: "k3s实践-01"
weight: 20
description: >
  k3s的安装及基本工作方式解读
---

{{% pageinfo %}}
本文主要介绍k3s的安装和核心组件解读。
{{% /pageinfo %}}

k3s是all-in-one的轻量k8s发行版，把所有k8s组件重新打包一个不到100M的二进制文件了。主要具备显著特点：

- 打包成单一二进制
- 默认集成了sqlite3来替代etcd，也可以指定其他数据库：etcd3、mysql、postgres。
- 默认内置Coredns、Metrics Server、Flannel、Traefik ingress、Local-path-provisioner等
- 默认启用了TLS加密通信。

## 安装

官方提供了一键安装脚本[install.sh](https://get.k3s.io) ，执行`curl -sfL https://get.k3s.io | sh -`可一键安装server端。此命令会从`https://update.k3s.io/v1-release/channels/stable`取到最新的稳定版安装，可以通过`INSTALL_K3S_VERSION`环境变量指定版本，本文将以1.19为例。

**启动 k3s server端(master节点).**
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.19.16+k3s1 sh -
```

> 由于网络原因，可能会失败，自行想办法[下载](https://github.com/k3s-io/k3s/releases/download/v1.19.16+k3s1/k3s)下来，放置 `/usr/local/bin/k3s`，附上执行权限`chmod a+x /usr/local/bin/k3s`, 然后上面的命令加上`INSTALL_K3S_SKIP_DOWNLOAD=true`再执行一遍即可。

不出意外，k3s server会被systemd启动，执行命令查看`systemctl status k3s`或者通过软链的kubectl验证是否启动成功：

```bash
➜  k3s kubectl get no
NAME            STATUS   ROLES    AGE     VERSION
gitlab-server   Ready    master   6m43s   v1.19.16+k3s1
```

**(Optional)** 启动 k3s agent端 (添加worker节点).

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://172.25.11.130:6443 K3S_TOKEN=bulabula INSTALL_K3S_VERSION=v1.19.16+k3s1 sh -
```

`K3S_TOKEN`内容需要从server端的`/var/lib/rancher/k3s/server/node-token`文件取出，`K3S_URL`中的IP是master节点的IP。

## 架构

