---
title: "fomo3d-上线部署要点"
linkTitle: "fomo3d-上线部署要点"
tags: ["区块链"]
weight: 4
date: 2017-01-05
description: >
  介绍以太坊私合约fomo3d本地调试要点
---

fomo3d游戏一出，国内疯狂clone上线，这里谈下我上线的思路和部署方法（纯手动的^_^，落伍了）

通过[原版合约地址](https://etherscan.io/address/0xa62142888aba8370742be823c1782d17a0389da1#code)，可以一层一层的拔下所有涉及到的合约代码。

目前据我统计共有8个合约，其中有两个闭源合约：

- F3DexternalSettingsInterface
- JIincInterfaceForForwarder

闭源合约不可怕，看明白什么功能，自己hack掉是不影响游戏本身的。

    提前预警，合约的内容细节还是要自己研究的，没时间写太细，

其实这个游戏本身只需要2个合约就可以跑起来，且没实质影响，只是单纯改变了部分利益分配方式。

下面说明，我尽可能少改动原版的情况下，部署上线合约，移除p3d修改后的原版合约代码[在这里](https://github.com/ChungkueiBlock/fomo3d/tree/master/sols)

## 部署前的准备

我一般使用[在线remix](https://remix.ethereum.org/#optimize=true&version=soljson-v0.4.24)工具部署合约在自己的私链上调试，私链建议如下启动（一键解万忧的方式，推荐创世块采用[POA共识](https://github.com/ChungkueiBlock/tools/tree/master/privateEth)-不消耗CPU），这样可以使用remix的debug功能
```
geth  \
    --datadir ./node0\
    --ws\
    --wsaddr 0.0.0.0\
    --wsapi "eth,net,web3,admin,personal,txpool,miner,clique,debug"\
    --wsport 8546\
    --wsorigins "*"\
    --rpc\
    --rpcapi "eth,net,web3,admin,personal,txpool,miner,clique,debug"\
    --rpccorsdomain "*"\
    --rpcaddr 0.0.0.0\
    --rpcport 8545\
    --rpcvhosts "*"\
    --mine\
    --etherbase 0xdbeb69c655b666b3e17b8061df7ea4cc2399df11\
    --unlock 0xdbeb69c655b666b3e17b8061df7ea4cc2399df11\
    --password ./password\
    --nodiscover\
    --maxpeers '50'\
    --networkid 378\
    --targetgaslimit 471238800\
    &
```

## 部署合约

按先后顺序如下部署

1. [p3d合约](https://github.com/ChungkueiBlock/sols/blob/master/fomo3d/Hourglass.sol)

真心不推荐部署带有p3d合约的游戏，这样项目方就可以吃掉本来要流到这里25%左右的流水资金了

我对p3d的合约内容还没有很深的研究，只知道它
 - 是一个自带“交易所”、发行总量为0的Token，
 - 通过Eth买入会自动增发，卖出会销毁
 - 买入和卖出都会扣掉10%的费用给仍持有Token的人
 - 每买一次都会使Token升值
 - 每卖一次会使Token降价

这个合约不需要改动，贴源码，编译后部署截图如下，点击红色记录下来部署后的合约地址

![p3d](/p3d部署.jpg)

1. 部署divies合约

这个合约专门往p3d持有者发分红的。

把刚才记录的p3d合约地址，替换到`HourglassInterface`后面的地址。如上贴源码，编译后部署`Divies`合约，

记录下divies的地址，并替换fomo3dlong.sol里的`DiviesInterface`地址

3. ~~部署JIincForwarder合约~~

这个合约是管理流向社区2%的资金的，被fomo3dlong里调用，
这里需要hack，因为其中涉及到一个闭源的合约，既然知道它是管理2%资金流向的，那直接在fomo3dLong的合约如下hack

- 把定义`Jekyll_Island_Inc`的地方，直接定义成一个普通地址 `address reward = 0xxxxxxx;`
- 把调用Jekyll_Island_Inc的地方， 写成`reward.transfer(_com);`， 注意有两个地方调用（都要换），一个是游戏进行时调用，一个是本轮结束后调用

```javascript
        // // community rewards
        // if (!address(Jekyll_Island_Inc).call.value(_com)(bytes4(keccak256("deposit()"))))
        // {
        //     // This ensures Team Just cannot influence the outcome of FoMo3D with
        //     // bank migrations by breaking outgoing transactions.
        //     // Something we would never do. But that's not the point.
        //     // We spent 2000$ in eth re-deploying just to patch this, we hold the 
        //     // highest belief that everything we create should be trustless.
        //     // Team JUST, The name you shouldn't have to trust.
        //     _p3d = _p3d.add(_com);
        //     _com = 0;
        // }

        reward.transfer(_com);
```

所以不需要部署这个合约，你只要想办法把流到这里的ETH，流到平台方就可以了。（流到开发者，我觉得也是可以的，哈哈～）

4. 部署Team合约

这个合约利用多签技术限制了影响团队的操作，需要改的地方就是把这些地址全部换成自己的，

把下面这些地址，改成你自己的地址，最好把`deployer`地址写成你用来部署合约的那个地址，后面调用playbook合约的`addGame`需要这里的权限

```javascript
address inventor = 0x18E90Fc6F70344f53EBd4f6070bf6Aa23e2D748C;
address mantso   = 0x8b4DA1827932D71759687f925D17F81Fc94e3A9D;
address justo    = 0x8e0d985f3Ec1857BEc39B76aAabDEa6B31B67d53;
address sumpunk  = 0x7ac74Fcc1a71b106F12c55ee8F802C9F672Ce40C;
address deployer = 0xF39e044e1AB204460e06E87c6dca2c6319fC69E3;

admins_[inventor] = Admin(true, true, "inventor");
admins_[mantso]   = Admin(true, true, "mantso");
admins_[justo]    = Admin(true, true, "justo");
admins_[sumpunk]  = Admin(true, true, "sumpunk");
admins_[deployer] = Admin(true, true, "deployer");
```

改完后，如上贴源码，编译后部署`TeamJust`合约，记录地址，替换playbook合约的`TeamJustInterface`地址

5. 部署playerBook合约

很有意思的合约，这里就是上面说的整个游戏其实只需要两个合约中的一个。
不解读细节了，直接改吧

你会发现这里怎么还有个`JIincForwarderInterface`地址，第三步不是说不部署这个了么 ？

这里的主要是收取别人注册名字开启邀请返佣机制时需要支付的那0.01ETH的

知道了这个，就跟第3步一样加个reward收款地址吧，细节不标

如上贴源码，编译后部署`PlayBook`合约，记录下地址， 替换fomo3d合约里的`PlayerBookInterface`地址。


6. 部署fomo3dLong合约

这个是另一个核心合约之一，这里也有个闭源合约，用来初始化控制时间的参数

直接注释掉，然后如下改动

```javascript
uint256 private rndExtra_ = 30;     // 和rndInit一起控制第一轮游戏开始的初始时间的，单位是秒
uint256 private rndGap_ = 30;         // 和rndInit一起控制下轮游戏开始的初始时间的，单位是秒
```

还有两个改动点就是`activate`和`setOtherFomo`里加上自己的deployer地址， 

额外把setOtherFomo里的往另一个游戏池子里输血的功能改到，因为我们没有其他的游戏，如第三步一样，换个收款码吧

```javascript
otherF3D_.transfer(_long);
```

部署吧！！！

最后部署这一步，很有可能遇到`errored: oversized data`的错误，刷新remix页面即可。


## 合约设置

先setOtherFomo,然后再设置playbook里的addgame，最后activate即可。


## 页面

页面直接Ctrl+s下载原版界面，把最后的fomo3dLong的合约地址替换下，另外那个后台API，其实没什么，自己试下就知道了，然后就可以上线了。。。

![fomo3d上线](/fomo3d上线.png)

# 感受

- 合约debug难如上青天
- 要替换一大堆合约里的地址，除了Interface类的要替成依赖的合约地址，其他的全可以写成你自己地址（即使都一样的也OK）就可以。

目前我们搞出来的定制版有：
 - 多级返佣模式， 可自定义级数
 - 空投fix版
 - 去除战队版
 - 移除p3d版本
 - 原版