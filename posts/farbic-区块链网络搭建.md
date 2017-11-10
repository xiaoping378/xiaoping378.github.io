# hyperledger farbic搭建高并发交易网络

针对每秒数千笔交易的场景，默认的CCVC（并发控制版本检查）会导致交易失败率的上升，其实不需要对基础网络本身做特殊设置，从合约代码入手可以解决，参考官方例子[farbic-samples](https://github.com/hyperledger/fabric-samples).

### 下载项目

基于目前最新的v1.0.3版本来说

```bash
git clone https://github.com/hyperledger/fabric-samples.git
cd fabric-samples/first-network
```

