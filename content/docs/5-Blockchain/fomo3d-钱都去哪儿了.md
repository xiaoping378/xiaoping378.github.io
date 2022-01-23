---
title: "fomo3d-钱都去哪儿了"
linkTitle: "fomo3d-钱都去哪儿了"
tags: ["区块链"]
weight: 5
date: 2017-01-05
description: >
  分析太坊私合约fomo3d的数据流向
---

fomo3d里有战队系统、邀请分佣机制、持key分红、空投系统、持p3d分红等玩法, 相信通过之前各类媒体的解读都有所了解。

下面通过分析合约代码，以讲解ETH数据流向的方式串下所有流程，让大家明明白白的知道自己的ETH都去了哪里。

以10ETH充币到fomod3d合约举例，分三种情况

- 早期用户（游戏刚启动时的激进者）
- 中期用户（为了赚分红、返佣的用户）
- 晚期用户（为了赢48%大奖的人）

## 早期
当合约被激活后，开发者做了一个很“仇富”的举动，每个地址在合约收到100ETH之前，只能购买1ETH的keys，防止被资本大鳄收割本轮后面入场的玩家。这里有个小hack的点，就是提前多准备些小号，多个地址去投，也可以做到比别人便宜多的价格买到keys。

这个阶段以买入10ETH举例，你只会买到等同于1ETH价值的keys，其余9个ETH会直接进入你的收益里，
演示如下：

![ethLimiter2](/ethLimiter2.png)

下面是实现此功能的代码

![ethLimiter](/ethLimiter.png)

代码里的规则(不限阶段)梳理：
- 提款功能可以无限次提，不影响本轮接下来的分红收益，你的收益来自于你持有keys的分红。
- 最低可以支付1e-09个Ether，当购买的Key数量大于或者等于1个时，倒计时会加30秒。
- 当支付的eth不小于0.1时，会送一次“彩票”，买key支付的金额越大，中奖的奖金也越大，最大可中“彩票池”里额度的75%，直译过来这个功能叫空投。

## 中期

所有阶段的用户如果是直接打开的官网，充币买keys时会触发合约的这个接口， 

![buyXaddr](/buyXaddr.png)

其中_affcode是值邀请人的地址，_team是指用户所有购买key所选的战队，默认的2是指蛇队。

如果是从别人的邀请进入的官网，要看邀请人给你发的是哪个链接，有三种形式的链接：

![affiliate](/affiliate.png), 

从上到下，分别会走`buyXaddr`、`buyXid`、`buyXname`的接口，比如我给人发了[exitscam.me/xxp](http://exitscam.me/xxp)的邀请链接，被邀的人买keys时会触发如下接口：

![affiliate2](/affiliate2.png)

这其中我个人会收到他买key总额度的10%佣金，这里还有个隐藏的点：

    如果用户是直接从官网进入买key的，那同样会有10%佣金的产生，只不过是流向p3d的持有者。


## 晚期

当有人买key时，都会选择一个战队，默认会被勾选蛇队的，当买到keys数量不小于1个时，会使所选战队成为本轮的潜在获胜队。

说了这么多废话，回归正体，你的10ETH到底去了哪里？？？

如果支付10ETH时，选的是蛇队，你10个ETH里的5.6个会被持keys的人均分，1个看情况是给p3d的人还是给邀请你的人，还有1个必定会分给持有p3d的人，另外2个会进入大池子，其中0.2个会分给社区贡献人，0.1个会给TeamJust的另一个游戏合约，还有0.1个会流到“彩票池”里。 

这里面根据你选的战队不通，分配比例不一样，具体看下的代码，执行这些ETH分配的是走`distributeExternal`，`distributeInternal` 出去的。

后面的PotSpit是本轮游戏结束后，如何分配大池子里的金额。

```javascript
// Team allocation structures
// 0 = whales
// 1 = bears
// 2 = sneks
// 3 = bulls

// Team allocation percentages
// (F3D, P3D) + (Pot , Referrals, Community)
// Referrals / Community rewards are mathematically designed to come from the winner's share of the pot.
fees_[0] = F3Ddatasets.TeamFee(30,6);   //50% to pot, 10% to aff, 2% to com, 1% to pot swap, 1% to air drop pot
fees_[1] = F3Ddatasets.TeamFee(43,0);   //43% to pot, 10% to aff, 2% to com, 1% to pot swap, 1% to air drop pot
fees_[2] = F3Ddatasets.TeamFee(56,10);  //20% to pot, 10% to aff, 2% to com, 1% to pot swap, 1% to air drop pot
fees_[3] = F3Ddatasets.TeamFee(43,8);   //35% to pot, 10% to aff, 2% to com, 1% to pot swap, 1% to air drop pot

// how to split up the final pot based on which team was picked
// (F3D, P3D)
potSplit_[0] = F3Ddatasets.PotSplit(15,10);  //48% to winner, 25% to next round, 2% to com
potSplit_[1] = F3Ddatasets.PotSplit(25,0);   //48% to winner, 25% to next round, 2% to com
potSplit_[2] = F3Ddatasets.PotSplit(20,20);  //48% to winner, 10% to next round, 2% to com
potSplit_[3] = F3Ddatasets.PotSplit(30,10);  //48% to winner, 10% to next round, 2% to com
```

还有很多细节要分享，碍于时间有限，不过我会持续更新这里的


## 感想

- 持有p3d的人和早期进入的才是最大的受益者
- 后期进入的人只有通过拉人赚佣金的方式回本了
- 这轮游戏应该是结束不了的： 总有人赔了，要拉人进来捞本，被拉的人周而复始。。。
- 结束只有两个可能： 1. 合约有重大漏洞，资金被盗 2. 当大池子里48%的收益足以对整个以太网络发动51%攻击。。。
- 矿工在背后偷着乐
- 你们谁知道TeamJust的下个游戏的合约地址么？ 我知道！！！

如果你也找到了，可以加我微信`yiyemeishui`， 加好友时请输入TeamJust的下个游戏合约地址，我们一起来票大的。。。

