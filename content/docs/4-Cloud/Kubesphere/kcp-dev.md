---
tags: ["k8s", "dev"]
title: "搭建kubesphere的开发调试环境"
linkTitle: "搭建KCP的开发环境"
weight: 42
description: >
  记录搭建kubesphere的开发调试环境 - 后端篇
---

## 依赖工具介绍

1. vscode，个人日常使用vscode开发，标配`Remote Development`插件
2. kt-connect，本地开发使用的流量代理工具，可以双向代理，本地可以直接访问pod和svc，也可以转发访问pod的流量到本地，相关[原理介绍](https://github.com/alibaba/kt-connect/blob/master/docs/zh-cn/reference/mechanism.md)。


## 准备调试环境

### 连接集群网络

使用kt-connect，打通网络，本地可直接访问Kubernetes集群内网

```bash
sudo ktctl connect --context kind-host  --portForwardTimeout 300
```

- `ktctl`采用本地`kubectl`工具的集群配置，默认为~/.kube/config文件中配置的集群。
- 如果kubeconfig有多个集群，可以通过`--context`指定要连接的具体集群
- 如果本地环境是`kind`集群，需要修改kubeconfig中server的127地址为`容器IP:6443`
- 如果网络比较差，会遇到错误`ERR Exit: pod kt-rectifier-tcxjk failed to start`，可适当增加等待时间`portForwardTimeout`

> 另外注意的是，kt-connect需要root权限，上条命令会默认读取`/root/.kube/config`文件，自行copy或者另通过`-c`指定文件

### clone代码

不表.

### 编辑调试的配置文件

首先编辑vscode的调试配置文件, 我是如下配置的：

```bash
➜  kubesphere git:(master) cat .vscode/launch.json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "ks-apiserver",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${workspaceFolder}/cmd/ks-apiserver/apiserver.go"
        },
        {
            "name": "controller-manager",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${workspaceFolder}/cmd/controller-manager/controller-manager.go"
        }
    ]
}
```

### 准备调试环境

按F5，启动调试，左上角选择要调试的组件，我这里以controller-manager举例（需要hack的注意点比较多）。

过会儿会发现出现错误，错误提示很明显，因缺少配置文件，导致无法启动，通过查看`deployment yaml`, 发现后面还会缺失`Admission Webhooks`的证书，可如下统一提取到本地：
```bash
# 提取启动的配置文件(调试apiserver的时候也需要这一步，但要把文件放到对应cmd/ks-apiserver目录下)
kubectl -n kubesphere-system get cm kubesphere-config -ojsonpath='{.data.kubesphere\.yaml}' > cmd/controller-manager/kubesphere.yaml
# 提取webhook用到的证书
mkdir -p /tmp/k8s-webhook-server/serving-certs/
export controller_pod=`kubectl -n kubesphere-system get pods -l app=ks-controller-manager  -o jsonpath='{.items[0].metadata.name}'`
kubectl -n kubesphere-system exec -it ${controller_pod} -- cat /tmp/k8s-webhook-server/serving-certs/ca.crt > /tmp/k8s-webhook-server/serving-certs/ca.crt
kubectl -n kubesphere-system exec -it ${controller_pod} -- cat /tmp/k8s-webhook-server/serving-certs/tls.crt > /tmp/k8s-webhook-server/serving-certs/tls.crt
kubectl -n kubesphere-system exec -it ${controller_pod} -- cat /tmp/k8s-webhook-server/serving-certs/tls.key > /tmp/k8s-webhook-server/serving-certs/tls.key
```

继续启动，发现还会有缺文件的错误，应该是编译镜像时，内置了些文件，通过查看`build/ks-controller-manager/Dockerfile`，发现后面会缺的东西还是比较多的，推荐直接从运行中的pod直接copy到本地：
```bash
sudo mkdir /var/helm-charts/
sudo chmod -R a+rw /var/helm-charts
kubectl -n kubesphere-system cp ${controller_pod}:/var/helm-charts  /var/helm-charts/
```

继续启动，成功 ！

## 开始调试

利用ktctl替换集群中的ks-controller-manager的服务为本地服务。
```bash
sudo ktctl exchange  ks-controller-manager --namespace kubesphere-system --mode scale  --recoverWaitTime 300 --expose 8443:8443
```

> 如果本地集群只有一个节点，上述命令会一直pending，可以通过如下命令替代
> ```bash
> sudo ktctl exchange  ks-controller-manager --namespace kubesphere-system  --expose 8443:8443
> kubectl -n kubesphere-system scale deployment ks-controller-manager --replicas=0
> ```
> 结束调试，记得还原刚才缩容`replicas`的设置。


后续就是vscode的正常断点调试或者本地开发验证了，有时间在整理贴图...