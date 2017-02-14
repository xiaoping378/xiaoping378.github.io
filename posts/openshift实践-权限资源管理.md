# openshift实践-权限资源管理

重点介绍 project，limitRange，resourceQuta和 user, group, rule，role，policy，policybinding的关系,
我刚接触时，这几个概念老搞不太清楚，这里梳理下

## 资源管理说明

可以对计算资源的大小和对象类型的数量来进行配额限制。

``ResourceQuota``是面向project（namespace的基础上加了些注解）层面的，只有集群管理员可以基于namespace设置。

``limtRange``是面向pod和container级别的，openshift额外还可以限制 image， imageStream和pvc，
也是只有集群管理源才可以基于project设置，而开发人员只能基于pod（container）设置cpu和内存的requests/limits。

### ResourceQuota

看看具体可以管理哪些资源，期待网络相关的也加进来.简单来讲，可以基于project来限制可消耗的内存大小和可创建的pods数量

```go
// The following identify resource constants for Kubernetes object types
const (
	// Pods, number
	ResourcePods ResourceName = "pods"
	// Services, number
	ResourceServices ResourceName = "services"
	// ReplicationControllers, number
	ResourceReplicationControllers ResourceName = "replicationcontrollers"
	// ResourceQuotas, number
	ResourceQuotas ResourceName = "resourcequotas"
	// ResourceSecrets, number
	ResourceSecrets ResourceName = "secrets"
	// ResourceConfigMaps, number
	ResourceConfigMaps ResourceName = "configmaps"
	// ResourcePersistentVolumeClaims, number
	ResourcePersistentVolumeClaims ResourceName = "persistentvolumeclaims"
	// ResourceServicesNodePorts, number
	ResourceServicesNodePorts ResourceName = "services.nodeports"
	// ResourceServicesLoadBalancers, number
	ResourceServicesLoadBalancers ResourceName = "services.loadbalancers"
	// CPU request, in cores. (500m = .5 cores)
	ResourceRequestsCPU ResourceName = "requests.cpu"
	// Memory request, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
	ResourceRequestsMemory ResourceName = "requests.memory"
	// Storage request, in bytes
	ResourceRequestsStorage ResourceName = "requests.storage"
	// CPU limit, in cores. (500m = .5 cores)
	ResourceLimitsCPU ResourceName = "limits.cpu"
	// Memory limit, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
	ResourceLimitsMemory ResourceName = "limits.memory"
)
```
openshift额外支持的images相关的限制策略

```go
// ResourceImageStreams represents a number of image streams in a project.
ResourceImageStreams kapi.ResourceName = "openshift.io/imagestreams"

// ResourceImageStreamImages represents a number of unique references to images in all image stream
// statuses of a project.
ResourceImageStreamImages kapi.ResourceName = "openshift.io/images"

// ResourceImageStreamTags represents a number of unique references to images in all image stream specs
// of a project.
ResourceImageStreamTags kapi.ResourceName = "openshift.io/image-tags"

```

此外，除了可以设置额度Quantity外，还可以指定配额的作用范围Scopes，其实就是作用于哪类pod上的:

 * 是否是长期运行的pod
 * 是否有资源上限的pod

目前只有pods数和计算资源（cpu,内存）才能指定作用域

```go
// A ResourceQuotaScope defines a filter that must match each object tracked by a quota
type ResourceQuotaScope string

const (
	// Match all pod objects where spec.activeDeadlineSeconds，这个是标明pod的运行时长参数
	ResourceQuotaScopeTerminating ResourceQuotaScope = "Terminating"
	// Match all pod objects where !spec.activeDeadlineSeconds ， 长期运行的pod
	ResourceQuotaScopeNotTerminating ResourceQuotaScope = "NotTerminating"
	// Match all pod objects that have best effort quality of service， 只能用来描述资源无上限的pod数
	ResourceQuotaScopeBestEffort ResourceQuotaScope = "BestEffort"
	// Match all pod objects that do not have best effort quality of service， 资源有上限的pod
	ResourceQuotaScopeNotBestEffort ResourceQuotaScope = "NotBestEffort"
)
```


