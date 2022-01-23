---
tags: ["容器"]
title: "构建生产环境级的docker Swarm集群-1"
linkTitle: "构建生产环境级的docker Swarm集群-1"
weight: 1
description: >
  构建生产环境级的docker Swarm集群。 
---

此文档适用于低于1.12版本的docker，之后swarm已内置于docker-engine里。

1. 硬件需求

  至少5台PC服务器, 分别如下作用
  * manager0
  * manager1
  * consul0
  * node0
  * node1

2. 每台PC上安装docker-engine

  一台一台的ssh上去执行，或者使用ansible批量部署工具。

  安装docker-engine
  ```
  curl -sSL https://get.docker.com/ | sh
  ```
  启动之，并使之监听2375端口
  ```
  sudo docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
  ```
  亦可修改配置，使之永久生效
  ```
mkdir /etc/systemd/system/docker.service.d
cat <<EOF >>/etc/systemd/system/docker.service.d/docker.conf
[Service]
    ExecStart=
    ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --dns 180.76.76.76  --insecure-registry registry.cecf.com -g /home/Docker/docker
EOF
  ```

3. 启动discovery后台

  在consul0上启动consul服务，manager用其来认证node连接并存储node状态， 理应建立discovery的高可用，这里简化之

  ```
  docker run -d -p 8500:8500 --name=consul progrium/consul -server -bootstrap
  ```

4. 创建Swarm集群

  在manager0上创建the primary manager， 自行替换manager0_ip和consul0_ip的真实IP地址。

  ```
  docker run -d -p 4000:4000 swarm manage -H :4000 --replication --advertise <manager0_ip>:4000 consul://<consul0_ip>:8500
  ```
  在manager1上启动replica manger
  ```
  docker run -d -p 4000:4000 swarm manage -H :4000 --replication --advertise <manager1_ip>:4000 consul://<consul0_ip>:8500
  ```

  --replication

5. 在node上执行加入集群操作

  分别在node0和node1上执行加入集群操作
  ```
  docker run -d swarm join --advertise=<node_ip>:2375 consul://<consul0_ip>:8500
  ```

6. 在manger0上查看集群状态

  ```
  docker -H :4000 info
  ```
