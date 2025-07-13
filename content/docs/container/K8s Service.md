---
title: "K8s Service"
weight: 2
bookToc: true
---

## Service

K8s Service 提供了以稳定的IP或域名的方式访问Pod的方法，解决了Pod IP不稳定的问题

以下yaml创建了一个名为`my-service`的`ClusterIP`的服务，K8s会为Service分配虚拟IP，通过`<虚拟IP>:80`访问任意标签为`app:MyApp`的Pod的`9376`端口
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

Service通过`selector`关联pod，会自动创建`endpoints`资源，用于动态存放对应pod的IP和Port信息

```bash
$ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    6h12m
nginx        ClusterIP   10.109.134.113   <none>        8080/TCP   3s
```

```bash
$ kubectl get ep
NAME         ENDPOINTS                                   AGE
kubernetes   172.16.188.19:8443                          6h12m
nginx        172.17.0.3:80,172.17.0.4:80,172.17.0.5:80   6s
```

```bash
$ kubectl get  pods -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-9456bbbf9-6dmhv   1/1     Running   0          60m   172.17.0.5   minikube   <none>           <none>
nginx-deployment-9456bbbf9-mfzbt   1/1     Running   0          60m   172.17.0.4   minikube   <none>           <none>
nginx-deployment-9456bbbf9-zbtt2   1/1     Running   0          60m   172.17.0.3   minikube   <none>           <none>
```

K8s Service 是由`kube-proxy`实现的，有2种模式，`iptables`或`ipvs`

## iptables

Service 主要通过DNAT实现，所以参看`nat`表下的规则，`KUBE-SERVICES`链用于处理K8s Service流量
```bash
$ sudo iptables -L -t nat

......

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-SERVICES  all  --  anywhere             anywhere             /* kubernetes service portals */
......
```

