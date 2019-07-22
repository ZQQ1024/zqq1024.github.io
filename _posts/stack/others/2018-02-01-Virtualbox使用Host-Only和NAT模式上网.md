---
title: "Virtualbox使用Host-Only和NAT模式上网"
categories:
  - others
tags:
  - virtualbox
classes: wide

excerpt: "Virtualbox的Host-Only和NAT模式"
---


## 1 需求
出现了这样的需求：
- 主机和虚拟机要互相访问
- 同时虚拟机要能够访问外网

解决方法是：在Virtualbox下使用Host-Only + NAT模式（或者使用VM，VM自带的一种好像就能够实现）

## 2 背景

- 主机操作系统：Ubuntu 16.04 LTS（Windows应该也行）
- 虚拟机操作系统：Ubuntu 16.04 LTS
- 虚拟机软件：Virtualbox
- Virtualbox网络模式：
  - NAT：虚拟机可以访问外网，但是主机和虚拟机不能互相访问
  - Host-Only：虚拟机不可以访问外网，但是主机和虚拟机能互相访问

## 3 解决方法
#### 3.1 添加网卡
添加2张网卡，分别为Host-Only和NAT模式：  
![网卡1](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190718192358.png)  
![网卡2](https://raw.githubusercontent.com/ZQQ1024/pictures/master/97557887-2018-02-01%2010-32-42%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

#### 3.2 网卡配置
进入虚拟机系统 ```ifconfig``` 查看网卡信息：  
![ifconfig](https://raw.githubusercontent.com/ZQQ1024/pictures/master/97557900-2018-02-01%2010-37-58%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

如上图所示，enp0s3是上面配置的第一个网卡（Host-Only模式）， ```ifconfig -a``` 查看所有网卡信息，出现enp0s8，是上面配置的第二个网卡（NAT模式），说明第二个网卡没有up。  
![ifconfig -a](https://raw.githubusercontent.com/ZQQ1024/pictures/master/97558048-2018-02-01%2010-47-13%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

通过命令 ```ifconfig enp0s8 up``` 启动网卡，然后 ```dhclient enp0s8``` 动态分配ip。然后ifconfig就可以看到2张网卡的信息了。

![2张网卡信息](https://raw.githubusercontent.com/ZQQ1024/pictures/master/97558049-2018-02-01%2010-50-49%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

#### 3.3 持久化
上面操作后，已经能够完成我们的需求了，但是没有持久化退出shell或者重启就没了，需要将上述网卡信息写入文件 ```/etc/network/interfaces``` 里（CentOS是在 ```/etc/sysconfig/network-scripts``` 目录下）。  
![网卡信息写入文件](https://raw.githubusercontent.com/ZQQ1024/pictures/master/97558052-2018-02-01%2010-59-21%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

## 4 测试
#### 4.1 ifconfig
```ifconfig``` 有2张网卡（enp0s3和enp0s8）的信息了

#### 4.2 ping www.baidu.com
虚拟机```ping www.baidu.com``` ，看能否访问外网：  
![ping](https://raw.githubusercontent.com/ZQQ1024/pictures/master/97558054-2018-02-01%2011-05-20%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

#### 4.3 虚拟机和主机互相ping
- 虚拟机ip：192.168.99.100
- 主机ip：192.168.99.1

主机ping虚拟机：  
![主机ping虚拟机](https://raw.githubusercontent.com/ZQQ1024/pictures/master/97558056-2018-02-01%2011-07-50%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

虚拟机ping主机：  
![虚拟机ping主机](https://raw.githubusercontent.com/ZQQ1024/pictures/master/97558055-2018-02-01%2011-09-02%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

## 5 参考
> [VirtualBox 设置 Host-Only 上网](http://www.wenzhixin.net.cn/2014/07/31/virtual_box_host_only_settings)