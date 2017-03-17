# opeshift实践-DevOps实战-0

主要涉及到``一键发布``，``快速回滚``，``弹性伸缩``，``蓝绿部署``方面。


- 启动openshift

      oc cluster up --version=v1.5.0-rc.0 --metrics --use-existing-config=true

    默认负责监控的pods占用资源太大了，可以这样限制下，或者cluster up时不加 ``--metrics``
      oc login -u system:admin
      oc env rc hawkular-cassandra-1 MAX_HEAP_SIZE=1024M -n openshift-infra

      #重建下,变量才会生效
      oc scale rc hawkular-cassandra-1 --replicas 0 -n openshift-infra
      oc scale rc hawkular-cassandra-1 --replicas 1 -n openshift-infra

- 建立本地Git仓

  默认官方给出的例子基本都需要和Github结合，实在不好本地实战演示，所以本地来一个``gogs``。

      oc login -u devloper
      oc new-project ci

      #先拉取所依赖镜像
      docker pull openshiftdemos/gogs:0.9.97
      docker pull centos/postgresql-94-centos7

      #创建gogs服务，并禁用webhook时的TLS校验，不然无法触发build
      oc new-app -f https://raw.githubusercontent.com/xiaoping378/gogs-openshift-docker/master/openshift/gogs-persistent-template.yaml -p SKIP_TLS_VERIFY=true -p HOSTNAME=gogs-ci.192.168.31.49.xip.io

  上面的HOSTNAME，注意要换成自己宿主机的IPv4地址，默认创建的其他服务的路由都是这个形式的，

  有个有意思的地方，为什么默认路由会是这中 ``name+IP+xip.io`` 的形式呢，奥秘在 http://xip.io 的公共服务上。
  它其实是个特殊的域DNS server，有人查询域名``gogs-ci.192.168.31.49.xip.io``时 ，它就会返回192.168.31.49的地址回来，
  而这个地址恰好是我们Router的地址，这样子Router会根据route的定义负责负载到对应的POD上。自己试验下就知道怎么回事了。

      dig http://gogs-ci.192.168.31.49.xip.io +short

  只做功能性演示，先不考虑https加密安全访问，创建完后，访问``http://gogs-ci.192.168.31.49.xip.io``

  ![gogs](/assets/openshift-gogs.png)

  这个项目，第一个注册用户即为管理员，比如我现在注册一个叫``developer``的用户。

- 找个项目来实战吧

    - 克隆远程项目，并设置
          git clone https://github.com/xiaoping378/nodejs-ex.git && cd nodejs-ex
          git remote add gogs http://gogs-ci.192.168.31.49.xip.io/developer/nodejs-ex.git

    - 通过web页面，在gogs上创建一个``nodejs-ex``仓库, 并如下push刚才克隆的项目
          $ git push gogs master

          Username for 'http://gogs-ci.192.168.31.49.xip.io': developer
          Password for 'http://developer@gogs-ci.192.168.31.49.xip.io':
          Counting objects: 431, done.
          Delta compression using up to 4 threads.
          Compressing objects: 100% (210/210), done.
          Writing objects: 100% (431/431), 145.16 KiB | 0 bytes/s, done.
          Total 431 (delta 159), reused 431 (delta 159)
          To http://gogs-ci.192.168.31.49.xip.io/developer/nodejs-ex.git
          * [new branch]      master -> master

      gogs的页面上会如实反馈信息

      ![gogs](/assets/gogs-create-push.png)

      OK，现在本地项目有了，接下来进入正题

    - 在openshift部署此nodejs应用
          #创建nodejs项目
          oc new-project web

          #先拉取依赖镜像
          docker pull centos/mongodb-32-centos7
          docker pull centos/nodejs-4-centos7

          #部署此项目，并启用国内npm源和对应的git仓
          oc new-app nodejs-mongo-persistent --name=nodejs-ex -p NPM_MIRROR=https://registry.npm.taobao.org -p SOURCE_REPOSITORY_URL=http://gogs-ci.192.168.31.49.xip.io/developer/nodejs-ex.git

        默认此模板会从指定的URL地址拉取代码，并根据指定配置，编译策略采取Source方式，基于istag nodejs:4镜像编译出来nodejs-mongo-persistent:latest:latest镜像，
        编译出来的镜像又会自动触发部署。


