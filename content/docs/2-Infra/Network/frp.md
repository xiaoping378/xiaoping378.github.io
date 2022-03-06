---
tags: ["网络"]
title: "frp内网穿透开发"
linkTitle: "frp内网穿透开发"
weight: 01
description: >
 利用frp实现外网代理ssh到内网开发机
---

## FRP介绍

frp 是国人开源的一种快速反向代理，可将 NAT 或防火墙后面的本地服务器暴露到互联网上。目前，它支持暴露 TCP 和 UDP，以及 HTTP 和 HTTPS 协议，也可以通过域名将请求转发到内部服务。

![](/images/frp-2022-03-06-20-21-49.png)

## 安装

二进制下载[地址](https://github.com/fatedier/frp/releases), 解压后内容如下：

```bash
➜  frp_0.39.1_linux_amd64 tree
.
├── frpc
├── frpc_full.ini
├── frpc.ini
├── frps
├── frps_full.ini
├── frps.ini
├── LICENSE
└── systemd
    ├── frpc.service
    ├── frpc@.service
    ├── frps.service
    └── frps@.service

1 directory, 11 files
```

可将其中的 frpc 拷贝到内网服务所在的机器上，将 frps 拷贝到具有公网 IP 的机器上，放置在任意目录。

后续使用思路：
- 编写配置文件，先通过 ./frps -c ./frps.ini 启动服务端，
- 再通过 ./frpc -c ./frpc.ini 启动客户端。
- 如果需要在后台长期运行，可使用压缩包内写好的的systemd服务
  
## 代理ssh

本人日常开发使用vsocde，众所周知，其[remote SSH开发](https://code.visualstudio.com/docs/remote/ssh)的特性非常惊艳，可以提供本地开发的体验，背后需要通过ssh连接到远端服务器，高配云主机价格不菲，公司电脑长期荒废中，，，，遂有了此想法，只购买一个低配的云主机用来代理到内网的公司电脑（已安装linux）上，然后利用vscode实现内网穿透开发。


### 服务端
外网云主机用作服务端的配置：
```bash
➜  ~ cat /etc/frp/frps.ini
# common为服务端基础配置
[common]
# 服务端监听的端口，用于和内网客户端建立通信，来接受需要代理服务的配置。
bind_port = 7380
# 服务端与客户端建立连接的Token，两边保持一致即可
token = PassWord
```

启动：

```bash
frps -c /etc/frp/frps.ini
```

### 客户端

内网本地linux上，编写客户端配置：
```bash
# common为客户端基础配置
[common]
# 上例服务端运行的外网IP
server_addr = 39.*.*.*
server_port = 7380
token = PassWord

# 需要代理的服务配置，可以根绝自身情况开启多个代理服务如web、 后端api等。
[ssh]
type = tcp
# 本地服务信息
local_ip = 127.0.0.1
local_port = 22
# 配置服务端代理端口， remote 2222 -> local 22
remote_port = 2222
```

启动：
```bash
frpc -c ./frpc.ini
```

如上设置运行，可在任意机器上，通过 ``ssh -p 2222 root@39.*.*.*`` 登录内网机器。

### vscode连接

安装正常步骤添加远端SSH机器，即可。
![](/images/frp-2022-03-06-21-22-51.png)

## 总结

frp还有很多其他安全、UI、负载均衡等方面的配置，可根据自身环境选择，服务端和客户端选项更全面的配置[见此](https://gofrp.org/docs/reference/)

总体上使用frp代理简单的服务，配置上可以很简单，但要配置稍复杂的场景，目前的设计感觉有些缺陷，让人容易犯迷糊，比如设计多种端口字段，来开启某特性。