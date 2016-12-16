# 离线安装 kubernetes 1.5

经常遇到全新初始安装k8s集群的问题，所以想着搞成离线模式，本着最小依赖原则，提高安装速度

基于Centos7-1511-minimal,  非此版本脚本应该会运行出错，自行修改吧

本离线安装所有的依赖都打包放到了[百度网盘](https://pan.baidu.com/s/1i5jusip)


## 第一步
基本思路是，在k8s-deploy目录下，临时启个http server， node节点上会从此拉取所依赖镜像和rpms

```
# python -m SimpleHTTPServer
Serving HTTP on 0.0.0.0 port 8000 ...

```

windows上可以用hfs临时启个http server， 自行百度如何使用

## master侧

运行以下命令，初始化master

192.168.56.1:8000 是我的http-server, 注意要将k8s-deploy.sh 里的HTTP-SERVER变量也改下

```
curl -L http://192.168.56.1:8000/k8s-deploy.sh | bash -s master
```

## minion侧

视自己的情况而定

```
curl -L http://192.168.56.1:8000/k8s-deploy.sh |  bash -s join --token=6669b1.81f129bc847154f9 192.168.56.100
```

## 总结

整个脚本实现比较简单， 坑都在脚本里解决了。脚本文件在[这里](https://gist.github.com/xiaoping378/3a129aa6c81eaecae199a50236ad8bf7)

就一个master-up和node-up， 基本一个函数只做一件事，很清晰，可以自己查看具体过程。

1.5 与 1.3给我感觉最大的变化是网络部分， 1.5启用了cni网络插件
不需要像以前一样非要把flannel和docker绑在一起了（先启flannel才能启docker）。

具体可以看这里
https://github.com/containernetworking/cni/blob/master/Documentation/flannel.md

master侧如果是单核的话，会因资源不足， dns安装失败。
