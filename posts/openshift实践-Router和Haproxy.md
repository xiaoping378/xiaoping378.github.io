# 分析openshift平台的Haproxy多用

haproxy在openshift里默认有两种用处，一个种负责master的高可用，一种是负责外部对内服务的访问（ingress controller）

平台部署情况：
- 3台master，etcd
- 1台node
- 1台lb（haproxy）

## haproxy负载均衡master的高可用

lb负责master间的负载均衡，其实负载没那么大，更多得是用来避免单点故障

### Debug介绍

- 默认安装haproxy1.5.18版本，开启debug方法
  ```
  # 默认systemd对haproxy做了封装，会以-Ds后台形式启动，debug信息是看不到的
  systemctl stop harproxy

  # vi /etc/haproxy/haproxy.cfg
   log         127.0.0.1 local3 debug

  # 手动启动haproxy
  haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -d
  ```
  不知道是不是哪里还需要设置，打印出来的日志，信息并不是不太多

  另外浏览``https://lbIP:9000``, 可以看到统计信息

### 配置介绍

- 使用[openshift-ansible](https://github.com/xiaoping378/openshift-deploy)部署后，harpxy的配置如下

  ```
  [root@node4 ~]# cat /etc/haproxy/haproxy.cfg
  # Global settings
  #---------------------------------------------------------------------
  global
      chroot      /var/lib/haproxy
      pidfile     /var/run/haproxy.pid
      maxconn     20000
      user        haproxy
      group       haproxy
      daemon
      log         /dev/log local0 info    #定义debug级别

      # turn on stats unix socket
      stats socket /var/lib/haproxy/stats

  #---------------------------------------------------------------------
  # common defaults that all the 'listen' and 'backend' sections will
  # use if not designated in their block
  #---------------------------------------------------------------------
  defaults                                #默认配置，后面同KEY的设置会覆盖此处
      mode                    http        #工作在七层代理，客户端请求在转发至后端服务器之前将会被深度分板，所有不与RFC格式兼容的请求都会被拒绝，一些七层的过滤处理手段，可以使用。
      log                     global      #默认启用gloabl的日志设置
      option                  httplog     #默认日志类别为http日志格式
      option                  dontlognull #不记录健康检查日志信息（端口扫描，空信息）
  #    option http-server-close
      option forwardfor       except 127.0.0.0/8 #如果上游服务器上的应用程序想记录客户端的真实IP地址，haproxy会把客户端的IP信息发送给上游服务器，在HTTP请求中添加”X-Forwarded-For”字段,但当是haproxy自身的健康检测机制去访问上游服务器时是不应该把这样的访问日志记录到日志中的，所以用except来排除127.0.0.0，即haproxy自身
      option                  redispatch         #代理的服务器挂掉后，强制定向到其他健康的服务器，避免cookie信息过时，仍可正常访问
      retries                 3     #3次连接失败就认为后端服务器不可用
      timeout http-request    10s   #默认客户端发送http请求的超时时间， 防DDOS攻击手段
      timeout queue           1m    #当后台服务器maxconn满了后，haproxy会把client发送来的请求放进一个队列中，一旦事件超过timeout queue，还没被处理，haproxy会自动返回503错误。
      timeout connect         10s   #haproxy与后端服务器连接超时时间，如果在同一个局域网可设置较小的时间
      timeout client          300s  #默认客户端与haproxy连接后，数据传输完毕，不再有数据传输，即非活动连接的超时时间
      timeout server          300s  #定义haproxy与后台服务器非活动连接的超时时间
      timeout http-keep-alive 10s   #默认新的http请求建立连接的超时时间，时间较短时可以尽快释放出资源，节约资源。和http-request配合使用
      timeout check           10s   #健康检测的时间的最大超时时间
      maxconn                 20000 #最大连接数

  listen stats :9000
      mode http
      stats enable
      stats uri /

  frontend  atomic-openshift-api
      bind *:8443
      default_backend atomic-openshift-api
      mode tcp        #在此模式下，客户端和服务器端之前将建立一个全双工的连接，不会对七层（http）报文做任何检查
      option tcplog

  backend atomic-openshift-api
      balance source  #是基于请求源IP的算法，此算法对请求的源IP时行hash运算，然后将结果除以后端服务器的权重总和，来判断转发至哪台后端服务器，这种方法可保证同一客户端IP的请求始终转发到固定定的后端服务器。
      mode tcp
      server      master0 192.168.56.100:8443 check
      server      master1 192.168.56.101:8443 check
      server      master2 192.168.56.102:8443 check
  ```

  [官方文档](http://cbonte.github.io/haproxy-dconv/1.5/configuration.html)介绍的非常详细，感兴趣的可以继续深入研究


## Router负责外部对内服务的访问

### 部署一个Router并实现高可用
  router是由harpoxy来承担的， 可以理解成kubernetes里的ingress controller部分，默认跑在容器里。

  - 使能 default项目下router，可以访问hostnetwork

        oc adm policy add-scc-to-user hostnetwork system:serviceaccount:default:router

  - 使能其可以查看 label

        oc adm policy add-cluster-role-to-user \
            cluster-reader \
            system:serviceaccount:default:router

  - 部署1个router， 选择具有标签``router=true``的节点
        # 对节点设置标签
        oc label 192.168.56.110 router=true
        # 部署并指定serviceaccount
        oc adm router router --replicas=1 --selector='router=true'  --service-account=router


  - 设置router自身的高可用，参考[这里](https://docs.openshift.org/latest/admin_guide/high_availability.html#admin-guide-high-availability)

      默认使用keepalived实现多个router的高可用，访问router变成访问VIP地址，keepalived再根据权重和健康监测，利用VRRP通告外界后台到底那个router在服务。

        # 添加另一个node作为冗余
        oc label no 192.168.56.111 router=true
        oc scale dc router --replicas=2

        #绑定serviceaccount特权，因为keepalived要操作iptables
        oc adm policy add-scc-to-user privileged system:serviceaccount:default:ipfailover

        #创建keepalived并指定VIP
        oc adm ipfailover ha-router \
          --replicas=2 --watch-port=80 \
          --selector="router=true" \
          --virtual-ips="192.168.56.170" \
          --iptables-chain="INPUT" \
          --service-account=ipfailover --create

      这样，刚才创建的router就自高可用了，通过``192.168.56.170``来访问，有一点值得注意，
      - 按照现在的例子，如果以后还有router做高可用的话，要加上``--vrrp-id-offset=1``，保证一个vip用一个独有的vrrp-id。

### Router分片

路由分片的概念，就是集群内有多个router，通过label来负责不同的routes。

这样可以实现一个project独享一个router，或者某几个route独享一个router，再或者大型集群，更多样化的需求，用这个router sharding的概念也可以满足。

我现在还没有具体的场景，先不实践，后续有机会会跟进更新下。
