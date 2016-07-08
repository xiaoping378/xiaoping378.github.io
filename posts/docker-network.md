# docker的网络

自去年就开始推动公司业务使用docker了， 至今也一年多了，但对docker网络的认知一直一知半解。。。

主要是太忙，加上线上业务也没出过关于网络吞吐性能方面的问题，就没太大动力去搞明白， 现在闲下来了，搞之！


### 环境声明

* 以下内容只针对OS: Ubuntu16.04 docker: 1.10.3的环境， 写本文时docker最新的release版本是1.11.2，还有什么CoreOS，Unikernel 之类的（表示都没玩过）。

> docker更新迭代速度太快了，公司业务只用到基本功能，所以没动力跟进它的更新了
> 各种新时代下的产物频出啊， CoreOS为linux的发行版， 没需求，好遗憾.

### docker的网络模式

一开始安装完docker， 它就会默认创建3个网络， 使用__docker network ls__查看

```bash
➜  blog git:(master) docker network ls
NETWORK ID          NAME                DRIVER
46416a43fbc6        bridge              bridge              
45398901e9f0        none                null                
9440a8140e68        host                host
```
当启动一个容器时， 默认使用bridge模式， 可以通过 --net 指定其它模式。

下面先简要说明下各自的概念

* bridge 模式

容器间之所以能通信，就靠宿主机上的docker0了， docker0就是bridge模式下默认创建的虚拟设备名称

```bash
➜  blog git:(master) ✗ ifconfig docker0
docker0   Link encap:Ethernet  HWaddr 02:42:49:56:7c:3b  
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:49ff:fe56:7c3b/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:78103 errors:0 dropped:0 overruns:0 frame:0
          TX packets:47578 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:17485434 (17.4 MB)  TX bytes:82163889 (82.1 MB)
```

ifocnfig可以看到很多信息， mac地址，IP等这些也可以通过参数指定成别的。


* none模式

none网络模式下的容器里是缺少网络接口的，例如eth0等，但会有一个lo设备。

没用过也没见过这样的业务场景， 不做过多说明

* host模式

容器直接操作宿主机的网络栈， 无疑是性能最好的网络模式， 可以认为是无带宽损耗的。



### 细说bridge模式

这也是我们线上在用的网络模式。
