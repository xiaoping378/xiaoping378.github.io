---
tags: ["容器", "K8S", "Openshift"]
title: "项目开发实战"
linkTitle: "项目开发实战"
weight: 3
description: >
  介绍openshift的项目开发实战。 
---

下面的所有操作，都可以通过cli，web console，RestFul API实现，默认使用cli说明

### 创建项目

这里是接着oc cluster up后，来说的， 默认`oc whoami`是 developer,拥有admin的Role角色，俗称项目经理（管理员）

1. 删除默认创建的项目，并创建一个实际中的项目

  ```shell
  oc delete project myproject
  oc new-project eshop --display-name="电商项目" --description="一个神奇的网站"
  ```
  现在项目管理员可以创建任意多个项目，从前面的源码可以看到目前是没法针对项目管理员去限制可创建项目上限的。

2. 查看项目状态

  ```
  #oc status
  In project 电商项目 (eshop) on server https://192.168.31.49:8443

  You have no services, deployment configs, or build configs.
  Run 'oc new-app' to create an application.
  ```
  空空如也，有提示语句提示可通过``oc new-app``去创建具体应用的

### 创建应用

前面也说过，openshift的核心就是围绕应用的整个生命周期来的，所以从new-app说起

new-app的入口是``NewCmdNewApplication()``, 大部分实现是 ``func (c *AppConfig) Run() (*AppResult, error)`` 感兴趣的可以根据源码来理解openshift的devops理念。

1. 创建应用的方式
  现在可以通过3种方式（源码， docker镜像， 模板）来创建一个应用。

  ```
  # oc new-app -h
  #此处省略。。。
  Usage:
    oc new-app (IMAGE | IMAGESTREAM | TEMPLATE | PATH | URL ...) [options]
  #此处省略。。。

  ```
  有很多灵活简便的方式来创建应用，甚至可以直接``oc new-app mysql``来创建一个mysql服务

  比如下面的例子，是基于nodejs-ex项目的master分支，创建应用
  ```
  oc new-app https://github.com/xiaoping378/nodejs-ex.git#master
  ```

  接着上面的nodejs-ex项目来说， 实际上，``oc new-app``就做了两件事，先build， 再deploy。

  new-app一般会先创建一个bc, bc会产出一个iamge，new-app典型的还会创建一个dc，去部署新生成的image，也会创建相应的service来负载均衡方访问刚部署上的镜像里的业务。

  这一切都是自动完成的，因为openshift origin里面有一些检测机制和默认规则，下面就针对上面那条命令看看内部都发生了什么

  * 首先openshift会执行 ``git ls-remote``， 来查看此项目的所有remote分支，

    如果存在master分支，下一步则直接clone和checkout了

    checkout后，接着就是根据解析规则来定义如何build了。

  * build策略

    首先会探测nodejs-ex项目根目录下，是否有dockerfile或者jenkinsfile，如果两者都没有则会根据“典型文件”判断这个项目的开发语言， 举例

    如果存在app.json或者package.json文件，则认为是nodejs类型的项目， 更多的典型文件如下：

    ![detector](/new-app-detector.png)

    这部分的代码实现主要在 detector.go

  未完，待续。。。
