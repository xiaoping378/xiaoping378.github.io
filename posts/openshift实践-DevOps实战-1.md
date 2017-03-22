# openshift实践-DevOps实战-1

本文主要介绍基于openshift如何完成``开发->测试->线上``场景的变更，这是一个典型的应用生产流程，来看看openshift是如何利用容器优雅的完成整个过程的吧

下文基于上篇[DevOps实战-0](./openshift实践-DevOps实战-0.md) 的``nodejs-ex``项目来说, 假设到这里，你本地已经有了nodejs-ex项目

### 准备3个project

  用这3个project来模拟开发，测试，线上环境

  现实中一般各个场景的服务器都是物理隔离的，这里可以利用``--node-selector``，来指定项目可以跑在哪些节点上。

  ```
  oc login -u sysetm:admin

  #晚上在笔记本上写此blog，没合适的环境，单机模拟多台 -- start
  oc label node 192.168.31.49 web-prod=true web-dev=true web-test=true
  #晚上在笔记本上写此blog，没合适的环境，单机模拟多台 -- end

  #1.创建web-dev项目
  #2.授权developer为开发组项目管理员
  #3.授权测试和运维人员可以从开发组拉取镜像
  oc adm new-project web-dev --node-selector='web-dev=true'
  oc policy add-role-to-user admin developer
  oc policy add-role-to-group system:image-puller system:serviceaccounts:web-test -n web-dev
  oc policy add-role-to-group system:image-puller system:serviceaccounts:web-prod -n web-dev

  oc adm new-project web-test --node-selector='web-test=true'
  oc policy add-role-to-user admin tester

  oc adm new-project web-prod --node-selector='web-prod=true'
  oc policy add-role-to-user admin ops
  ```

  - 你可能会注意到，这里用的`new-project` 前面还加了adm， 其实`oc adm`等效于`oadm`， 一般管理集群相关的用这个命令，这里是因为需要读取节点的标签（label）信息。

  - 指定项目要运行那些节点，则是利用了注解-annotations， 即在原有的project结构上设置了注解，这样openshift在相应的项目里创建任何pod时，都对会自动注入node-selector

  - 另外需要注意的，默认项目的管理员（developer）是没有权限读取node标签信息的，以前写过[权限管理相关blog](./openshift实践-权限资源管理.md)，集群管理员可以授权node访问权限，即使如此developer还是不能改写项目级别的标签的，举个例子: developer在开发环境的pod上指定了``--node-selector='web-dev=false'``， 最终这个pod的node-selector会是`'web-dev=true, web-dev=flase'`, 导致最终不会被调度到任何节点上。

  - 上面分别授权了3个用户，这里是不关心这些用户是否真实存在的，只是一个RABC的描述，因为是`oc cluster up`起来的环境，默认使用`anypassword`的身份认证，所以登录时，任意用户名和密码都是可以登录OK的。

  - 通过`oc describe policybinding -n web-dev` 可以查看授权情况，  如果觉得默认的role不满足需求的话，也可以自定义role，另外通过`oc policy remove-role-from-group/user <Role> <name>`可以移除相关授权，

### 初始化web-dev, web-test, web-prod环境

  按照上篇`DevOps实战-0`里的方式

  初始化我们的开发环境, 进入源码`nodejs-ex`目录

  ```
  oc new-app -f openshift/template/nodejs-mongo-persistent.json --name=nodejs-ex \
    -p NPM_MIRROR=https://registry.npm.taobao.org \
    -p SOURCE_REPOSITORY_URL=http://gogs-ci.192.168.31.49.xip.io/developer/nodejs-ex.git \
    -n web-dev
  ```

  初始化测试环境，相较于上一步的模板json，只是注掉bc和更改了triggers的is,后面会详细介绍之间的差异

  ```
  oc new-app -f openshift/template/nodejs-mongo-persistent-test.json --name=nodejs-ex-test -n web-test
  ```

  以tester登录web console，会发现只有mongodb部署上了，而前端nodejs还在等待依赖的镜像 `web-dev/nodejs-mongo-persistent:test`。

  ![webtest-wait](/assets/openshift-web-test-wait.png)


  初始化生产环境, 这个生产的template.json有点儿简单，负载均衡和弹性伸缩都没有启用。

  ```
  oc new-app -f openshift/template/nodejs-mongo-persistent-prod.json --name=nodejs-ex-prod -n web-prod
  ```

