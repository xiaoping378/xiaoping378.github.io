### 构建生产环境级的docker Swarm集群

此文档适用于不低于1.12版本的docker，因为swarm已内置于docker-engine里。

1. 硬件需求

  这里以5台PC服务器为例, 分别如下作用
  * manager0
  * manager1
  * node0
  * node1
  * node2

2. 每台PC上安装docker-engine

  一台一台的ssh上去执行，或者使用ansible批量部署工具。

  安装docker-engine
  ```
  curl -sSL https://get.docker.com/ | sh
  ```
  启动之，并使之监听2375端口
  ```
  sudo docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
  ```
  亦可修改配置，使之永久生效
  ```
mkdir /etc/systemd/system/docker.service.d
cat <<EOF >>/etc/systemd/system/docker.service.d/docker.conf
[Service]
    ExecStart=
    ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --dns 180.76.76.76  --insecure-registry registry.cecf.com -g /home/Docker/docker
EOF
  ```
  如果开启了防火墙，需要开启如下端口
  * **TCP port 2377** for cluster management communications
  * **TCP** and **UDP port 7946** for communication among nodes
  * **TCP** and **UDP port 4789** for overlay network traffic

3. 创建swarm

  ```bash
  docker swarm init --advertise-addr <MANAGER-IP>
  ```

  我的实例里如下：

  ```bash
  [root@manager0 ~]# docker swarm init --advertise-addr 10.42.0.243
  Swarm initialized: current node (e5eqi0lue90uidzsfddeqwfl8) is now a manager.

  To add a worker to this swarm, run the following command:

      docker swarm join \
      --token SWMTKN-1-3iskhw3lsc9pkdtijj1d23lg9tp7duj18f477i5ywgezry7zlt-dfwjbsjleoajcdj13psu702s6 \
      10.42.0.243:2377

  To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
  ```
  使用 `--advertise-addr` 来声明manager0的IP，其他的nodes必须可以和此IP互通，
  一旦完整初始化，此node即是manger又是worker node.

  通过`docker info`来查看

  ```
  $ docker info

  Containers: 2
  Running: 0
  Paused: 0
  Stopped: 2
    ...snip...
  Swarm: active
    NodeID: e5eqi0lue90uidzsfddeqwfl8
    Is Manager: true
    Managers: 1
    Nodes: 1
    ...snip...
  ```

  通过`docker node ls`来查看集群的node信息

  ```
  [root@manager0 ~]# docker node ls
  ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
  e5eqi0lue90uidzsfddeqwfl8 *  manager0  Ready   Active        Leader

  ```

  这里的`*`来指明docker client正在链接在这个node上。

4. 加入swarm集群

  执行在manager0上产生`docker swarm init`产生的结果即可

  如果当时没记录下来，还可以在manager上补看
  想把node以worker身份加入，在manager0上执行下面的命令来补看。

  ```
  docker swarm join-token worker
  ```
  想把node以manager身份加入，在manager0上执行下面的命令来来补看。

  ```
  docker swarm join-token manager
  ```

  为了manager的高可用，我这里需要在manager1上执行
  ```
  docker swarm join \
  --token SWMTKN-1-3iskhw3lsc9pkdtijj1d23lg9tp7duj18f477i5ywgezry7zlt-86dk7l9usp1yh4uc3rjchf2hu \
  10.42.0.243:2377
  ```

  我这里就是依次在node0-2上执行
  ```
  docker swarm join \
    --token SWMTKN-1-3iskhw3lsc9pkdtijj1d23lg9tp7duj18f477i5ywgezry7zlt-dfwjbsjleoajcdj13psu702s6 \
    10.42.0.243:2377
  ```

  这样node就会加入之前我们创建的swarm集群里。

  再通过`docker node ls`来查看现在的集群情况， swarm的集群里是以node为实例的

  ```bash
  [root@manager0 ~]# docker node ls
  ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
  0tr5fu8ebi27cp2ot210t67fx    manager1  Ready   Active        Reachable
  46irkik4idjk8rjy7pqjb84x0    node1     Ready   Active        
  79hlu1m7x9p4cc4npa4xjuax3    node0     Ready   Active        
  9535h8ow82s8mzuw5kud2mwl3    consul0   Ready   Active        
  e5eqi0lue90uidzsfddeqwfl8 *  manager0  Ready   Active        Leader
  ```

  这里MANAFER标明各node的身份，空即为worker身份。

5. 部署服务

  ```
  Usage:	docker service COMMAND

  Manage Docker services

  Options:
        --help   Print usage

  Commands:
    create      Create a new service
    inspect     Display detailed information on one or more services
    ps          List the tasks of a service
    ls          List services
    rm          Remove one or more services
    scale       Scale one or multiple services
    update      Update a service
  ```

  部署示例如下：
  ```
  docker service create --replicas 2 --name helloworld alpine ping 300.cn
  ```

  `docker service ls`罗列swarm集群的所有services
  `docker service ps helloworld`查看service部署到了哪个node上
  `docker service inspect helloworld` 查看service 资源、状态等具体信息
  `docker servcie scale helloworld=5`来扩容service的个数
  `docker service rm helloworld` 来删除service
  `docker service update` 来实现更新service的各项属性，包括滚动升级等。

  可更新的属性包含如下：
  ```
  Usage:	docker service update [OPTIONS] SERVICE

  Update a service

  Options:
        --args string                    Service command args
        --constraint-add value           Add or update placement constraints (default [])
        --constraint-rm value            Remove a constraint (default [])
        --container-label-add value      Add or update container labels (default [])
        --container-label-rm value       Remove a container label by its key (default [])
        --endpoint-mode string           Endpoint mode (vip or dnsrr)
        --env-add value                  Add or update environment variables (default [])
        --env-rm value                   Remove an environment variable (default [])
        --help                           Print usage
        --image string                   Service image tag
        --label-add value                Add or update service labels (default [])
        --label-rm value                 Remove a label by its key (default [])
        --limit-cpu value                Limit CPUs (default 0.000)
        --limit-memory value             Limit Memory (default 0 B)
        --log-driver string              Logging driver for service
        --log-opt value                  Logging driver options (default [])
        --mount-add value                Add or update a mount on a service
        --mount-rm value                 Remove a mount by its target path (default [])
        --name string                    Service name
        --publish-add value              Add or update a published port (default [])
        --publish-rm value               Remove a published port by its target port (default [])
        --replicas value                 Number of tasks (default none)
        --reserve-cpu value              Reserve CPUs (default 0.000)
        --reserve-memory value           Reserve Memory (default 0 B)
        --restart-condition string       Restart when condition is met (none, on-failure, or any)
        --restart-delay value            Delay between restart attempts (default none)
        --restart-max-attempts value     Maximum number of restarts before giving up (default none)
        --restart-window value           Window used to evaluate the restart policy (default none)
        --stop-grace-period value        Time to wait before force killing a container (default none)
        --update-delay duration          Delay between updates
        --update-failure-action string   Action on update failure (pause|continue) (default "pause")
        --update-parallelism uint        Maximum number of tasks updated simultaneously (0 to update all at once) (default 1)
    -u, --user string                    Username or UID
        --with-registry-auth             Send registry authentication details to swarm agents
    -w, --workdir string                 Working directory inside the container
  ```
