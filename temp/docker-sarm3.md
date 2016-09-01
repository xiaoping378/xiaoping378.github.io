### 构建生产环境级的docker Swarm集群 3

如前文所述，默认已经搭建好环境，基于docker1.12版本。

```bash
[root@manager0 ~]# docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
0bbmd3r7aphs374qaea4zcieo    node2     Ready   Active
3qmxzyauc0bz4kjqvld9uogz5    manager1  Ready   Active        Reachable
5ewbdtvaopj4ltwqx0a4i65nt *  manager0  Ready   Drain         Leader
5oxxpgk69fnwe5w210kovrqi9    node1     Ready   Active
7s1ilay2wkjgt09bp2z0743m7    node0     Ready   Active
```

1. 创建第一个服务，以redis为例
  swarm里容器间通信需要使用overlay模式，所以需要提前建立一个

  ```
  docker network create -d overlay  --subnet 10.254.0.0/16 --gateway  10.254.0.1 mynet1
  docker service create --name redis --network mynet1 redis
  ```

2. 在manager上查看服务部署情况

  ```
  [root@manager0 ~]# docker service ps redis
  ID                         NAME     IMAGE  NODE   DESIRED STATE  CURRENT STATE           ERROR
  9avksjfqr2gxm413dfrezrmgr  redis.1  redis  node1  Running        Running 17 seconds ago
  ```

  实例里，同样可以去node1上用``docker ps``查看

  以上只是最基本的集群创建服务的用法，从中可见，swarm的的调度基本单元是task, 没有pod的概念，一个task可以简单理解成一个docker run的结果。目前swarm里也不支持compose。

  docker官方称，以后会支持vm、pod的调度单元，具体日期未知。

3. 服务调度策略

  使用``docker service create``创建服务， 这其中选择再哪个节点部署，docker 提供了三种调度策略；

  * spread: 默认策略，尽量均匀分布，找容器数少的结点调度
  * binpack: 和spread相反，尽量把一个结点占满再用其他结点
  * random: 随机



4. 服务的高可用和load-balance

  通过``--replicas``参数可以设置服务容器的数量，已达到高可用状态；

  ```
  #创建多副本
  docker service update --replicas 4 redis

  #查看副本部署情况
  [root@manager0 ~]# docker service ps redis
  ID                         NAME     IMAGE  NODE      DESIRED STATE  CURRENT STATE               ERROR
  9avksjfqr2gxm413dfrezrmgr  redis.1  redis  node1     Running        Running 13 minutes ago      
  0olv1sfz6d79wdnorw7jgoyri  redis.2  redis  manager1  Running        Running about a minute ago  
  f3n6deesjlkxu4k48lzabieus  redis.3  redis  node2     Running        Preparing 3 minutes ago     
  80bzarvkiytpv1690sla6unt2  redis.4  redis  node0     Running        Running about a minute ago

  #验证多可用， 总共4个副本，docker内置的DNS服务会默认使用round-robin调度策略来解析主机。
  root@9ed77b4b4432:/data# redis-cli -h redis
  redis:6379> set user 1
  OK
  redis:6379> exit
  root@9ed77b4b4432:/data# redis-cli -h redis
  redis:6379> get user
  (nil)
  redis:6379> set user 2
  OK
  redis:6379> exit
  root@9ed77b4b4432:/data# redis-cli -h redis
  redis:6379> get user
  (nil)
  redis:6379> set user 3
  OK
  redis:6379> exit
  root@9ed77b4b4432:/data# redis-cli -h redis
  redis:6379> get user
  (nil)
  redis:6379> set user 4
  OK
  redis:6379> exit
  root@9ed77b4b4432:/data# redis-cli -h redis
  redis:6379> get user
  "1"
  redis:6379>

  ```
