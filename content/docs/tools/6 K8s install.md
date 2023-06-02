---
title: "K8s install"
weight: 6
bookToc: true
---

# K8s install

主要记录`K8s`单节点集群搭建和多节点集群搭建过程

## 1 单节点
此部分主要说明单节点learning 集群的搭建

安装方式：kubeadm  
机器规格：2c4g * 1台   
节点系统：Centos 7.6  
K8s版本：1.18.5  
容器运行时：docker  
网络插件：calico  

### 1.1 安装容器运行时
> [https://docs.docker.com/engine/install/centos/](https://docs.docker.com/engine/install/centos/)
```bash
[root@test ~]# sudo yum remove docker \
                   docker-client \
                   docker-client-latest \
                   docker-common \
                   docker-latest \
                   docker-latest-logrotate \
                   docker-logrotate \
                   docker-engine

[root@test ~]# sudo yum install -y yum-utils

[root@test ~]# sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

[root@test ~]# sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

[root@test ~]# systemctl enable docker
[root@test ~]# systemctl start docker
```

### 1.2 安装kubelet、kubeadm、kubectl
修改内核参数
```bash
[root@test ~]# cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

[root@test ~]# sudo sysctl --system
```
{{< hint info >}}
为什么 kubernetes 环境要求开启 bridge-nf-call-iptables
> https://imroc.cc/post/202105/why-enable-bridge-nf-call-iptables/
{{< /hint >}}

先禁用掉`swap`，持久化修改需要修改`/etc/fstab`
```bash
[root@test ~]# sudo swapoff -a
```

关闭`selinux`
```bash
[root@test ~]# sudo setenforce 0
[root@test ~]# sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

三个kubexxx组件的作用如下：
- `kubeadm`: 用于创建、管理、引导集群
- `kubelet`: 启动在每个集群节点，用于创建pod等
- `kubectl`: 用于使用集群的命令行工具

```bash
[root@test ~]# cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

[root@test ~]# sudo yum install -y kubelet-1.18.5-0 kubeadm-1.18.5-0 kubectl-1.18.5-0 --disableexcludes=kubernetes

[root@test ~]# sudo systemctl enable --now kubelet
```

> [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

### 1.3 kubeadm初始化集群
```bash
[root@test ~]# kubeadm init --kubernetes-version=1.18.5 --pod-network-cidr=192.168.0.0/16
```
> [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

{{< hint info >}}
不指定kubernetes-version会使用stable-18版本，即1.8.20
{{< /hint >}}

```bash
[root@test ~]# export KUBECONFIG=/etc/kubernetes/admin.conf
```
可以看到在没有安装网络插件前，coredns处于`Pending`状态
```bash
[root@test ~]# kubectl get pods -n kube-system
NAME                            READY   STATUS    RESTARTS   AGE
coredns-66bff467f8-qxw2c        0/1     Pending   0          6m31s
coredns-66bff467f8-wcn6w        0/1     Pending   0          6m31s
etcd-test                       1/1     Running   0          6m33s
kube-apiserver-test             1/1     Running   0          6m33s
kube-controller-manager-test    1/1     Running   0          6m33s
kube-proxy-jz2ng                1/1     Running   0          6m31s
kube-scheduler-test             1/1     Running   0          6m33s
```

### 1.4 安装网络插件
```bash
[root@test ~]# kubectl create -f https://docs.projectcalico.org/archive/v3.18/manifests/tigera-operator.yaml

// 注意文件中的IP pool CIDR需要和上面init阶段保持一致
[root@test ~]# kubectl create -f https://docs.projectcalico.org/archive/v3.18/manifests/custom-resources.yaml
```
> [https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

{{< hint info >}}
需要注意文档中的`System requirements`，如`Calico v3.26`明确说明了不支持`Kubernetes v1.20 or below`  
https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements

需要找到支持示例中`1.18`版本K8s的Calico版本，如`Calico v3.18`  
https://docs.tigera.io/archive/v3.18/getting-started/kubernetes/requirements
{{< /hint >}}

去掉master污点以便pod能调度到master节点上
```bash
[root@test ~]# kubectl taint nodes --all node-role.kubernetes.io/master-
```

节点最终为`Ready`状态
```bash
[root@test ~]# kubectl get nodes
NAME    STATUS   ROLES    AGE    VERSION
test    Ready    master   8m8s   v1.18.5
```

## 2 多节点

此部分主要说明多节点production 集群的搭建

安装方式：kubeadm  
机器规格：2c4g * 5台（k8s01-03: 3master / k8s04-05: 2worker）  
节点系统：Centos 7.6  
K8s版本：1.18.5  
容器运行时：docker  
网络插件：calico  

{{< hint warning >}}
生产环境一般会有以下改动：
- `etcd`服务满足高可用
- `kubeadm init`时使用config配置文件而不是命令行参数
- 无法访问公网，需要指定内部使用的镜像仓库
- `kubectl`等客户端访问的apiserver endpoint满足高可用
{{< /hint >}}

### 2.1 安装容器运行时
在所有5个节点上安装容器运行时，参看1.1章节

### 2.2 安装kubelet、kubeadm、kubectl
在所有5个节点上安装kubelet、kubeadm、kubectl，参看1.2章节

### 2.3 kubeadm初始化集群/加入集群
```
sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs
```

### 2.4 安装网络插件

## 3 其他操作

修改主机名重新加入集群

升级支持集群

支持ipv6

容器运行时切换

## 4 内部细节

此部分讲解安装过程中涉及到的一些内部细节

`init`过程包含了很多个`phase`:
- `preflight`: 做了一些检测，如是否打开了防火墙、是否关闭了swap、容器运行时是否在运行、进行`kubeadm config images pull`拉取镜像等
- `kubelet-start`: 启动kubelet
- `certs`: 生成一堆证书
- `kubeconfig`: 生成了一堆kubeconfig，如`admin.conf`/`kubelet.conf`/`controller-manager.conf`/`scheduler.conf`
- `control-plane`: 生成了静态pod的yaml文件，如`kube-apiserver`/`kube-controller-manager`/`kube-scheduler`
- `etcd`: 生成`etcd`静态yaml文件
- `wait-control-plane`: 等待静态pod运行起来
- `kubelet-check`:
- `apiclient`:
- `upload-config`:
- `kubelet`:
- `upload-certs`:
- `mark-control-plane`:
- `bootstrap-token`:
- `kubelet-finalize`:
- `addons`: 启动`CoreDNS`和`kube-proxy`

各个组件的作用及关系

生成的一堆证书作用，需要先知道不同组件是怎么通讯的


3 master的作用，master是不是越多越好

Having an odd number of control plane nodes can help with leader selection in the case of machine or zone failure.