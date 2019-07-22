---
title: "Docker Swarm"
categories:
  - others
tags:
  - docker
classes: wide

excerpt: "swarm 集群"
---

# 1 环境
机器：
- master：192.168.99.1
- worker1：192.168.99.100
- worker2：192.168.99.101

docker version：
- 192.168.99.1：
  - Client：17.12.0-ce API：1.35
  - Server：17.12.0-ce API：1.35
- 192.168.99.100：
  - Client：17.12.0-ce API：1.35
  - Server：17.12.0-ce API：1.35
- 192.168.99.101：
  - Client：17.12.0-ce API：1.35
  - Server：17.12.0-ce API：1.35

# 2 初始化集群
master init：

```
$ sudo docker swarm init --advertise-addr 192.168.99.1
Swarm initialized: current node (t9tws9uh15gz1n1cycc3zxuje) is now a manager.

To add a worker to this swarm, run the following command:

   docker swarm join \
   --token SWMTKN-1-1gjc0gerhu8qs03udtel58rdclhk30w7ufovfy832dbglf49o6-daory5kfg21ksrm4d0a69fwb5 \
   192.168.99.1:2377


To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

2个分别worker join：
```
$ sudo docker swarm join --token SWMTKN-1-1gjc0gerhu8qs03udtel58rdclhk30w7ufovfy832dbglf49o6-daory5kfg21ksrm4d0a69fwb5 192.168.99.1:2377
This node joined a swarm as a worker.
```
master上显示节点信息：
```
$ sudo docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
w4dyqo85llom2tcwh8my0d5f6     ubuntu01            Ready               Active              
3f67rqzrnfk0cngxaqkztdvbq     ubuntu02            Ready               Active              
t9tws9uh15gz1n1cycc3zxuje *   zqq-910S3L          Ready               Active              Leader
```

# 3 创建服务
创建一个overlay的网路：
```
$ sudo docker network create --driver overlay test-network
v5ssrkygdfc01tnm38yg15twd
$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
9e891f61df37        bridge              bridge              local
bf72e2f0dd6d        docker_gwbridge     bridge              local
bebcb1917988        host                host                local
zmf7d89zax79        ingress             overlay             swarm
44febd158c6b        none                null                local
8277b6f73c0c        test                bridge              local
v5ssrkygdfc0        test-network        overlay             swarm
```
创建一个replicas=2的openresty集群，模拟作为引擎集群，通过```docker service inspect```获得engine service的VIP```10.0.0.28/24```：  
```
$ sudo docker service create --replicas=2 --name engine --network test-network openresty/openresty
rxuangcyut8yk6r4aei5ete48
overall progress: 2 out of 2 tasks 
1/2: running   
2/2: running   
verify: Service converged
```
创建一个replicas=3的openresty集群，模拟作为接入集群，对外暴露节点的30000端口：  
```
$ sudo docker service create --replicas=3 --name access --network test-network --publish 30000:80 openresty/openresty
rveevga170trascverngis5c8
overall progress: 3 out of 3 tasks 
1/3: running   
2/3: running   
3/3: running   
verify: Service converged 
```
查看当前服务和某个服务运行在哪个节点上：  
```
$ sudo docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                        PORTS
rveevga170tr        access              replicated          3/3                 openresty/openresty:latest   *:30000->80/tcp
rxuangcyut8y        engine              replicated          2/2                 openresty/openresty:latest   

$ sudo docker service ps access
ID                  NAME                IMAGE                        NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
l0870dr35bl4        access.1            openresty/openresty:latest   ubuntu02            Running             Running 6 minutes ago                       
1z2jy8v4cqjc        access.2            openresty/openresty:latest   zqq-910S3L          Running             Running 6 minutes ago                       
u1nmjaatiqtk        access.3            openresty/openresty:latest   ubuntu01            Running             Running 6 minutes ago 
```

# 4 模拟
修改每个access容器，代理到引擎集群的VIP```10.0.0.28```：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221113802.png)

修改每个engine容器，反向代理到业务源站（一个代理到百度，一个没有代理）：

engine1：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221113833.png)

engine2：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221113849.png)


连续访问3个节点：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/Peek%202018-02-04%2022-30.gif)


之前第三个节点不能访问，因为创建access的时候出错，具体原因是manager节点的docker版本低于worker的，更新manager版本解决：
```
zqq@zqq-910S3L:~$ sudo docker service ps access
ID            NAME          IMAGE                       NODE                   DESIRED STATE  CURRENT STATE            ERROR                             PORTS
b4xkba9hfspr  access.1      openresty/openresty:latest  localhost.localdomain  Running        Running 34 minutes ago                                     
jahrbpv052el   \_ access.1  openresty/openresty:latest  localhost.localdomain  Shutdown       Rejected 34 minutes ago  "Failed to find a load balance…"  
ykbxa4rrar1x   \_ access.1  openresty/openresty:latest  localhost.localdomain  Shutdown       Rejected 35 minutes ago  "Failed to find a load balance…"  
rtxlj5rccgv9   \_ access.1  openresty/openresty:latest  localhost.localdomain  Shutdown       Rejected 35 minutes ago  "Failed to find a load balance…"  
6mm5ch64t93p  access.2      openresty/openresty:latest  zqq-910S3L             Running        Running 35 minutes ago                                     
zqq@zqq-910S3L:~$ sudo docker service ps engine
ID            NAME      IMAGE                       NODE                   DESIRED STATE  CURRENT STATE           ERROR  PORTS
nsgfji4ybg1c  engine.1  openresty/openresty:latest  localhost.localdomain  Running        Running 36 minutes ago         
iz02utmhm254  engine.2  openresty/openresty:latest  localhost.localdomain  Running        Running 36 minutes ago 
```

# 5 结论
1. 接入集群可以访问引擎集群的VIP，实现访问引擎的负载均衡
2. docker swarm管理会导致性能下降，具体下降多少，要后面测试
3. 之前测试刻意让docker版本不一致，会出现很多问题，比如service rm，容器不会删除等等

# 6 附录
docker remote api 远程管理docker daemon：

1. 在每台机器上修改`/etc/docker/daemon.json`:
```
{
"hosts":["tcp://IP:2375","unix:///var/run/docker.sock"]
}
```
并重启docker  
2. 使用Docker API或者SDK for Python，管理docker，例如拉取镜像或者创建容器或swarm等

附简单写的一个代码：
```
# encoding=utf8
import docker

# 模拟从API获取到的数据
image = "nginx"
tag = "1.9.3"
container = "engine"
port = 80
replicas = 2

# 模拟获取到的节点IP
nodes = ["172.17.122.21","172.17.122.229","172.17.123.99"]

return_value = {}

for ip in nodes:
    # 连接docker daemon
    url = "tcp://%s:2375" % ip
    #print url
    client = docker.DockerClient(base_url=url,version='1.24',timeout=10)
    #print dir(client.images)

    # pull镜像
    #print client.images.list()
    client.images.pull(image, tag)

    # run容器
    # 先把旧的删了
    try:
        old_container = client.containers.get(container)
        #print old_container
        old_container.remove(force=True)
        del old_container
    except:
        pass

    # 启动容器
    hostport = 80
    image_tag = image + ":" + tag
    client.containers.run(image_tag, name=container,ports={hostport:port},detach=True)
    replicas -= 1

    node_container = {}
    node_container["name"] = container
    node_container["port"] = port
    return_value[ip] = node_container

    del client
    if( replicas == 0 ):
        print return_value
        break
```