### 实战模拟，开发->测试->发布

  - developer开发完特性或者修复完bug，push代码到镜像仓。

    这里分享一个很方便的技巧，就是 `oc rsync`, 这个可以实时的同步本地目录到容器了，避免了频繁编译镜像和临时挂载目录到镜像里的hack了。

    ```
    vim ...
    git add .
    git commit -m "fix bugs"
    git push gogs master
    ```

    如上，由于上篇中设置了webhook, developer提交代码会触发了自动编译并部署，确认部署后的环境是否修复了bug，如果单元测试通过，那就要通知测试团队（如今大部分公司，应该没有测试人员了吧，也可以直接变更到线上）

    测试那边的环境里一直在等待这个镜像`web-dev/nodejs-mongo-persistent:test`， 而默认developer配置成默认编译出来的是`web-dev/nodejs-mongo-persistent:latest`

    ```
    oc login -u developer
    oc tag web-dev/nodejs-mongo-persistent:latest web-dev/nodejs-mongo-persistent:v1.1
    oc tag web-dev/nodejs-mongo-persistent:v1.1 web-dev/nodejs-mongo-persistent:test
    ```

    如上操作后，开发人员更新版本号，然后在web-dev环境里会打上一个test的镜像tag出来，操作完如下所示

    ```
    ➜  nodejs-ex git:(master) oc get is
    NAME                      DOCKER REPO                                       TAGS               UPDATED
    nodejs-mongo-persistent   172.30.1.1:5000/web-dev/nodejs-mongo-persistent   test,v1.1,latest   49 seconds ago
    ➜  nodejs-ex git:(master)
    ➜  nodejs-ex git:(master) oc get istag
    NAME                             DOCKER REF                                                                                                                UPDATED          IMAGENAME
    nodejs-mongo-persistent:latest   172.30.1.1:5000/web-dev/nodejs-mongo-persistent@sha256:55615da49dd299064e7bba75923ac7996bf0d109e0322d4f84f9b41665b2e4c7   3 minutes ago    sha256:55615da49dd299064e7bba75923ac7996bf0d109e0322d4f84f9b41665b2e4c7
    nodejs-mongo-persistent:v1.1     172.30.1.1:5000/web-dev/nodejs-mongo-persistent@sha256:55615da49dd299064e7bba75923ac7996bf0d109e0322d4f84f9b41665b2e4c7   2 minutes ago    sha256:55615da49dd299064e7bba75923ac7996bf0d109e0322d4f84f9b41665b2e4c7
    nodejs-mongo-persistent:test     172.30.1.1:5000/web-dev/nodejs-mongo-persistent@sha256:55615da49dd299064e7bba75923ac7996bf0d109e0322d4f84f9b41665b2e4c7   55 seconds ago   sha256:55615da49dd299064e7bba75923ac7996bf0d109e0322d4f84f9b41665b2e4c7
    ```

    这样一来，测试环境里就会自动部署上刚才开发人员的环境了，再也不会有因为环境差异问题和测试吵吵了。

    这一切都得益于openshift里新添加的imageStreams，它打通了编译和部署的环节，能自动通知对方，继而自动触发下一步动作。

    测试通过后，通知Ops再重新tag成线上所需要的镜像tag，这样线上就会根据配置自动滚动升级了。

    ```
    #假设一个叫ops的人负责上线,那首先ops得有具备web-dev项目里编辑is的能力
    oc login -u developer
    #不该给ops这么高权限的，应该自定义一个只能tag is的role，这里为了简单演示
    oc policy add-role-to-user edit ops -n web-dev
    ```

    如上操纵，ops就具备了tag web-dev项目的镜像的能力，也可以通过UI来查看和授权

    ![](/assets/openshift-add-role.png)

    ```
    oc login -u ops
    oc tag web-dev/nodejs-mongo-persistent:v1.1 web-dev/nodejs-mongo-persistent:prod
    ```

    然后打上线上依赖的镜像tag即可，发布上线，这样就完成了开发->测试->发布一条线，很快捷的`人工干预`上线了

## 总结

  openshift 利用镜像tag的能力，来实现不同场景的同步，单纯基于docker也可以实现以上目标的，只是不够平台化，还是以前的脚本打天下，远不如openshift在API层面解决来的强大和灵活。
