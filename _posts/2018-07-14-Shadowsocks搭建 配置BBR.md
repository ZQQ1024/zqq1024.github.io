---
title: "Shadowsocks搭建 配置BBR"
categories:
  - linux
tags:
  - ss
  - BBR
  - 翻墙
classes: wide
---

主要内容是包含如下：
- 购买Vultr VPS
- 搭建Shadowsocks代理
- 使用BBR加速

简单介绍下以下概念：
- VPS：虚拟专用服务器，多台VPS共享一台机器的物理基础，所以较专用服务器会受到虚拟化技术和"邻居"的影响，一般至少有一个专用的IP地址
- Shadowsocks：基于sock5（支持UDP和IPv6，会话层协议，在表示层和传输层之间）加密网络传输协议实现的客户端-服务器代理工具
- BBR：谷歌的、开源的TCP拥塞控制算法，弥补了之前几种算法的缺点，能够显著提升网络性能


# 1 购买VPS

这次使用的是Vultr的VPS，5美元一月（按小时计费，销毁不再计费）：

配置如下：
```
CPU: 1 vCore
RAM: 1024 MB
Storage: 25 GB SSD
Bandwidth: 1000 GB
Location: Frankfurt
OS: Ubuntu 16.04 x64
```
# 2 安装配置Shadowsocks

分别在Client（如自己的电脑）和Server（刚刚购买的VPS）安装配置Shadowsocks

## 2.1 Server

ssh连接上服务器：
```
ssh root@xx.xx.xx.xx
```

安装`python-pip`：
```
apt-get install python-pip
```
安装`Shadowsocks`：
```
pip install shadowsocks
```
创建和编辑`/etc/shadowsocks.json`（Server端的配置文件），填写如下内容：
```
{
   "server":"0,0,0,0",
   "server_port":444,
   "password":"xxxx", //这里改下密码
   "timeout":600,
   "method":"aes-256-cfb",
   "fast_open":"true"
}
```
启动Server端，并设置开机自起，将如下命令写入到`/etc/rc.local`（exit 0之前）中：
```
ssserver -c /etc/shadowsocks.json -d start
```

## 2.2 Client

安装`python-pip`：
```
apt-get install python-pip
```
安装`Shadowsocks`：
```
pip install shadowsocks
```
创建和编辑`/etc/shadowsocks.json`（Client端的配置文件），填写如下内容：
```
{
    "server":"xxxx", //替换为VPS的ip
    "server_port":444,
    "local_address":"127.0.0.1",
    "local_port":1080,
    "password":"xxxx", //Server端设置的密码
    "timeout":600,
    "method":"aes-256-cfb",
    "fast_open": true,
    "workers": 1
}
```
启动Client端，并设置开机自起，将如下命令写入到`/etc/rc.local`（exit 0之前）中：
```
sslocal -c /etc/shadowsocks.json
```

至此正常情况下，Client端已经可以访问Google等网站了，但是速度不一定理想

ps：这里可以给VPS打个快照，Vultr现在打快照还不要钱

# 3 配置BBR
主要是修改VPS现有的TCP拥塞控制算法为BBR来提升网络性能，要求是Linux内核版本需要达到4.9

## 3.1 升级内核
ssh连接上服务器：
```
ssh root@xx.xx.xx.xx
```

`uname -r`若显示当前版本在4.9以下，运行如下命令升级Linux内核到最新版本：
```
apt-get install --install-recommends linux-generic-hwe-16.04 
```
再一次`uname -r`显示当前内核版本，满足要求：
```
4.13.0-45-generic
```
重启使用新的内核：
```
reboot
```

## 3.2 使用BBR算法

编辑`/etc/sysctl.conf`文件，在文件底部添加如下内容：
```
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
```
重新载入配置：
```
sysctl -p
```
查看当前使用的算法，输出为`net.ipv4.tcp_congestion_control = bbr`：
```
sysctl net.ipv4.tcp_congestion_control
```

