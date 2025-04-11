---
title: "Internet"
weight: 5
bookToc: true
---

## GNS3访问互联网

部分场景下可能会有`GNS3`中的虚拟机或设备需要访问**互联网**的情况，可以使用`GNS3`中的`NAT`节点实现

网络拓扑如下：

![](/data/image/gns3/internet/image-20250410140425426.png)

`webterm`为一个`debian-based`的docker镜像，里面包含了火狐浏览器，`traceroute`等网路工具。

NAT节点需要运行在 GNS3 VM 或者具备 libvirt 的 Linux 环境中，其默认带了一个DHCP服务器，网段为`192.168.122.0/24`

`webterm`启动前需修改以下配置，dhcp自动获得IP。

![](/data/image/gns3/internet/image-20250410140653559.png)

然后就可以访问互联网

![](/data/image/gns3/internet/image-20250410140929069.png)

{{< hint info >}}
因为NAT的特性，internet或者LAN无法主动访问内部。可以使用下面的Cloud节点实现互访
{{< /hint >}}


## GNS3与本机环境互访

部分场景下可能会有本机（运行`GNS3`的机器）需要和`GNS3`中的虚拟机或设备**互相访问**的情况，可以使用`GNS3`中的`Cloud`节点实现

网络拓扑如下：

![](/data/image/gns3/internet/image-20250411102444707.png)

`Cloud`运行在`GNS3 VM`上（这里GNS3 VM运行在PC的Hype-V中），使用默认的`eth0`网卡

![](/data/image/gns3/internet/image-20250411102639217.png)

{{< hint info >}}
这里测试了如果将Cloud运行在PC上并选择接入网络的以太网卡会出现一些莫名奇妙的问题，如PC收到了关于自己的ARP报文但是并没有回应，也在[该Issue](https://github.com/GNS3/gns3-server/issues/1869)中看到了有人说使用以太网可能会出现一些问题
{{< /hint >}}

`debian`配置如下，链接Cloud的一侧配置dhcp即可（也可以手动分配IP）

```bash
debian@debian:~$ sudo su -

root@debian:~# dhclient ens4

root@debian:~# ip addr
2: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 0c:ca:32:8c:00:00 brd ff:ff:ff:ff:ff:ff
    altname enp0s4
    inet 172.17.82.246/20 brd 172.17.95.255 scope global dynamic ens4
       valid_lft 86064sec preferred_lft 86064sec
    inet6 fe80::eca:32ff:fe8c:0/64 scope link
       valid_lft forever preferred_lft forever
```



验证，debian（GNS3内部）访问外部，本机访问debian：

![](/data/image/gns3/internet/image-20250411104013033.png)

## 参考链接


> https://docs.gns3.com/docs/using-gns3/advanced/the-nat-node
>
> https://docs.gns3.com/docs/using-gns3/advanced/connect-gns3-internet
