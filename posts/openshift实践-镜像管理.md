# openshift实践-镜像管理

刚接触docker时，第一个接触到的应该就是镜像了，docker之所以如此火热，个人认为一大部分原因就是这个镜像的提出，极大的促进了DevOps和软件复用的能力。

而openshift对镜像的管理非常强大，直到写这篇blog，我才真正意识到这点，甚至犹豫是不是要放到开发实战篇后再来写``镜像管理``。

利用镜像的方式：

- 首先openshift可以利用任何实现了``Docker registry API``的镜像仓，比如，Vmware的Harbor项目，Docker hub以及集成镜像仓（ integrated registry）
- 集成镜像仓， openshift内部的，可以动态生成，自动让用户编译的镜像有地方存， 其次它还负责通知openshift镜像的变动，然后openshift会根据策略去决定编译其他依赖镜像还是部署应用
- 第三方镜像， 可通过命令``oc import-image <stream>``来实时获取镜像tag信息并转换成镜像流，继而触发后续的编译或者部署。
- 当然也支持直接从第三方镜像仓里pull，push镜像

下面先从安装镜像仓开始，然后介绍image Streams 和 istag的概念和应用场景。

## 安装独立镜像仓

可以通过openshift-ansible一次性安装OK，这里使用CLI安装，正好可以串一下openshift里的各种概念的使用场景。

### 部署
```
oc project default
oc adm policy add-scc-to-user privileged system:serviceaccount:default:registry
oc label node 192.168.56.102 registry=true
#使用hostPath存储,要在node上放行权限，默认registry用的1001 userID， 不然后续挂载进去的volume，没有写权限
sudo chown 1001:root /home/registry
#注意要指定使用的镜像版本，默认是拉取最新的， 一定要指明``--volume``参数，不然deploy，不会挂载主机目录，官文这里遗漏了。
oc adm registry --images="openshift/origin-docker-registry:v1.4.1" --selector="registry=true" --mount-host="/home/registry" --service-account=registry --volume='/registry'```

中间莫名出现部署``error``的状态，重新部署了下``oc deploy docker-registry --retry``

### 加密镜像仓

- 先拿到serviceIP
```bash
[root@node0 master]# oc get svc docker-registry
NAME              CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
docker-registry   170.16.132.252   <none>        5000/TCP   1h
```

- 创建自签名证书,如果已经有了，跳过此步
```
oc adm ca create-server-cert \
    --signer-cert=/etc/origin/master/ca.crt \
    --signer-key=/etc/origin/master/ca.key \
    --signer-serial=/etc/origin/master/ca.serial.txt \
    --hostnames='hub.example.com,docker-registry.default.svc.cluster.local,170.16.132.252:5000' \
    --cert=/etc/secrets/registry.crt \
    --key=/etc/secrets/registry.key
```

- 创建secret， secret是专门来保存敏感信息的，比如密码，sshkey，token信息等等

  更详细的介绍可以查看[这里](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/secrets.md)
  ```
  oc secrets new registry-secret \
      /etc/secrets/registry.crt \
      /etc/secrets/registry.key
  ```

- 绑定secret到serviceaccount
```
oc secrets link registry registry-secret
oc secrets link default  registry-secret
```

- 更新dc，添加volume，把新创建的secret挂进去
```
oc volume dc/docker-registry --add --type=secret \
    --name=docker-registry --secret-name=registry-secret -m /etc/secrets
```
如果想移除的话，如下
```
oc volume dc/docker-registry --remove --name=docker-registry
```

- 更新环境变量
```
oc set env dc/docker-registry \
    REGISTRY_HTTP_TLS_CERTIFICATE=/etc/secrets/registry.crt \
    REGISTRY_HTTP_TLS_KEY=/etc/secrets/registry.key
```

- 更新健康监测 HTTP->HTTPS
```
oc patch dc/docker-registry -p '{"spec": {"template": {"spec": {"containers":[{
    "name":"registry",
    "livenessProbe":  {"httpGet": {"scheme":"HTTPS"}}
  }]}}}}'
```
```
oc patch dc/docker-registry -p '{"spec": {"template": {"spec": {"containers":[{
      "name":"registry",
      "readinessProbe":  {"httpGet": {"scheme":"HTTPS"}}
    }]}}}}'
```

- 验证是否OK
```
[root@node0 master]# oc logs dc/docker-registry  | grep tls
time="2017-03-02T16:01:41.113619323Z" level=info msg="listening on :5000, tls" go.version=go1.7.4 instance.id=594aa09b-4540-4e38-a85a-851261cd1254
```

### 添加路由

```
oc create route passthrough registry --service=docker-registry --hostname=hub2.300.cn
```

### 登录镜像仓

```
oc policy add-role-to-user admin developer -n default
oc login -u developer
docker login -u developer -p `oc whoami -t` hub2.300.cn
```

- push镜像

```
docker push hub2.300.cn/default/busybox
```

### 安装镜像仓console

