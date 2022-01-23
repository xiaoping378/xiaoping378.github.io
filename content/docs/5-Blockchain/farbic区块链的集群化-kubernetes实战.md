---
title: "farbic-区块链的生产集群化"
linkTitle: "farbic-区块链的生产集群化"
tags: ["区块链"]
weight: 9
date: 2017-01-05
description: >
  介绍fabric示例中的生产中的集群化操作
---

默认社区的demo是基于docker-compose给出的，达到了“一键部署”的效果，但生产上考虑多节点的情况，还需要费些手脚，这里考虑用kompose结合k8s来做这件事。

### k8s集群 1.7的初始化

每个节点都要安装docker的步骤，此处略过不表，这里主要介绍利用kubeadm初始化k8s集群，这里不考虑k8s集群本身的高可用，以前有文章专门介绍过。

```bash
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm
# 默认会自动安装这些包 ebtables kubeadm kubectl kubelet kubernetes-cni socat
```
如果你本机是centos的话，可以用如下命令安装
```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y kubelet kubeadm
systemctl enable kubelet && systemctl start kubelet
```

上面的命令，需要翻墙才能跑通，没条件的可以去[release项目](https://github.com/kubernetes/release)自己编译deb包或者rpm包，如下运行

```bash
git clone https://github.com/kubernetes/release.git && cd release

# debian系如下
docker build --tag=debian-packager debian
docker run --volume="$(pwd)/debian:/src" debian-packager
# docker run -e "HTTPS_PROXY=127.0.0.1:8118" --net=host --volume="$(pwd)/debian:/src" debian-packager
# 默认debs包在目录debian/bin/stable/xenial下

# centos系的如下
cd rpm
./docker-build.sh
#默认rpm包在目录output/x86_64/下
```

必要依赖搞到手后，就可以简单的利用kubeadm启动集群了
在master节点上如下执行初始化，此过程会启动 etcd，controller-manager，scheduler，api-server组件
```bash
kubeadm init --kubernetes-version stable-1.7 --pod-network-cidr=10.244.0.0/16
```

在其他节点上执行join操作
现在必须要加上`--node-name`参数，不然报错误，这是个[bug](https://github.com/kubernetes/kubeadm/issues/347)
```bash
kubeadm join --token 4cc663.c4d99a546c9f3974 192.168.10.78:6443 --node-name node-0
```


默认还需要启动pod network，我默认用的flannel。
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
```

另外默认master节点是不会被调度容器的，如下可放开限制
```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

默认我们得到如下的集群状态
```bash
➜  ~ kubectl get no
NAME      STATUS    AGE       VERSION
air13     Ready     2h        v1.7.2
node-0    Ready     17m       v1.7.2
```

### 准备fabric的k8ss所需yaml文件

这里需要用到下载我改过的kompose工具，默认官方的对hostpath的处理，需要引入PV，PVC，虽然这样无可厚非，但对与现阶段的我增加了不必要的复杂度，就动手加了个`--hostpaths`的选项，

```bash
# 下载我改过的工具项目-kompose
git clone https://github.com/xiaoping378/kompose
cd kompose && make
# 如上会编译出kompose来，自己搞到$PATH里
```

这里以[fabric-samples项目](https://github.com/hyperledger/fabric-samples.git)里的balance-transfer为例，演示一个完整的CC运行在k8s上。

下载fabric-samles项目，
```bash
git clone https://github.com/hyperledger/fabric-samples.git
cd fabric-samples/balance-transfer

kompose convert -f artifacts/docker-compose.yaml -d --hostpaths
# 如上会在当前目录出现批量的deployment和service yaml文件，这里需要针对hostpath的volumes稍作修改

sed -i 's/.\/channel/\/root\/balance-transfer\/artifacts\/channel/' *-deployment.yaml
#如上改成绝对路径，另外还需要保证各节点都要有channel目录
rsync -avz balance-transfer root@node-0:~/
```

因为ca, peer, orderer都需要从本地读取证书相关的信息，所以要把各节点利用`nodeSelector`特性绑定到指定的节点上，这一点以后得改掉，利用env来动态生成（待验证）
```bash
#比如让要在特定节点调度特定容器，需要如下操作
kubectl label node node0 ca=ture
```
还要在对应的deployment文件做如下操作

![nodeSelector](/nodeSelector.png)



-- 未完待续。。。