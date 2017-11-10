# 以太坊 Truffle开发实战

整个过程主要演示chrome扩展 METAMASK， OpenZepplin库和truffle框架的使用。

## 搭建私连网络

主要参考之前的[以太坊-私有链搭建初步实践](./以太坊-私有链搭建初步实践.md)， 这里只用单节点的网络。

还是先准备账户：

```bash
mkdir node0
# 会在node0/keystore目录里生成一个keyfile json文件
geth --datadir node0 account new

#利用puppeth生成genesis.json的过程不表，参考上边的链接
geth --datadir node0 init genesis.json

# 把刚才的账号的密码写入node0/password文件
# 启动私链，顺便开启console
echo node0 > node0/password
geth --datadir node0 --port 30000 --nodiscover --unlock '0' --password ./node0/password --mine --rpc --rpccorsdomain "*" --rpcapi "eth,net,web3,admin,personal" console
```

我们把这个账号的json文件导入到chorme插件metamask里，便于后面调试和演示

![](/assets/ethereum-new-account.png)

ubuntu系统上的chrome插件会有窗口消失的bug，在URL栏里打开`chrome-extension://nkbihfbeogaeaoehlefnkodbefgpgknn/popup.html`


## truflle初始化项目

需要下载truffle命令号

```bash
npm install -g truffle

mkdir token && cd token

# 利用trulle下载token代笔示例
truffle unbox tutorialtoken

npm intall zeppelin-solidity
```

如上必要的依赖框架和库已经下载到了本地， 接下来就创建自己的代币合约

在contract目录创建TutorialToken.sol文件，内容如下：

```js
pragma solidity ^0.4.11;


import 'zeppelin-solidity/contracts/token/StandardToken.sol';


/**
 * @title SimpleToken
 * @dev Very simple ERC20 Token example, where all tokens are pre-assigned to the creator. 
 * Note they can later distribute these tokens as they wish using `transfer` and other
 * `StandardToken` functions.
 */
contract TutorialToken is StandardToken {

  string public name = "TutorialToken";
  string public symbol = "SIM";
  uint256 public decimals = 18;
  uint256 public INITIAL_SUPPLY = 10000;

  /**
   * @dev Contructor that gives msg.sender all of existing tokens. 
   */
  function TutorialToken() {
    totalSupply = INITIAL_SUPPLY;
    balances[msg.sender] = INITIAL_SUPPLY;
  }

}
```

在以太坊里几乎所有操作都是当做交易来看的，部署合约就是一种交易，交易就要花钱(gas消耗)，所以truffle做的是增量部署（少消耗gas），现在在`migrations`目录添加新的部署文件`2_deploy_contracts.js`。

```js
var TutorialToken = artifacts.require("./TutorialToken.sol");

module.exports = function(deployer) {
  deployer.deploy(TutorialToken);
};
```

一切准备就绪，编译，部署开始：

```bash
# 编译
➜  truffle compile
Compiling ./contracts/Migrations.sol...
Compiling ./contracts/TutorialToken.sol...
Compiling zeppelin-solidity/contracts/math/SafeMath.sol...
Compiling zeppelin-solidity/contracts/token/BasicToken.sol...
Compiling zeppelin-solidity/contracts/token/ERC20.sol...
Compiling zeppelin-solidity/contracts/token/ERC20Basic.sol...
Compiling zeppelin-solidity/contracts/token/StandardToken.sol...
Writing artifacts to ./build/contracts
# 根据truffle.js的配置进行部署
➜  truffle migrate
Using network 'development'.

Running migration: 1_initial_migration.js
  Deploying Migrations...
  ... 0x65ccd2d6a4f4248466dd7887da7a2ac35d18c7ab0ec826cb25580bc785a2c3b8
  Migrations: 0xc64569558f90302f4b3884929ac5540c645674dc
Saving successful migration to network...
  ... 0xf9043ca886d352f05a05642047f63eed11d9b328fb815becc68baffc4d953d60
Saving artifacts...
Running migration: 2_deploy_contracts.js
  Deploying TutorialToken...
  ... 0x19350625474c36316046b103e671eaad45834a60c17a5b9c64cf96316754560f
  TutorialToken: 0x7f469dc1ec17c3b7c52a3ad74611cb4b7e6807e1
Saving successful migration to network...
  ... 0xe57ba56dd5f1b18d410577def8bc7089f7de56e8d8718c3098430995d4b81353
Saving artifacts...

```