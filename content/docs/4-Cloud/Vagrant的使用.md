---
title: "Vagrant实践整理"
linkTitle: "Vagrant实践整理"
weight: 6
description: >
  介绍Vagrant的实践整理。 
---

很早就听说vagrant的大名，是个创建和管理虚机环境的工具，但一直没有机会实践下，近日我的VirtuablBox让我搞砸了，决定试用下，便于快速搭建各种环境。

## 安装vagrant

图省事儿的话，直接`sudo apt install vagrant`就可以安装，不过版本有点儿低，是1.8.1。

通过官方[下载地址](https://www.vagrantup.com/downloads.html), 可直接下载最新的安装包。我这里安装的是1.9.4

```bash
➜  ~ vagrant -v
Vagrant 1.9.4
```

## 常用命令

一个打包好的操作系统在Vagrant中称为Box，即Box是一个打包好的操作系统环境，网上有很多打包好的环境，官方也有下载各种Boxes的[地址](https://atlas.hashicorp.com/boxes/search)

一般使用流程如下：

- vagrant box add 添加box的操作
- vagrant init 初始化box的操作
- vagrant up 启动虚拟机的操作
- vagrant ssh 登录虚拟机的操作

额外还有些常用的命令

- vagrant box list 显示当前已经添加的box列表
- vagrant box remove 删除相应的box
- vagrant halt -f 冷关机（切断电源）
- vagrant suspend 挂起当前的虚拟机

## 实践

目前vagrant 1.9.4支持适配VirtualBox, VMware，Hyper-V, 和 Docker，本文使用的是VirtualBox。

需要你本机已经[安装virtuablbox](https://www.virtualbox.org/wiki/Downloads)环境

一般只要如下初始化，就会有个最新的centos-7虚机环境
```bash
➜  vagrant init centos/7
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
➜  ls
Vagrantfile
➜  vagrant up --provider virtualbox
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Box 'centos/7' could not be found. Attempting to find and install...
    default: Box Provider: virtualbox
    default: Box Version: >= 0
==> default: Loading metadata for box 'centos/7'
    default: URL: https://atlas.hashicorp.com/centos/7
==> default: Adding box 'centos/7' (v1704.01) for provider: virtualbox
    default: Downloading: https://atlas.hashicorp.com/centos/boxes/7/versions/1704.01/providers/virtualbox.box
    default: Progress: 2% (Rate: 94086/s, Estimated time remaining: 1:52:14)
```

但如上需要依赖外网, 国内环境一般会下载失败，我这里介绍一种通过本地iso创建Box的方法，然后通过本地Box启动虚机环境。

这个iso转box的方法需要`Packer`工具，此工具同样和vagrant都是`HashiCorp`公司出品的，目前国外很火的工具，
支持创建各式各样的镜像，包括各种国内外主流的公有云, openstack的镜像，甚至docker镜像都是可以OK的。



### 参考
 - https://github.com/astaxie/go-best-practice/blob/master/ebook/zh/01.2.md