下面举个例子

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources-long-running
spec:
  hard:
    pods: "4"
    limits.cpu: "4"
    limits.memory: "2Gi"
  scopes:
  - NotTerminating
```

上面的意思即是， 限制长期运行的pod最多只能创建4个，且共用4c和2G内存

如果不指定scopes的话，是描述的所有scopes的限制；

> 本文参考[这里](https://docs.openshift.org/latest/admin_guide/quota.html)

可以看到，通过资源配额管理，可以帮助我们解决以下问题：

* 控制计算资源使用量

  我们在实际生产环境中经常遇到的情况是，用户申请了过多的资源，用户应用的资源使用率太低，造成了资源的浪费。管理员通常会给集群设置超卖系数，来提高整个集群的资源使用率；另外管理员也会给用户设置资源配额上限，来限制用户使用资源的数量。通过上面的介绍我们可以看到，kubernetes的资源配额，我们可以从应用的层次上来进行配额管理，可以设置不同应用的资源配额上限。

* 控制besteffort类型POD资源使用量

  如果POD中的所有容器都没有设置request和limit，那么这些POD的QoS类型是besteffort，这种类型的POD更方便kubernetes进行调度，但是存在的问题是，如果不对这些POD进行资源管理，那么就会导致这个kubernetes集群资源过载，会影响这个集群中的所有应用，所以通过将资源配额管理的作用范围设置成besteffort，kubernetes可以通过限制这些POD的资源，避免整个集群资源过载。

* 控制长期运行的应用和短暂运行的应用资源使用率

  在实际使用中，在kubernetes集群中会同时存在两种类型的应用，一种是长期运行的应用，比如网站这种web应用，还有一种就是短暂运行的应用，比如编译网站的这种应用。通过资源配额管理，可以同时对这两种不同类型的应用设置资源使用上限，来控制不同应用的资源使用。

### LimitRange

``limtRange``是面向pod和container级别的，为什么只能集群管理员才可设置呢，因为这个的提出是为了防止有些应用忘记加资源边界的限定，而占用过多的资源，那么有了limitRange就给它来个默认限制。

```yaml
apiVersion: "v1"
kind: "LimitRange"
metadata:
  name: "core-resource-limits"
spec:
  limits:
    - type: "Pod"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "200m"
        memory: "6Mi"
    - type: "Container"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "100m"
        memory: "4Mi"
      default:
        cpu: "300m"
        memory: "200Mi"
      defaultRequest:
        cpu: "200m"
        memory: "100Mi"
      maxLimitRequestRatio:
        cpu: "10"
    - type: "openshift.io/Image"
      max:
        storage: "1Gi"
    - type: "openshift.io/ImageStream"
      max:
        openshift.io/image-tags: "10"
        openshift.io/images: "12"
