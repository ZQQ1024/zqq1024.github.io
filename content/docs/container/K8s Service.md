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

TODO