- 测试最基本的DevOps

  即push代码触发自动编译和滚动部署

  要实现这个目标前，需要先把webhook填写到gogs里。

  在openshift界面上复制webhook地址，

  ![find_webhook](/assets/openshift-github-webhook.png)

  然后在gogs上填加一个webhook

  ![add_webhook](/assets/openshift-gogs-webhook.png)


  这样我们随意修改些，然后推送代码，就会自动出发编译部署

      ➜  nodejs-ex git:(master) vim views/index.html        
      ➜  nodejs-ex git:(master) ✗ git add .
      ➜  nodejs-ex git:(master) ✗ git commit -m "这又是个测试"
      [master 082f05e] 这又是个测试
      1 file changed, 1 insertion(+), 1 deletion(-)
      ➜  nodejs-ex git:(master) git push gogs master
      Username for 'http://gogs-ci.192.168.31.49.xip.io': developer
      Password for 'http://developer@gogs-ci.192.168.31.49.xip.io':
      Counting objects: 4, done.
      Delta compression using up to 4 threads.
      Compressing objects: 100% (3/3), done.
      Writing objects: 100% (4/4), 365 bytes | 0 bytes/s, done.
      Total 4 (delta 2), reused 0 (delta 0)
      To http://gogs-ci.192.168.31.49.xip.io/developer/nodejs-ex.git
      c3592e6..082f05e  master -> master

      ![auto-deploy](/assets/push-build-deploy.png)

  现实中，如果没有很好的自动化测试的话，我们肯定不会这样操作的，除非想被开掉了。

  可以去掉webhook自动构建，采用手动触发build，界面操作的话，去build界面，点击``Start Build``，命令行如下

      oc start-build nodejs-mongo-persistent

  如果想回滚以前的部署版本的话，也有对应的界面和命令

      oc rollback nodejs-mongo-persistent --to-version=3

      ![rollBack](/assets/openshift-roll-back.png)

- 弹性伸缩

  目前可以根据CPU使用率来进行弹性伸缩

  有人问能不能基本mem进行弹性呢，其实这个是没有意义的，一般应用都会自行缓存，内存基本只增不长， 所以cpu能很好的实时反应业务量的负载。

  弹性伸缩前，要确保应用先行设置了cpu request，设置 cpu request

  部署的菜单栏

  ![dc-menu](/assets/openshift-dc-menu.png)

  添加 cpu request.

  ![cpu-request](/assets/openshift-cpu-request.png)

  也可以通过命令行开启，弹性伸缩特性

      oc autoscale dc/nodejs-mongo-persistent --max=3 --cpu-percent=80

  OK，之后我们通过ab工具简单做个压力模拟，因为环境是我的笔记本上，所以模拟发送100万个连接，并发100吧

      ab -n 1000000 -c 100 http://nodejs-mongo-persistent-web.192.168.31.49.xip.io/

  后台每1分钟采集一次数据，过不了一会儿，就会看到nodejs实例自动扩展到了3个，

  ![autoscale-0](/assets/openshift-autoscale-0.png)

  当业务量降下来时，还可以自动减少实例，是根据平均CPU使用率来操作的。

  ![autoscale-1](/assets/openshift-autoscale-1.png)


- 蓝绿部署

  这个也是API级别的支持，不描述具体操作细节了，原理还是以前的，从负载均衡层面入手。 实现新旧版本同时存在，
  当面并不是所有业务都适合蓝绿部署的，要看后台数据是否允许，多个新旧版本同时产生读写数据

  在openshift里具体就是在Route层面添加同一应用的多个版本的service，并设置分流权重
  截图如下

  界面设置

  ![bluegreen-0](/assets/openshift-blue-green-0.png)

  实际展示效果

  ![bluegreen-1](/assets/openshift-blue-green-1.png)



### 总结

Openshift 平台本身在API层面实现了DevOps，所以基于它很容易做到DevOps as an service， 上面的演示可能若现实世界不太一样，

比如真实情况需要，测试，预发布，线上环境，下次再分享，openshift基于jenkins pipeline如果实现更真实场景的需求。
