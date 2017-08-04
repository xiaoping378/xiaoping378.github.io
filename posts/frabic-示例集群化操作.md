# fabric1.0 - 示例集群化操作

fabric给出的cc样例都是跑在docker-compose上，这里介绍利用已有的docker-compose.yaml如何集群化运行。

### 准备样例CC

以官方`fabric-samples`项目里的balance-transfer为例，准备拆分运行在4个虚机里。

```bash
git clone https://github.com/hyperledger/fabric-samples.git
cd fabric-samples/balance-transfer
```

本CC的示例，主要由3个部分组成:

* 2 CAs
* 1 SOLO orderer
* 4 peers (2 peers per Org)


其中`artifacts`目录里放置了：

* 由 **cryptogen** 工具生成的证书信息，后面运行时需要挂载到各自的peer节点里
* 由 **configtxgen** 工具生成的初始块 `genesis.block` 和 channel配置信息`mychannell.tx`

### 准备集群

根据上面的情况，下面准备四个虚机来集群化操作, 虚机规划信息如下：

* air13, 192.168.10.78, 运行ca1，ca2, 这是我的本机
* node0, 192.168.10.110, 运行orderer
* node1, 192.168.10.114, 运行org1的peer0、peer1
* node2, 192.168.10.115, 运行org2的peer0、peer1

每个虚机都预先安装docker和docker-compose

修改`artifacts/docker-compose.yaml`文件，在每个service下添加如下信息：

```yaml
    extra_hosts:
      - "ca.org1.example.com:192.168.10.78"
      - "ca.org2.example.com:192.168.10.78"
      - "orderer.example.com:192.168.10.110"
      - "peer0.org1.example.com:192.168.10.114"
      - "peer1.org1.example.com:192.168.10.114"
      - "peer0.org2.example.com:192.168.10.115"
      - "peer1.org2.example.com:192.168.10.115"
```

此举的作用是在每个容器的`/etc/hosts`文件里，添加上面的映射，最终docker-compose.yaml文件放置[gist](https://gist.github.com/xiaoping378/8ba8e796552e27277073e56cfd7b281a)上了.


最后一步，因为此CC样例运行时，需要挂载本地目录里一些提前生成好的证书，我们还需要把这么需要挂载的东西，同步到每个虚机里，如下操作：

```bash
rsync -az balance-transfer root@192.168.10.110:~/
rsync -az balance-transfer root@192.168.10.114:~/
rsync -az balance-transfer root@192.168.10.115:~/
```


### 集群化运行

ssh进入虚机，按照规划启动各自的服务

```bash
# 进入air13虚机
cd balance-transfer/artifacts/
docker-compose up --no-deps ca.org1.example.com ca.org2.example.com

# ssh进入node0虚机
cd balance-transfer/artifacts/
docker-compose up --no-deps orderer.example.com

# ssh进入node1虚机
cd balance-transfer/artifacts/
docker-compose up --no-deps peer0.org1.example.com peer1.org1.example.com

# ssh进入node2虚机
cd balance-transfer/artifacts/
docker-compose up --no-deps peer0.org2.example.com peer1.org2.example.com
```

如上fabric的区块链网络已经集群化运行了


### 运行APP并测试CC

在虚机air13上运行我们的前端APP，并测试在网络集群化后的的CC，

此外由于此APP样例的特殊性，还需要修改其他的地方

* config.json文件里的orderer地址改为`"grpcs://192.168.10.110:7050"`
* app/network-config.json里指定地址的地方需要改成[这样](https://gist.github.com/xiaoping378/a599e3bb3080135b2548c1242ca8cc80)

进入balance-transfer目录，如下执行：

```bash
$ npm isntall
$ PORT=4000 node app
```

这样前端APP就运行起来了，接下来利用测试脚本`testAPIs.sh`测试下我们的CC,运行前，还需要修改下脚本里的peer地址，修改的文件在[这里](https://gist.github.com/xiaoping378/dbfd8801b7b125b0d7add7fce7d4e854)

运行脚本：

```bash
./testAPIs.sh
POST request Enroll on Org1  ...

{"success":true,"secret":"DbzvtdmATgve","message":"Jim2 enrolled Successfully","token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1MDE4NzQ5MjUsInVzZXJuYW1lIjoiSmltMiIsIm9yZ05hbWUiOiJvcmcxIiwiaWF0IjoxNTAxODM4OTI1fQ.LUmfEstviquaD5k5oBMd9KUFaKF1s6ZMY8iO67dKiH0"}

。
。
。

GET query Installed chaincodes

["name: mycc, version: v0, path: github.com/example_cc"]

GET query Instantiated chaincodes

["name: mycc, version: v0, path: github.com/example_cc"]

GET query Channels

{"channels":[{"channel_id":"mychannel"}]}

Total execution time : 13 secs ...

```

本样例修改后的完整地址在[这里](https://github.com/xiaoping378/fabric-samples)

### 总结

目前集群化后网络里的peers ledger和orderer的数据还是存储在容器里，是否要考虑挂载出来，不然容器挂了再启动后，数据是否会乱了，后续研究下。

后续还会继续搞下CA，MSP和动态添加节点以及合约迭代升级等问题。
