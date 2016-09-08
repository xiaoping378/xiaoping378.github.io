# Deploy kubernetes on centos

#### 一. 先介绍最省事的部署方法，直接从官网下release版本安装:

git clone 代码步骤省略 ...

1. 下载各依赖的release版本

  通过修改配置文件 **cluster/centos/config-build.sh**， 可自定义（k8s, docker, flannel, etcd）各自的下载地址和版本， 不同的版本的依赖可能会需要小改下脚本（版本变更有些打包路径发生了变化，兼容性问题）

  ```
  cd cluster/centos && ./build.sh all
  ```

2. 安装并启动k8s集群环境

  通过修改配置文件 **cluster/centos/config-default.sh**，定义你环境里的设备的IP和其他参数，推荐运行脚本前先通过ssh-copy-id做好免密钥认证；

  ```
  export KUBERNETES_PROVIDER=centos && cluster/kube-up.sh
  ```


#### 二. 源码级编译安装
  本步骤基于上一大步来说,
  先来看下载各依赖的release后，cluster/centos下目录发生了什么变化

  ![](/assets/k8s-binaries-tree.png)

  多了一个binaries的目录，里面是各master和minion上各依赖的二进制文件， 所以我们只要源码编译的结果，替换到这里来， 然后继续上一大步的第2小步即可。

  这里说下，本地编译k8s的话，需要设置安装godep，然后命令本地化。
  ```
  export PATH=$PATH:$GOPATH/bin
  ```

  最后只需要去源码根目录下执行， 编译结果在_output目录下
  ```
  make
  ```

  替换到相应的binaries目录下，重新运行kube-up.sh即可。