## 3.3 速度测试

PING测试（主要想看下VPS的丢包率）：
```
--- 217.163.28.72 ping statistics ---
30 packets transmitted, 30 received, 0% packet loss, time 29009ms
rtt min/avg/max/mdev = 247.026/300.165/541.389/58.684 ms
```
原始测试：  
[http://www.speedtest.net/result/7469689142](http://www.speedtest.net/result/7469689142)

搬瓦工CN2：  
[http://www.speedtest.net/result/7469444151](http://www.speedtest.net/result/7469444151)

速度测试无BBR：  
[http://www.speedtest.net/result/7469441560](http://www.speedtest.net/result/7469444151)

速度测试有BBR：  
[http://www.speedtest.net/result/7469518415](http://www.speedtest.net/result/7469518415)  
[http://www.speedtest.net/result/7469687562](http://www.speedtest.net/result/7469687562)

ps：若速度不错，打个快照，如果速度不行自己决定要不要恢复前一个快照

# 4 出现的问题

问题一：
```
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_TIME = "zh_CN.UTF-8",
	LC_MONETARY = "zh_CN.UTF-8",
	LC_ADDRESS = "zh_CN.UTF-8",
	LC_TELEPHONE = "zh_CN.UTF-8",
	LC_NAME = "zh_CN.UTF-8",
	LC_MEASUREMENT = "zh_CN.UTF-8",
	LC_IDENTIFICATION = "zh_CN.UTF-8",
	LC_NUMERIC = "zh_CN.UTF-8",
	LC_PAPER = "zh_CN.UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
```
安装中文语言包解决：
```
sudo apt-get -y install language-pack-zh-hans
```


问题二：
```
pip install shadowsocks
Collecting shadowsocks
  Using cached https://files.pythonhosted.org/packages/02/1e/e3a5135255d06813aca6631da31768d44f63692480af3a1621818008eb4a/shadowsocks-2.8.2.tar.gz
    Complete output from command python setup.py egg_info:
    Traceback (most recent call last):
      File "<string>", line 1, in <module>
    ImportError: No module named 'setuptools'
    
    ----------------------------------------
Command "python setup.py egg_info" failed with error code 1 in /tmp/pip-install-u3a7l2cy/shadowsocks/
```
安装setuptools解决：

```
pip install --upgrade setuptools
```

# 5 参考链接

> Vultr：  
[https://www.vultr.com/](https://www.vultr.com/)  
Shadowsocks Server安装配置：  
[https://gist.github.com/nathanielove/40c1dcac777e64ceeb63d8296d263d6d](https://gist.github.com/nathanielove/40c1dcac777e64ceeb63d8296d263d6d)  
VPS IP相关：  
[https://www.quora.com/With-a-VPS-do-you-get-your-own-IP-or-is-the-IP-shared-across-each-VPS-on-the-machine](https://www.quora.com/With-a-VPS-do-you-get-your-own-IP-or-is-the-IP-shared-across-each-VPS-on-the-machine/)  
升级内核：  
[https://askubuntu.com/questions/905049/how-to-update-kernel-in-ubuntu-16-04](https://askubuntu.com/questions/905049/how-to-update-kernel-in-ubuntu-16-04)  
问题一解决：  
[https://zhidao.baidu.com/question/2141287767447338868.html](https://zhidao.baidu.com/question/2141287767447338868.html)  
问题二解决：  
[https://stackoverflow.com/questions/22531360/no-module-named-setuptools](https://stackoverflow.com/questions/22531360/no-module-named-setuptools)  
配置BBR：  
[https://www.linuxbabe.com/ubuntu/enable-google-tcp-bbr-ubuntu](https://www.linuxbabe.com/ubuntu/enable-google-tcp-bbr-ubuntu)  
测速网站：  
[http://www.speedtest.net/](http://www.speedtest.net/)
