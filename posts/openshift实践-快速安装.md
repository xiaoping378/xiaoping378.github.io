# openshift 实践 - 快速安装

不知道为什么openshift在国内热度这么低，那些要做自己容器云的公司，不知道有openshift项目的存在么？完全满足我的需求。

docker负责应用的隔离打包，k8s提供集群管理和容器的编排服务，而openshfit则负责整个应用的生命周期：

* 源码管理，CI&CD能力
* 多租户管理, 支持LDAP和Oauth
* 集成监控日志于web console

先说下自接触到openshift项目就遇到的一个困惑，就是openshift origin/online/ocp之间的关系： ```orgin相当于Fedora， 其他的相当于RHEL```

接下来谈下我用自己的笔记本实践的过程与感受：

1. 快速安装

  本人日常基于ubuntu16.04办公，所以用oc直接上, oc相当于kubectl

  [这里](https://github.com/openshift/origin/releases)直接下载oc客户端，或者自行编译, 编译结果在_output目录下
  ```
  git clone --depth=1 https://github.com/openshift/origin.git
  cd origin && make
  mv _output/local/bin/linux/amd64/oc  /usr/local/bin

  ```
  启动openshift, 默认开启监控并初始安装自最新版本，当前是v1.5.0-alpha.2
  ```
  oc cluster up --metrics=true  --version=latest --insecure-skip-tls-verify=true --public-hostname=air13
  ```

  过程中会拉取所需镜像, 我这里显示比较多，之前已经做了些实验
  ```
  ➜  ~ docker images | grep openshift | awk '{print $1}'
  openshift/node
  openshift/origin-sti-builder
  openshift/origin-docker-builder
  openshift/origin-deployer
  openshift/origin-gitserver
  openshift/origin-docker-registry
  openshift/origin-haproxy-router
  openshift/origin
  openshift/hello-openshift
  openshift/openvswitch
  openshift/origin-pod
  openshift/origin-metrics-cassandra
  openshift/origin-metrics-hawkular-metrics
  openshift/origin-metrics-heapster
  openshift/origin-metrics-deployer
  openshift/mysql-55-centos7
  openshift/origin-logging-curator
  openshift/origin-logging-fluentd
  openshift/origin-logging-deployment
  openshift/origin-logging-elasticsearch
  openshift/origin-logging-kibana
  openshift/origin-logging-auth-proxy
  ```
  启动后，会打印如下信息

  ```
  OpenShift server started.
  The server is accessible via web console at:
      https://air13:8443

  The metrics service is available at:
      https://metrics-openshift-infra.192.168.31.49.xip.io

  You are logged in as:
      User:     developer
      Password: developer

  To login as administrator:
      oc login -u system:admin
  ```

  打开浏览器，访问https://air13:8443，默认用developer登录，其实现在任意用户任意密码都可以的。

  web console里是空空如野的，可以临时授权developer用户操作所有项目
  ```
  oc adm policy add-cluster-role-to-user cluster-admin developer
  ```


2.技巧总结

  * 命令行自动补全, 其实kubectl也可以如此

  ``source <(oc completion bash)``

  * 默认监控占用的资源太大了，可以如下降低资源占用，当然也可以web操作限制资源利用率
  ```
  oc env rc hawkular-cassandra-1 MAX_HEAP_SIZE=1024M -n openshift-infra
  oc delete pod hawkular-cassandra-1-df23x -n openshift-infra
  ```
  因为是rc，所以直接杀掉没关系，要不env不生效

推荐此人[blog](http://guifreelife.com/)，有几篇干货


3.后面会重点说下权限/资源管理和整个app开发的流程
