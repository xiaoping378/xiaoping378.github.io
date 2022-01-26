---
tags: ["容器", "K8S"]
title: "TKEStack all-in-one入坑指南"
linkTitle: "TKEStack all-in-one入坑指南"
weight: 25
description: >
  TKEStack的all-in-one安装、多租户和多集群管理功能解读
---

{{% pageinfo %}}
本文主要介绍当前最新版本TkeStack 1.8.1 的allinone安装入坑指南和基本核心功能介绍。
{{% /pageinfo %}}

## 安装实录

官方推荐至少需要2节点方可安装，配置如下，**硬盘空间**一定要保障。也支持ALL-in-ONE的方式安装，但有BUG。

![](/images/TKEStack-allinone-2022-01-25-08-47-30.png)

## 启动init服务

启动init服务，即安装tke-installer和registry服务，安装命令行如下：
```bash
arch=amd64 version=v1.8.1 \
    && wget https://tke-release-1251707795.cos.ap-guangzhou.myqcloud.com/tke-installer-linux-$arch-$version.run{,.sha256} \
    && sha256sum --check --status tke-installer-linux-$arch-$version.run.sha256 \
    && chmod +x tke-installer-linux-$arch-$version.run \
    && ./tke-installer-linux-$arch-$version.run  
```

如上命令执行后，会下载8G左右的安装包，并执行解压后的install.sh脚本，启动3个容器：1个为tke-installer和另2个为registry仓，且为containerd容器，需要使用`nerdctl [images | ps]`等命令查看相关信息。

通过查看脚本，上文启动的本地registry的启动命令等效如下：
```bash
nerdctl run --name registry-https -d --net=host --restart=always -p 443:443 \  
    -v /opt/tke-installer/registry:/var/lib/registry \  
    -v registry-certs:/certs \  
    -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \  
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/server.crt \  
    -e REGISTRY_HTTP_TLS_KEY=/certs/server.key  \  
    tkestack/registry-amd64:2.7.1  
```

还有个http 80的registry，这里不贴了，后面的部分坑，就是这里埋下的，预先占用了节点的80和443端口，后面tke的gateway pod会启动失败。

## 启动TKE集群

上章节执行完后，会启动tke-installer（一个web操作台），通过访问本地8080端口，可访问界面操作安装global集群。按照官方指引操作就行，此处不表。另外需要说明的是在安装过程中，如果要查看本地容器，不能使用`docker ps`了，需要使用`nerdctl -n k8s.io ps`。整个安装过程是使用ansible和kubeadm完成的，kubelet是通过systemd启动的，k8s组件为静态pod。

因为我是使用的ALL-in-ONE安装，遇到了不少问题，可详见FAQ如何解决。安装成功后会提示如下指引：
![](/images/TKEStack-allinone-2022-01-25-09-10-56.png)

默认初始安装后，很多pod是双副本的，我这里仅是验证功能使用，全部改成了单副本。

## 多租户管理

tkestack采用Casbin模型作为权限管理，TODO.

## FAQ

### 安装过程出现循环等待apiserver启动

```log
2022-01-19 14:43:32.225 error   tke-installer.ClusterProvider.OnCreate.EnsureKubeadmInitPhaseWaitControlPlane   check healthz error {"statusCode": 0, "error": "Get \"https://****:6443/healthz?timeout=30s\": net/http: TLS handshake timeout"}
```

我这里是因为在installer上指定的master的IP为外网IP（我使用外网IP是有原因的，穷... 后面需要跨云厂商组集群），通过查看kubelet日志提示本机找不到IP，如下开启网卡多IP，可通过。

```bash
ip addr add 118.*.*.* dev eth0
```

### Gateway POD启动失败

我这里是因为init节点和gobal master节点，共用了一个，本registry服务占用了80和443端口，需要修改gateway hostNetwork为false，另外可以通过修改svc 为nodePort，还需要修改targetPort，官方现在这里有bug，不知道为指到944*的端口上，我这里设置的30080来访问安装好的集群。

### 页面登录错误Unregistered redirect_uri

官方没有相关说明，一切都是ALL-in-ONE的原因，我改动了默认集群console的访问端口为30080。。。 通过查看源码发现是每次认证时dex会校验tke-auth-api向它注册过的合法client地址。于是我就修改了tke命名空间下tke-auth-api的相关configmap：

![](/images/TKEStack-allinone-2022-01-25-09-47-16.png)

重启tke-auth-api后，问题依旧存在，继续源码走查，发现这玩意儿叫init真的只发挥一次作用，改完配置，不会重新读取，细读逻辑发现etcd中不存在这个key，会重新读取写入一次，于是决定删除etcd中的相关key。

```bash
etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/apiserver-etcd-client.crt --key=/etc/kubernetes/pki/apiserver-etcd-client.key del /tke/auth-api/client/default  --prefix
```

### 添加节点的过程中failed，无法删除节点重试

ssh信息设置完后，如果中间出问题，会陷入无限重试...

![](/images/TKEStack-allinone-2022-01-25-09-48-43.png)

遇事不决，看日志，找不到日志，看源码...

通过翻找源码，发现是`platform`相关组件在负责，查看相关日志`kubectl -n logs tke-platform-controller-*** --tail 100 -f`，定位问题，我这里是以前各种安装的残留信息，导致添加节点初始化失败。删除之... 解决。

为避免添加节点`no clean`再次出现问题，建议预先执行下[clean.sh](https://tke-release-1251707795.cos.ap-guangzhou.myqcloud.com/tools/clean.sh)脚本。


## 小技巧

如下使用，可以愉快的敲命令了，因为我是用oh-my-zsh的shell主题(没有自动加载kubectl plugin)，kubectl的命令补全使用zsh，可根据实际情况调整。

```bash
source <(nerdctl completion bash)  
source <(kubectl completion zsh) 
```
