# KVM和Libvirt在Ubuntu 16.04的实践整理

## 介绍

此前一直用Virtualbox操作虚机的东西，对于个人搭建环境还是显的有些笨重，不能实现Iac的目标，故尝试了Vagrant和Libvirt，综合考虑我选择libvirt继续深入下去，也是希望以后有机会可以深入搞下openstack的nova组件。

## 安装必要依赖


```bash
sudo apt install bridge-utils qemu-kvm virtinst -y
```

- qemu-kvm: 这个负责hypervisor层和仿真器（可以模拟x86, arm体系）.
- virtinst: 安装和管理虚机的命令行工具
- bridge-utils： 创建和管理bridge网络


安装完输入`kvm-ok`查看是否安装OK，另外还`需要重启`以使kvm和libvirt daemon启动。

## 配置网络

未完。。。