```


如上使用oc create后，会看到我们对某namespace下的pod和container做了默认的资源设置，

![limitRange](/assets/limitRange.png)

## 权限管理说明

这里不涉及到认证登录的介绍，openshfit支持很多认证方式，比如AllowAll，CA认证, HTPasswd, KeyStone, LDAP, Oauth等，这里为了简化，用默认的AllowAll来做权限控制的说明

权限管理，即访问API资源之前，必须要经过的访问策略校验，主要分为5种： AlwaysDeny、AlwaysAllow（默认）、ABAC、RBAC、Webhook

主要说明user, group, rule，role，policy，policybinding之间的关系，以及提出这些概念，各自是为了解决什么问题

* user和group

说到user其实就是一个用户账号(userAccount)，用它来和k8s集群做交互（登录，kubectl等）， 但还有一个容易混淆的概念就是sercieAccount，有了userAccount为什么还又来个serviceAccount的设计， 这两者有什么区别 ？ 以下是kubernetes官方对两者的解释

>user account是为人类设计的，而service account则是为跑在pod里的进程用的，运行在pod里的进程需要调用Kubernetes API以及非Kubernetes API的其它服务（如image repository/被mount到pod上的NFS volumes中的file等）;

>user account是global的，即跨namespace使用；而service account是namespaced内的，即仅在所属的namespace下使用;

>user account可能会涉及到很多权限设定和商业逻辑在里面，而后者是更轻量级的，是集群用户针对某namespace内的服务使用的，一般遵循最小特权原则，如监控服务需要访问APIsever等;

>useraccount需要借助第三方实现，后者系统都会默认在namesspace里创建default，亦可自定义

两者大部分流程是一致的，都是要先认证通过再校验权限，然后才是action， 实际上一般是由userAccount来控制serviceAccount来完成特定的任务，
比如一个用户A自建了服务1和服务2， 但只想把服务2开发给用户B，这样的serviceAccount就可以排上用场了,
又或者我有几个服务，有了serviceAccount就可以来限制用户的访问权限（list, watch, update, delete）了.


说到group就是方便对user的权限批量操作而设计；

用户可以被分配到一个或多个组，每个组代表一组特定的用户。组在同时向多个用户管理权限时非常有用。

* rule和role

rule是规则， 是对一组对象上被允许的动作（get, list, create, update, delete, deletecollection 和 watch）描述，可操作对象主要是 container，images，pod，servcie， project， user， build， imagestream， dc， route， templeate。

role 就是规则的集合，俗称角色， 不同对象上的不同动作，可以任意组成各种角色，系统默认的有 ``admin basic-user cluster-admin  cluster-admin edit self-provisioner view``；

policy，是策略, 保存特定namespace的所有角色roles的对象。 每个命名空间最多只有一个Policy策略。

rolebinding， 就是把user或者group与角色role进行关联，注意. user和group可以被关联到多个roles

pollicybing, 就是就是多个rolebindings的描述；

这样看，policy的概念提出有点儿扯淡了，感觉没什么用，其实不然，policy的提出主要是为了区分cluster-policy和local-policy的。

cluster policy是适用于所有namespace的角色和绑定；
local policy则是试用于具体的某个namespace的；

以上可以通过``oc describe clusterPolicy default``来看查看所有详细的信息；

小节：

可以通过``oc policy can-i --list``查看自己可以干些什么

还可以通过``oc policy who-can <动作> <资源对象>``， 比如说查看谁能get pod之类的，就是``oc policy who-can get pod``

```shell
➜  openshift-docs git:(master) ✗ oc policy who-can get pod
Namespace: myproject
Verb:      get
Resource:  pods

Users:  developer
        system:admin
        system:serviceaccount:default:pvinstaller
        system:serviceaccount:myproject:deployer
        system:serviceaccount:openshift-infra:build-controller
        system:serviceaccount:openshift-infra:deployment-controller
        system:serviceaccount:openshift-infra:deploymentconfig-controller
        system:serviceaccount:openshift-infra:endpoint-controller
        system:serviceaccount:openshift-infra:namespace-controller
        system:serviceaccount:openshift-infra:pet-set-controller
        system:serviceaccount:openshift-infra:pv-binder-controller
        system:serviceaccount:openshift-infra:pv-recycler-controller

Groups: system:cluster-admins
        system:cluster-readers
        system:masters
        system:nodes
```

如果openshift自带的角色不能满足的话，还可以自定义角色role
```shell
$ oc get clusterrole view -o yaml > clusterrole_view.yaml
$ cp clusterrole_view.yaml localrole_exampleview.yaml
$ vim localrole_exampleview.yaml
# 1. Update kind: ClusterRole to kind: Role
# 2. Update name: view to name: exampleview
# 3. Remove resourceVersion, selfLink, uid, and creationTimestamp
$ oc create -f path/to/localrole_exampleview.yaml -n <project_you_want_to_add_the_local_role_exampleview_to>
```

## 下文介绍实战，结合实际场景，如何设置权限，即整个开发管理流程实践说明
