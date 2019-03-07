---
title: "haproxy + keepalived"
categories:
  - linux
tags:
  - haproxy
  - keepalived
---

# 1 介绍

本文要实现一个高可用的负载均衡器

`Haproxy`是一个负载均衡器，`Keepalived`可以让`Haproxy`高可用


![image](http://pic.yupoo.com/840486874/HBmemrMh/medish.jpg)

基本原理如下：
- 用户只需访问固定的公网IP
- 公网IP有个到VIP的映射
- VIP是virtual shared/floating ip，自动实现在主从之间的切换
- 协议是vrrp，主发广播，当从没收到或者受到的优先级低于自己的，从升为主

初始说明：

```
Master Node IP：192.168.56.101 ubuntu01
Slave Node IP：192.168.56.102 ubuntu02
VIP：192.168.56.200
```
# 2 安装配置

两台ubuntu上分别安装`Haproxy`和`Keepalived`

`vi /etc/sysctl.conf`添加如下内容来允许绑定非local的IP，`sysctl -p`：

```
netn .ipv4.ip_nonlocal_bind=1
```
**master**节点上`vi /etc/keepalived/keepalived.conf`：
```
global_defs {
# Keepalived process identifier
lvs_id haproxy_DH
}
# Script used to check if HAProxy is running
vrrp_script check_haproxy {
script "killall -0 haproxy"
interval 2
weight 2
}
# Virtual interface
# The priority specifies the order in which the assigned interface to take over in a failover
vrrp_instance VI_01 {
state MASTER
interface enp0s3
virtual_router_id 51
priority 101
# The virtual ip address shared between the two loadbalancers
virtual_ipaddress {
192.168.56.200
}
track_script {
check_haproxy
}
}
```

**slave**节点上`vi /etc/keepalived/keepalived.conf`：
```
global_defs {
# Keepalived process identifier
lvs_id haproxy_DH_passive
}
# Script used to check if HAProxy is running
vrrp_script check_haproxy {
script "killall -0 haproxy"
interval 2
weight 2
}
# Virtual interface
# The priority specifies the order in which the assigned interface to take over in a failover
vrrp_instance VI_01 {
state SLAVE
interface enp0s3
virtual_router_id 51
priority 100
# The virtual ip address shared between the two loadbalancers
virtual_ipaddress {
192.168.56.200
}
track_script {
check_haproxy
}
}
```
配置说明如下：
- 当主和从都没有运行Haproxy时，master的优先级是101，slave的优先级是100，vip在master上
- 当主和从都运行了Haproxy时，master的优先级是103，slave的优先级是102，vip在master上
- 当主没有运行Haproxy，从运行了Haproxy，master的优先级是101，slave的优先级是102，vip在slave上

编辑Haproxy配置文件，`vi /etc/haproxy/haproxy.cfg`，Haproxy配置不重要不多说：
```
global
        log 127.0.0.1   local0
        log 127.0.0.1   local1 notice
        #log loghost    local0 info
        maxconn 4096
        #chroot /usr/share/haproxy
        user haproxy
        group haproxy
        daemon
        #debug
        #quiet

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        retries 3
        option redispatch
        maxconn 2000
        contimeout      5000
        clitimeout      50000
        srvtimeout      50000

listen stats 
        bind 192.168.56.ABC(101 or 102):8989
        mode http
        stats enable
        stats uri /stats
        stats realm HAProxy\ Statistics
        stats auth admin:admin

listen am_cluster 
        bind 0.0.0.0:80
        mode http
        balance roundrobin
        option httpclose
        option forwardfor
        cookie SERVERNAME insert indirect nocache
        server am-1 192.168.X.ABC:80 cookie s1 check
        server am-2 192.168.Y.ABC:80 cookie s2 check
```

# 3 测试

主要两个方向测试：
- master和slave都运行Haproxy
- master停掉Haproxy

**master和slave都运行Haproxy**：

`sudo tcpdump vrrp`：

![image](http://pic.yupoo.com/840486874/HBne7SNF/medish.jpg)

![image](http://pic.yupoo.com/840486874/HBnesW2w/medish.jpg)

`ip addr`：

![image](http://pic.yupoo.com/840486874/HBneYly0/medish.jpg)

![image](http://pic.yupoo.com/840486874/HBneYqb7/medish.jpg)

优先级为103，vip在master(192.168.56.101)上，测试通过

**master停止Haproxy**：

`sudo service haproxy stop`

`sudo tcpdump vrrp`：

![image](http://pic.yupoo.com/840486874/HBnh59gt/medish.jpg)

![image](http://pic.yupoo.com/840486874/HBnh5dj9/medish.jpg)

`ip addr`：

![image](http://pic.yupoo.com/840486874/HBnhjvgB/medish.jpg)

![image](http://pic.yupoo.com/840486874/HBnhjzpA/medish.jpg)

优先级为102，vip在slave(192.168.56.102)上，测试通过

# 4 出现的问题

问题一：本地测试通过，用阿里云的时候没有通过，两个都有vip

解决方法：如下方式改成单播
```
unicast_src_ip  192.168.1.21##（本地IP地址）
unicast_peer {
                192.168.1.22##（对端IP地址）此地址一定不能忘记
                       }
```

问题二：Haproxy按照参考连接配置没有运行起来
```
'listen' cannot handle unexpected argument '192.168.56.101:8989'. parsing [/etc/haproxy/haproxy.cfg:1] : please use the 'bind' keyword for listening addresses. Error(s) found in configuration file : /etc/haproxy/haproxy.cfg
```
解决方法：将`listen  stats   192.168.56.10:8989`改为现在的

# 5 参考链接

[http://dasunhegoda.com/how-to-setup-haproxy-with-keepalived/833/](http://dasunhegoda.com/how-to-setup-haproxy-with-keepalived/833/)

[http://bbs.chinaunix.net/forum.php?mod=viewthread&tid=4174822&page=1](http://bbs.chinaunix.net/forum.php?mod=viewthread&tid=4174822&page=1)

[https://stackoverflow.com/questions/42348849/configuration-for-haproxy-in-ubuntu](https://stackoverflow.com/questions/42348849/configuration-for-haproxy-in-ubuntu)

[https://q.cnblogs.com/q/94405/](https://q.cnblogs.com/q/94405/)
