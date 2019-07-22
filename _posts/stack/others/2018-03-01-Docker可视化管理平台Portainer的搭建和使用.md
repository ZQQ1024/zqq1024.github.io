---
title: "Docker Portainer"
categories:
  - others
tags:
  - docker
classes: wide

excerpt: "Docker可视化管理平台Portainer的搭建和使用"
---


# 1 背景
Docker可视化管理平台的选择：
- [Rancher](https://github.com/rancher/rancher)：支持K8s，支持中文
- [Portainer](https://github.com/portainer/portainer)：UI for Docker演变而来
- [Shipyard](https://github.com/shipyard/shipyard)：不再维护
- [UI for docker](https://github.com/kevana/ui-for-docker)：变为portainer继续

所以现在推荐使用Rancher和Portainer，本文介绍Portainer的使用。

# 2 目标
Portainer是基于远程或本地管理docker daemon来对docker进行管理的，所以涉及修改docker daemon端口和tls认证的过程。

本文使用portainer要达成的目标是：
- 在IP为192.168.99.101的机器上管理两台docker daemon机器，一台（IP地址192.168.99.100）使用了tls的tcp套接字，一台本机（IP地址192.168.99.101）的unix套接字
- Portainer UI网站使用https访问
- 使用Portainer一些基本功能的使用

# 3 部署
已经为IP地址为192.168.99.101的docker deamon配置了tls，配置过程参看docker client和docker deamon之间的双向认证。

Portainer的作用就是以docker client的身份来管理docker server。

在IP为192.168.99.101上准备之前生成的Client端的证书，为了能够管理docker server：
```
zqq@ubuntu01:~/cert$ pwd
/home/zqq/cert
zqq@ubuntu01:~/cert$ ls
ca.pem  cert.pem  key.pem
```
还是在IP为192.168.99.101上生成自签名的证书，用于Portainer网站的https访问：
```
zqq@ubuntu01:~$ mkdir certs
zqq@ubuntu01:~$ cd certs
zqq@ubuntu01:~/certs$ openssl genrsa -out portainer.key 2048
zqq@ubuntu01:~/certs$ openssl ecparam -genkey -name secp384r1 -out portainer.key
zqq@ubuntu01:~/certs$ openssl req -new -x509 -sha256 -key portainer.key -out portainer.crt -days 3650
zqq@ubuntu01:~/certs$ ls
portainer.crt  portainer.key
```

还是在192.168.99.101上启动Portainer，以容器的方式启动部署Portainer,启动命令是：
```
sudo docker run -d -p 443:9000 \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /home/zqq/cert:/cert \
-v /home/zqq/certs:/certs \
portainer/portainer \
--tlsverify --tlscacert /cert/ca.pem --tlscert /cert/cert.pem --tlskey /cert/key.pem \ #实际不需要和-H搭配才有用
--ssl --sslcert /certs/portainer.crt --sslkey /certs/portainer.key
```
命令的解释如下：
- 三个-v，因为以docker方式部署Portainer，Portainer是以client的身份去管理docker daemon的，所以把需要的文件全部挂载到容器里。```-v /var/run/docker.sock:/var/run/docker.sock```是为了让Portainer容器能够管理本机的docker daemon，```-v /home/zqq/cert:/cert```是为了让容器能够提供Client的tls证书，```-v /home/zqq/certs:/certs```是给容器提供Web UI https访问的自签名证书。
- ```--tlsverify --tlscacert /cert/ca.pem --tlscert /cert/cert.pem --tlskey /cert/key.pem```是作为CMD启动命令的参数存在的，Client的tls证书。
- ```--ssl --sslcert /certs/portainer.crt --sslkey /certs/portainer.key```也是作为CMD启动命令的参数存在的，向容器提供https访问的证书。

至此上面目标说的前两个目标已经基本达成了，下面说Portainer的基本使用。

# 4 使用
访问```https://192.168.99.101/```：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221115207.png)


添加信任后，出现如下界面，设置管理员密码：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221115240.png)

添加对本机（192.168.99.101）docker daemon的管理：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221115253.png)

local的Dashboard：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221115307.png)

添加第二台:tcp://192.168.99.101:2376的endpoint，上传ca.pem、cert.pem和key.pem：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221115318.png)

remote的Dashboard：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221115341.png)


因为local是swarm mode的manager节点，所以有涉及swarm的东西，比如service、secrets等。

pull镜像：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221115427.png)
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221115449.png)

run container：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221115511.png)
端口8080:8080，运行了一个tomcat镜像，管理员账户才能管理。

查看容器：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221115527.png)

容器日志：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221115540.png)

监控数据：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221123516.png)
exec：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221123538.png)
inspect：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221123559.png)
访问：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221123618.png)
添加用户：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221123635.png)
设置名为local的endpoint只能是管理员访问，切换成刚刚创建的用户，则不能访问local的endpoint的docker daemon：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221123658.png)
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20181221123715.png)  
（感觉权限设置这块还是有问题的，而且现在没有使用之前搭的镜像仓库）

# 5 参考
> [https://docs.docker.com/engine/security/https/](https://docs.docker.com/engine/security/https/)    
[https://portainer.readthedocs.io/en/latest/index.html](https://portainer.readthedocs.io/en/latest/index.html)