由`10.109.134.113`可得，`KUBE-SVC-2CMXP7HKUVJN7L6M`对应上面的`nginx`服务
```bash
$ sudo iptables -t nat -L KUBE-SERVICES -v
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination

    0     0 KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  any    any     anywhere             10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:https
    0     0 KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  any    any     anywhere             10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:domain
    0     0 KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  any    any     anywhere             10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:domain
    0     0 KUBE-SVC-JD5MR3NA4I4DYORP  tcp  --  any    any     anywhere             10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
    0     0 KUBE-SVC-2CMXP7HKUVJN7L6M  tcp  --  any    any     anywhere             10.109.134.113       /* default/nginx cluster IP */ tcp dpt:webcache
  823 49400 KUBE-NODEPORTS  all  --  any    any     anywhere             anywhere             /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

```bash
$ sudo iptables -t nat -L KUBE-SVC-2CMXP7HKUVJN7L6M -v
Chain KUBE-SVC-2CMXP7HKUVJN7L6M (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  any    any    !10.244.0.0/16        10.109.134.113       /* default/nginx cluster IP */ tcp dpt:webcache
    0     0 KUBE-SEP-C2Y64IBVPH4YIBGX  all  --  any    any     anywhere             anywhere             /* default/nginx */ statistic mode random probability 0.33333333349
    0     0 KUBE-SEP-QST3VAMBSCRN7S4W  all  --  any    any     anywhere             anywhere             /* default/nginx */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-MRYDKJV5U7PLF5ZN  all  --  any    any     anywhere             anywhere             /* default/nginx */
```

或者`iptables-save`导出生成命令

```bash
$ sudo iptables-save -t nat | grep KUBE-SVC-2CMXP7HKUVJN7L6M -A 10

-A KUBE-SERVICES -d 10.109.134.113/32 -p tcp -m comment --comment "default/nginx cluster IP" -m tcp --dport 8080 -j KUBE-SVC-2CMXP7HKUVJN7L6M

-A KUBE-SVC-2CMXP7HKUVJN7L6M ! -s 10.244.0.0/16 -d 10.109.134.113/32 -p tcp -m comment --comment "default/nginx cluster IP" -m tcp --dport 8080 -j KUBE-MARK-MASQ
-A KUBE-SVC-2CMXP7HKUVJN7L6M -m comment --comment "default/nginx" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-C2Y64IBVPH4YIBGX
-A KUBE-SVC-2CMXP7HKUVJN7L6M -m comment --comment "default/nginx" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-QST3VAMBSCRN7S4W
-A KUBE-SVC-2CMXP7HKUVJN7L6M -m comment --comment "default/nginx" -j KUBE-SEP-MRYDKJV5U7PLF5ZN
```

```bash
$ sudo iptables -t nat -L KUBE-SEP-C2Y64IBVPH4YIBGX -v
Chain KUBE-SEP-C2Y64IBVPH4YIBGX (1 references)
 pkts bytes target     prot opt in     out     source               destination

    0     0 KUBE-MARK-MASQ  all  --  any    any     172.17.0.3           anywhere             /* default/nginx */
    0     0 DNAT       tcp  --  any    any     anywhere             anywhere             /* default/nginx */ tcp to:172.17.0.3:80
```

可以看出
- `KUBE-SVC-XXXX`：某个Service的入口链，负责**负载均衡**到多个（这里例子是3个）Endpoint
- `KUBE-SEP-XXXX`：代表一个具体的Endpoint，负责**将流量DNAT到Pod的IP和端口**

可以看出负载均衡算法：基于概率的随机分发
- 第1条：当前3条条目，33.33% 概率 → SEP-C2Y...
- 第2条：还剩2条条目，50%概率 → SEP-QST... (实际概率=66.67%×50%=33.33%)
- 第3条：还剩1条条目，100%概率，剩余流量 → SEP-MRY... (概率=33.33%)

架构如下
![](/data/image/container/4.png)

## ipvs

IPVS（IP Virtual Server） 是 Linux 内核中的一个模块，主要实现了 四层（传输层）负载均衡，基于 Netfilter 框架 实现。LVS（Linux Virtual Server） 是一个负载均衡解决方案或项目，IPVS 是其核心组件。可以理解为 `LVS = IPVS（核心） + 管理工具（如 ipvsadm、keepalived）+ 高可用配置`。

真实服务器对外统一表现为一个虚拟服务（Virtual Service），这个服务绑定在一个**虚拟IP**上。天然契合 Kubernetes 的 Service 设计理念，单一 IP 对外、多后端转发。

---

虽然 Kubernetes 在 v1.6 版本就已支持扩展到 5000 个节点，但在实际落地中，kube-proxy 使用 `iptables` 模式成了一个限制扩展性的重要瓶颈。`iptables` 本质是一个防火墙工具，设计初衷是**包过滤**，规则处理是线性的，需要逐条匹配规则。

当 Service 和 Pod 数量非常大时，规则数量迅速膨胀，匹配效率大幅下降，例如，一个集群节点数 5000 个，如果有 2000 个 NodePort 类型的Service，如果每个 Service 平均有 10 个 Pods，那么**每个节点上都**将有 20000 条 iptables 记录，这将导致内核查找速度下降，重建规则链耗时严重，整体网络性能变差，CPU 占用升高等一系列问题。

相比之下，IPVS 模式用哈希表结构管理规则，查找效率为 O(1)，具备更好的性能和可扩展性。

---

当创建一个 ClusterIP 的Service时，会创建一个dummy网卡(默认名为`kube-ipvs0`)，并将ip绑定到该网卡上。目的是为了让 ClusterIP 成为本机接收数据包的合法目标地址，使其能够进入内核、被 IPVS 正确调度处理，而不会被丢弃。
```bash
$ ip addr
...
7: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether 1a:01:7c:70:74:9a brd ff:ff:ff:ff:ff:ff
    inet 10.96.0.10/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.96.0.1/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.103.93.185/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
...
```

接下来就是会为每个ClusterIP 的Service创建 IPVS Virtual Server。

每个 Service IP + Port 对应一个 IPVS Virtual Server（1:1），每个 Endpoint 对应一个 IPVS Real Server。一个 Service 如果绑定多个 IP（如 ClusterIP、ExternalIP），就会有多个 IPVS 虚拟服务（1:N）。

`-> 172.17.0.3:80                Masq `表示Forward 类型为 Masq（IP伪装，即 SNAT），返回流量也会经过IPVS，DNAT 是默认行为，不需要显式标出来，它始终会做 DNAT 把 VIP 转换为 Pod的IP。
```bash
$ sudo ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 192.168.49.2:8443            Masq    1      2          0
TCP  10.96.0.10:53 rr
  -> 172.17.0.2:53                Masq    1      0          0
TCP  10.96.0.10:9153 rr
  -> 172.17.0.2:9153              Masq    1      0          0
TCP  10.109.134.113:8080 rr
  -> 172.17.0.3:80                Masq    1      0          0
  -> 172.17.0.4:80                Masq    1      0          0
  -> 172.17.0.5:80                Masq    1      0          0
```

虽然使用 IPVS 模式替代了大量 iptables 转发表达式，但由于 K8s 的实际网络需求复杂，仍然需要 iptables 和 ipset 作为补充（这里 ipset 用于减少 iptables 规则数量），用于辅助处理 SNAT、入口访问等边缘场景。

如指定了 Cluster CIDR，结合 iptables 判断哪些 IP 属于集群内部 Pod 网络；如Service 类型为 NodePort，IPVS 不能监听主机网卡，用 iptables 建立 DNAT 规则：NodeIP:Port → ClusterIP:Port。


## References

> https://kubernetes.io/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/