- oc利用官方模板安装
```
oc create -n default -f https://raw.githubusercontent.com/openshift/openshift-ansible/master/roles/openshift_hosted_templates/files/v1.4/origin/registry-console.yaml
oc create route passthrough --service registry-console \
    --hostname hub3.300.cn \
    -n default
oc new-app -n default --template=registry-console \
    -p OPENSHIFT_OAUTH_PROVIDER_URL="https://192.168.31.100:8443" \
    -p REGISTRY_HOST=$(oc get route docker-registry -n default --template='{{ .spec.host }}') \
    -p COCKPIT_KUBE_URL=$(oc get route registry-console -n default --template='https://{{ .spec.host }}')
```

- 登录浏览器打开``https://hub3.300.cn/registry``,使用已有的账户登录，比如这里是默认的developer和developer。
![界面](/assets/openshift-registry-console.png)


## 镜像管理

- openshift基于docker的image概念又延伸出了Image Streams和Image Stream Tags概念

  默认openshift项目下有一些默认集成的is 和 istag
  ```
  oc get is -n openshift
  oc get istag -n openshift
  ```

  image，通俗讲就是对应用运行依赖（库，配置，运行环境）的一个打包。``docker pull push``， 就是操作的镜像。
  为什么openshift还要抽象出is和istag呢，主要是为了打通集成编译和部署环节（bc和dc），原生API就支持了DevOps理念。后面会细讲bc和dc

  is,开发人员可以理解成git的分支，每个分支都会编译很多临时版本出来，这个就是对应到is～=分支和istag～=版本号。
  其实is和istag只是记录了一些映射关系，并不会存放实际镜像数据，比如is里记录了build后要output的镜像仓地址和所有tags，而istag里又记录了具体某个tag与image（可能是存于外部镜像仓，也能是某个is）的关系， 利用此实现了bc/dc和镜像的解耦。

  这里通过部署jenkins服务，来初步了解下具体的含义,
  - 创建ci项目
  ```
  oc new-project ci
  # 先拉取必要镜像
  docker pull openshift/jenkins-1-centos7
  #通过模板部署，其实通过下面一条命令就可以创建一个临时的jenkins服务的
  #oc new-app jenkins-ephemeral
  #跑之前我们先来注意几点
  ```

  - 更改默认的is

  先来查看默认的is
  ```
  oc get template jenkins-ephemeral -n openshift -o json
  {
      "name": "NAMESPACE",
      "displayName": "Jenkins ImageStream Namespace",
      "description": "The OpenShift Namespace where the Jenkins ImageStream resides.",
      "value": "openshift"
  },
  {
      "name": "JENKINS_IMAGE_STREAM_TAG",
      "displayName": "Jenkins ImageStreamTag",
      "description": "Name of the ImageStreamTag to be used for the Jenkins image.",
      "value": "jenkins:latest"
  }
  ```

  可以看到默认模板里会从openshfit的namespace里拉取jenkins:latest的镜像, 去openshift里找找看，确实有
  ```
  ➜  ~ oc get is -n openshift | grep jenkins
  jenkins      170.16.131.234:5000/openshift/jenkins      latest,1                     2 days ago
  ➜  ~ oc get istag -n openshift | grep jenkins
  jenkins:latest      openshift/jenkins-1-centos7@sha256:ab590529e20470e53c1d4b6b970de5d4fd357d864320735a75c1df0e2fffde07       2 days ago   sha256:ab590529e20470e53c1d4b6b970de5d4fd357d864320735a75c1df0e2fffde07
  jenkins:1           openshift/jenkins-1-centos7@sha256:ab590529e20470e53c1d4b6b970de5d4fd357d864320735a75c1df0e2fffde07       2 days ago   sha256:ab590529e20470e53c1d4b6b970de5d4fd357d864320735a75c1df0e2fffde07

  ```

  上面的命令的输出，有两个点要阐述下， is里的``170.16.131.234:5000/openshift/jenkins``是个可有可无的地址， 而istag里``openshift/jenkins-1-centos7@sha256:ab``则是指明了对应tag的镜像来源，

  这样的话，默认执行``oc new-app jenkins-ephemeral``的话，会从docker.io那里拉取镜openshift/jenkins-1-centos7@sha256:ab590529e20470e53c1d4b6b970de5d4fd357d864320735a75c1df0e2fffde07

  为了加速部署，我们把刚才pull的镜像，push到集成镜像仓里
  ```
  #添加用户
  #htpasswd /etc/origin/master/htpasswd xxp
  docker tag openshift/jenkins-1-centos7 hub2.300.cn/openshift/jenkins
  docker login -u developer -p `oc whoami -t` hub2.300.cn
  docker push hub2.300.cn/openshift/jenkins
  ```

  push完后再来看istag的变化，由以前的``openshift/jenkins-1-centos``变为了``170.16.132.252:5000/openshift/jenkins``
  ```
  ➜  ~ oc get istag -n openshift | grep jenkins:latest
  jenkins:latest      170.16.132.252:5000/openshift/jenkins@sha256:dc0f434a492d11d6ae13711e77f87303a06a8fc0fb3a97ae327a4b88c33435b6   21 hours ago   sha256:dc0f434a492d11d6ae13711e77f87303a06a8fc0fb3a97ae327a4b88c33435b6

  ```
  这里是push到integrated registry后，系统自动更新了is和istag， 其实更新或变更istag还可以通过import-image来完成

  这样再来部署就快速多了

  未完待续。。。
