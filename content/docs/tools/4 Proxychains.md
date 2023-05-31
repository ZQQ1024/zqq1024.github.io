---
title: "Proxychains"
weight: 4
bookToc: false
---

# Proxychains

项目地址：[https://github.com/rofl0r/proxychains-ng](https://github.com/rofl0r/proxychains-ng)，为原`proxychains`的ng版本，现在为4.x版本

项目介绍
```
  ProxyChains is a UNIX program, that hooks network-related libc functions
  in DYNAMICALLY LINKED programs via a preloaded DLL (dlsym(), LD_PRELOAD)
  and redirects the connections through SOCKS4a/5 or HTTP proxies.
  It supports TCP only (no UDP/ICMP etc).
```

实际使用Proxychains时主要有以下2种场景：
1. 让不支持使用代理的软件使用代理
2. 内网渗透时，依次使用每一层搭建的代理形成一条代理链，去访问最内层主机

## 1 安装

{{< tabs "Proxychains install" >}}
{{< tab "CentOS" >}}
```bash
yum install git
cd ~
git clone https://github.com/rofl0r/proxychains-ng.git 
cd proxychains-ng 
./configure && make && make install  
make install-config
```
> https://gist.github.com/rahman541/65760f3c7818f22a3b5c7298950d37a8
{{< /tab >}}
{{< tab "Ubuntu" >}}
```bash
sudo apt install proxychains4
```
> https://manpages.ubuntu.com/manpages/bionic/man1/proxychains4.1.html
{{< /tab >}}
{{< tab "Kali" >}}
```bash
sudo apt install proxychains4
```
> https://www.kali.org/tools/proxychains-ng/
{{< /tab >}}
{{< tab "Windows" >}}
使用Proxifier替代
> https://www.proxifier.com/
{{< /tab >}}
{{< /tabs >}}

## 2 使用场景

### 2.1 让不支持使用代理的软件使用代理

部分应用可以指定代理使用代理，如`curl`应用可以感知`http_proxy`环境变量使用代理

但是有其他应用不支持代理使用，如`telnet`

{{< hint info >}}
`http_proxy`和`HTTP_PROXY`格式区别？
> https://unix.stackexchange.com/questions/212894/whats-the-right-format-for-the-http-proxy-environment-variable-caps-or-no-ca
{{< /hint >}}

修改`proxychains.conf`配置文件
```
[ProxyList]
socks5  10.211.55.2 1086
```
对比`curl ip.gs`是否使用`proxychains`的区别
```bash
[root@k8s01 ~]# curl ip.gs
223.108.79.97

[root@test ~]# proxychains4 curl ip.gs
[proxychains] config file found: /usr/local/etc/proxychains.conf
[proxychains] preloading /usr/local/lib/libproxychains4.so
[proxychains] DLL init: proxychains-ng 4.16
[proxychains] Strict chain  ...  10.211.55.2:1086  ...  ip.gs:80  ...  OK
207.148.117.135
```

### 2.2 代理链

代理访问链路
```
your_host <--> proxy1 <--> proxy2 <--> target_host
```

修改`proxychains.conf`配置文件
```
[ProxyList]
socks5  10.211.55.2 1086
socks5  192.168.1.1 1086
```

注意命令输出
```bash
[root@test ~]# proxychains4 curl ip.gs
[proxychains] config file found: /usr/local/etc/proxychains.conf
[proxychains] preloading /usr/local/lib/libproxychains4.so
[proxychains] DLL init: proxychains-ng 4.16
[proxychains] Strict chain  ...  10.211.55.2:1086  ...  192.168.1.1:1086  ...  ip.gs:80
```
