# 容器应用模板介绍


### 概要
- Helm是一个管理kubernetes集群内应用的工具，提供了一系列管理应用的快捷方式，例如 inspect， install， upgrade， delete等，经验可以沿用以前apt，yum，homebrew的,区别就是helm管理的是kubernetes集群内的应用。

- 还有一个概念必须得提，就是`chart`， 它代表的就是被helm管理的应用包，里面具体就是放一些预先配置的Kubernetes资源(pod, rc, deployment, service, ingress)，一个包描述文件(`Chart.yaml`), 还可以通过指定依赖来组织成更复杂的应用，支持go template语法，可参数化模板，让使用者定制化安装
charts可以存放在本地，也可以放在远端，这点理解成yum仓很合适。。。

这里有个[应用市场](https://kubeapps.com) ，里面罗列了各种应用charts。由开源项目[monocular](https://github.com/helm/monocular)支撑

下面主要介绍helm的基本使用流程和具体场景的实践。

### 初始化k8s集群v1.6.2

先来准备k8s环境，可以通过[k8s-deploy](https://github.com/xiaoping378/k8s-deploy)项目来离线安装高可用kubernetes集群，我这里是单机演示环境。

```bash
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
```
这样一个比较完整的k8s环境就具备了，另外监控和日志不在此文的讨论范围内。

### 初始化Helm环境

由于刚才创建的k8s集群默认启用RBAC机制，个人认为这个特性是k8s真正走向成熟的一大标志，废话不表，为了helm可以安装任何应用，我们先给他最高权限。

```
kubectl create serviceaccount helm --namespace kube-system
kubectl create clusterrolebinding cluster-admin-helm --clusterrole=cluster-admin --serviceaccount=kube-system:helm
```

初始化helm，如下执行，会在kube-system namepsace里安装一个tiller服务端，这个服务端就是用来解析helm语义的，后台再转成api-server的API执行：

```bash
➜  helm init --service-account helm
$HELM_HOME has been configured at /home/xxp/.helm.
Tiller (the helm server side component) has been installed into your Kubernetes Cluster.
Happy Helming!

➜  helm version
Client: &version.Version{SemVer:"v2.4.1", GitCommit:"46d9ea82e2c925186e1fc620a8320ce1314cbb02", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.4.1", GitCommit:"46d9ea82e2c925186e1fc620a8320ce1314cbb02", GitTreeState:"clean"}

#命令行补全
➜  source <(helm completion zsh)
```

### 安装第一个应用

初始化Helm后，默认就导入了2个repos，后面安装和搜索应用时，都是从这2个仓里出的，当然也可以自己通过`helm repo add`添加本地私有仓

```bash
➜  helm repo list
NAME  URL
stable	https://kubernetes-charts.storage.googleapis.com
local http://127.0.0.1:8879/charts
```

其实上面的repo仓的索引信息是存放在`~/.helm/repository`的, 类似/etc/yum.repos.d/的作用

helm的使用基本流程如下:

- helm search:    搜索自己想要安装的应用（chart）
- helm fetch:    下载应用（chart）到本地，可以忽略此步
- helm install:  安装应用
- helm ls:        查看已安装的应用情况

这里举例安装redis

```bash
➜  helm install stable/redis --set persistence.enabled=false
```

如上，如果网络给力的话，很快就会装上最新的redis版本，Helm安装应用，目前有四种方式：

- `helm install stable/mariadb`    通过chart仓来安装
- `helm install ./nginx-1.2.3.tgz`  通过本地打包后的压缩chart包来安装
- `helm install ./nginx`            通过本地的chart目录来安装
- `helm install https://example.com/charts/nginx-1.2.3.tgz` 通过绝对网络地址来安装chart压缩包

### 实战演示

主要从`制作自己的chart`， `构建自己的repo`， `组装复杂应用的实战`三方面来演示

#### 制作自己的chart

helm有一个很好的引导教程模板, 如下会自动创建一个通用的应用模板

```bash
➜  helm create myapp
Creating myapp

➜  tree myapp
myapp
├── charts  //此应用包的依赖包定义（如果有的话，也会是类似此包的目录结构）
├── Chart.yaml  // 包的描述文件
├── templates  // 包的主体目录
│   ├── deployment.yaml  // kubernetes里的deployment yaml文件
│   ├── _helpers.tpl  // 模板里如果复杂的话，可能需要函数或者其他数据结构，这里就是定义的地方
│   ├── ingress.yaml // kubernetes里的ingress yaml文件
│   ├── NOTES.txt // 想提供给使用者的一些注意事项，一般提供install后，如何访问之类的信息
│   └── service.yaml // kubernetes里的service yaml文件
└── values.yaml // 参数的默认值

2 directories, 7 files
```

如上操作，我们就有了一个`myapp`的应用，目录结构如上，来看看看values.yaml的内容, 这个里面就是模板里可定制参数的默认值

很容易看到，kubernetes里的rc实例数，镜像名，servie配置，路由ingress配置都可以轻松定制。

```YAML
# Default values for myapp.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
replicaCount: 1
image:
  repository: nginx
  tag: stable
  pullPolicy: IfNotPresent
service:
  name: nginx
  type: ClusterIP
  externalPort: 80
  internalPort: 80
ingress:
  enabled: false
  # Used to create Ingress record (should used with service.type: ClusterIP).
  hosts:
    - chart-example.local
  annotations:
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  tls:
    # Secrets must be manually created in the namespace.
    # - secretName: chart-example-tls
    #   hosts:
    #     - chart-example.local
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

 note.
 >一般拿到一个现有的app chart后，这个文件是必看的，通过`helm fetch myapp`会得到一个类似上面目录的压缩包


我们可以通过 `--set`或传入values.yaml文件来定制化安装，

```bash
# 安装myapp模板： 启动2个实例，并可通过ingress对外提供myapp.192.168.31.49.xip.io的域名访问
➜  helm install --name myapp --set replicaCount=2,ingress.enabled=true,ingress.hosts={myapp.192.168.31.49.xip.io} ./myapp

➜  helm ls
NAME                  	REVISION	UPDATED                 	STATUS  	CHART      	NAMESPACE
exasperated-rottweiler	1       	Wed May 10 13:58:56 2017	DEPLOYED	redis-0.5.2	default  
myapp                 	1       	Wed May 10 21:46:51 2017	DEPLOYED	myapp-0.1.0	default

#通过传入yml文件来安装
#helm install --name myapp -f myvalues.yaml ./myapp
```

#### 构建私有charts repo

通过 `helm repo list`, 得知默认的local repo地址是`http://127.0.0.1:8879/charts`， 可以简单的通过`helm serve`来操作，再或者自己起个web server也是一样的。


这里举例，把刚才创建的myapp放到本地仓里

```bash
➜  helm search myapp
No results found
➜
➜  source <(helm completion zsh)
➜    
➜  helm package myapp
➜  
➜  helm serve &
[1] 10619
➜  Regenerating index. This may take a moment.
Now serving you on 127.0.0.1:8879

➜  deis helm search myapp
NAME       	VERSION	DESCRIPTION                
local/myapp	0.1.0  	A Helm chart for Kubernetes
```

目前个人感觉体验不太好的是，私有仓里的app必须以tar包的形式存在。


#### 构建复杂应用

透过例子学习，会加速理解，我们从deis里的workflow应用来介绍

```bash
➜  ~ helm repo add deis https://charts.deis.com/workflow
"deis" has been added to your repositories
➜  ~
➜  ~ helm search workflow                               
NAME         	VERSION	DESCRIPTION  
deis/workflow	v2.14.0	Deis Workflow
➜  ~
➜  ~ helm fetch deis/workflow --untar
➜  ~ helm dep list workflow
NAME                    	VERSION	REPOSITORY                                      	STATUS  
builder                 	v2.10.1	https://charts.deis.com/builder                 	unpacked
slugbuilder             	v2.4.12	https://charts.deis.com/slugbuilder             	unpacked
dockerbuilder           	v2.7.2 	https://charts.deis.com/dockerbuilder           	unpacked
controller              	v2.14.0	https://charts.deis.com/controller              	unpacked
slugrunner              	v2.3.0 	https://charts.deis.com/slugrunner              	unpacked
database                	v2.5.3 	https://charts.deis.com/database                	unpacked
fluentd                 	v2.9.0 	https://charts.deis.com/fluentd                 	unpacked
redis                   	v2.2.6 	https://charts.deis.com/redis                   	unpacked
logger                  	v2.4.3 	https://charts.deis.com/logger                  	unpacked
minio                   	v2.3.5 	https://charts.deis.com/minio                   	unpacked
monitor                 	v2.9.0 	https://charts.deis.com/monitor                 	unpacked
nsqd                    	v2.2.7 	https://charts.deis.com/nsqd                    	unpacked
registry                	v2.4.0 	https://charts.deis.com/registry                	unpacked
registry-proxy          	v1.3.0 	https://charts.deis.com/registry-proxy          	unpacked
registry-token-refresher	v1.1.2 	https://charts.deis.com/registry-token-refresher	unpacked
router                  	v2.12.1	https://charts.deis.com/router                  	unpacked
workflow-manager        	v2.5.0 	https://charts.deis.com/workflow-manager        	unpacked

➜  ~ ls workflow
charts  Chart.yaml  requirements.lock  requirements.yaml  templates  values.yaml
```

如上操作，我们会得到一个巨型应用，实际上便是deis出品的workflow开源paas平台，具体这个平台的介绍下次有机会再分享

整个大型应用是通过 wofkflow/requirements.yaml组织起来的，所有依赖的chart放到charts目录，然后charts目录里就是些类似myapp的小应用

更复杂的应用，甚至有人把openstack用helm安装到Kubernetes上，感兴趣的可以参考[这里](https://github.com/openstack/openstack-helm)
