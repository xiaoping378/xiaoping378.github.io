# DEIS 开源PAAS平台实践总结

DEIS（目前已被微软收购）的workflow是开源的Paas平台，基于kubernetes做了一层面向开发者的CLI和接口，做到了让开发者对容器无感知的情况下快速的开发和部署线上应用。

> workflow是 on top of k8s的，所有组件默认全是跑在pod里的，不像openshift那样对k8s的侵入性很大。

特性如下：
  - S2I(自动识别源码直接编译成镜像)
  - 日志聚合
  - 应用管理（发布，回滚）
  - 认证&授权机制
  - 边界路由

![Workflow_Detail](/assets/Workflow_Detail.png)

下面从环境搭建，安装workflow及其基本使用做个梳理。

### 初始化k8s集群

  可以通过[k8s-deploy](https://github.com/xiaoping378/k8s-deploy)项目来离线安装高可用kubernetes集群，我这里是单机演示环境。

    kubeadm init --kubernetes-version v1.6.2 --pod-network-cidr 12.240.0.0/12
    #方便命令自动补全
    source <(kubectl completion zsh)

    #安装cni网络
    cp /etc/kubernetes/admin.conf $HOME/.kube/config
    kubectl apply -f kube-flannel-rbac.yml
    kubectl apply -f kube-flannel.yml

    #使能master可以被调度
    kubectl taint node --all  node-role.kubernetes.io/master-

    #安装ingress-controller, 边界路由作用
    kubectl create -f ingress-traefik-rbac.yml
    kubectl create -f ingress-traefik-deploy.yml

### 初始化helm

  helm相当于kubernetes里的包管理器，类似yum和apt的作用，只不过它操作的是charts（各种k8s yaml文件的集合，额外还有Chart.yaml -- 包的描述文件）可以理解为基于k8s的应用模板管理类工具， 后面会用它来安装workflow到上面跑起来的k8s集群里。

  从k8s 1.6之后，kubeadm安装的集群，默认会开启RBAC机制，为了让helm可以安装任何应用，我们这里赋予tiller cluster-admin权限

    kubectl create serviceaccount helm --namespace kube-system
    kubectl create clusterrolebinding cluster-admin-helm --clusterrole=cluster-admin --serviceaccount=kube-system:helm

  初始化helm：

    ➜  helm init --service-account helm
    $HELM_HOME has been configured at /home/xxp/.helm.

    Tiller (the helm server side component) has been installed into your Kubernetes Cluster.
    Happy Helming!

    ➜  helm version
    Client: &version.Version{SemVer:"v2.4.1", GitCommit:"46d9ea82e2c925186e1fc620a8320ce1314cbb02", GitTreeState:"clean"}
    Server: &version.Version{SemVer:"v2.4.1", GitCommit:"46d9ea82e2c925186e1fc620a8320ce1314cbb02", GitTreeState:"clean"}


  安装后，默认导入了2个repos，后面安装和搜索应用时，都是从这2个仓里出的，当然也可以自己通过`helm repo add`添加本地私有仓

    ➜  helm repo list
    NAME  	URL                                             
    stable	https://kubernetes-charts.storage.googleapis.com
    local 	http://127.0.0.1:8879/charts                    

  helm的使用基本流程如下:

  - helm search:    搜索自己想要安装的应用（chart）
  - helm fetch:     下载应用（chart）到本地，可以忽略此步
  - helm install:   安装应用
  - helm list:      查看已安装的应用情况

### 安装workflow

  添加workflow的repo仓

    helm repo add deis https://charts.deis.com/workflow

  开始安装workflow，因为RBAC的原因，同样要赋予workflow各组件相应的权限，yml文件在[这里]（https://gist.github.com/xiaoping378/798c39e0b607be4130db655f4873bd24）

    kubectl apply -f workflow-rbac.yml --namespace deis

    helm install deis/workflow --name workflow --namespace deis \
      --set global.experimental_native_ingress=true,controller.platform_domain=192.168.31.49.xip.io

  其中会拉取所需镜像，不出意外会有如下结果：

    ➜  kubectl --namespace=deis get pods
    NAME                                     READY     STATUS    RESTARTS   AGE
    deis-builder-1134410811-11xpp            1/1       Running   0          46m
    deis-controller-2000207379-5wr10         1/1       Running   1          46m
    deis-database-244447703-v2sh9            1/1       Running   0          46m
    deis-logger-2533678197-pzmbs             1/1       Running   2          46m
    deis-logger-fluentd-08hms                1/1       Running   0          42m
    deis-logger-redis-1307646428-fz1kk       1/1       Running   0          46m
    deis-minio-3195500219-tv7wz              1/1       Running   0          46m
    deis-monitor-grafana-59098797-mdqh1      1/1       Running   0          46m
    deis-monitor-influxdb-168332144-24ngs    1/1       Running   0          46m
    deis-monitor-telegraf-vgbr9              1/1       Running   0          41m
    deis-nsqd-1042535208-40fkm               1/1       Running   0          46m
    deis-registry-2249489191-2jz3p           1/1       Running   2          46m
    deis-registry-proxy-qsqc2                1/1       Running   0          46m
    deis-router-3258454730-3rfpq             1/1       Running   0          41m
    deis-workflow-manager-3582051402-m11zn   1/1       Running   0          46m


### 注册管理用户

  由于我们是本地ingress-controller, 必须保障deis-builder.$host可以被解析, 自行创建ingress of deis-builder.

    kubectl apply -f deis-buidler-ingress.yml

  确保traefik有如下状态：

  ![traefik-status](/assets/traefik-status.png)

  如下操作注册，默认第一个用户为管理员用户，可操作所有其他用户。

    ➜  ~ kubectl get --namespace deis ingress
    NAME                                 HOSTS                               ADDRESS   PORTS     AGE
    builder-api-server-ingress-http      deis-builder.192.168.31.49.xip.io             80        18m
    controller-api-server-ingress-http   deis.192.168.31.49.xip.io                     80        1h
    ➜  ~
    ➜  ~ deis register deis.192.168.31.49.xip.io
    username: admin  
    password:
    password (confirm):
    email: xiaoping378@163.com
    Registered admin
    Logged in as admin
    Configuration file written to /home/xxp/.deis/client.json
    ➜  ~
    ➜  ~ deis whoami
    You are admin at http://deis.192.168.31.49.xip.io


### 部署第一个应用
