---
title: "MongoDB之初见"
linkTitle: "MongoDB之初见"
weight: 6
description: >
  介绍MongoDB的基本使用。 
---

1. 手动启动

```bash
# 下载二进制
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-3.4.9.tgz
tar -zxvf mongodb-linux-x86_64-3.4.9.tgz
ln -s $PWD/mongodb-linux-x86_64-3.4.9/bin/* /home/xxp/Software/bin
# 创建数据存储目录	
mkdir mongodb
sudo chown -R $USER ./mongodb
mongod -dbpath=$PWD/mongodb
```
默认监听27017， 根据情况选择关闭warning。


2. 推荐调试方法

默认可以使用``mongo``进入shell交互模式，
亦可使用图形管理界面，推荐``robo3T``, 目前1.1.1版本在ubuntu桌面上有crash问题，需要如下操作：

```bash
curl -O https://download.robomongo.org/1.1.1/linux/robo3t-1.1.1-linux-x86_64-c93c6b0.tar.gz
tar zxvf robo3t-1.1.1-linux-x86_64-c93c6b0.tar.gz

mkdir ~/robo-backup
mv robo3t-1.1.1-linux-x86_64-c93c6b0/lib/libstdc++* ~/robo-backup/
robo3t-1.1.1-linux-x86_64-c93c6b0/bin/robo3t
```

![](/robo3t.png)

1. 基本使用

shell里敲mongo进入交互界面，

手续推荐查看[mongodb中文文档](http://www.mongoing.com/docs/reference/sql-comparison.html)

