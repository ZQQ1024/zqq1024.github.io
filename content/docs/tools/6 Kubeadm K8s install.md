---
title: "Kubeadm K8s install"
weight: 6
bookToc: true
---

# Kubeadm K8s install

主要记录`K8s`单节点集群搭建和多节点集群搭建思路及一些常见操作

## 1 单节点 - 学习环境
此部分主要说明单节点learning 集群的搭建

安装方式：kubeadm  
机器规格：2c4g * 1台   
节点系统：Centos 7.6  
K8s版本：1.18.5  
容器运行时：docker 19.03  
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

[root@test ~]# sudo yum install docker-ce-19.03.9-3.el7 docker-ce-cli-19.03.9-3.el7

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

关闭防火墙
```bash
[root@test ~]# sudo systemctl stop firewalld
[root@test ~]# sudo systemctl disable firewalld
```

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

## 2 多节点 - 生产环境

此部分主要说明多节点production 集群的搭建

安装方式：kubeadm  
机器规格：2c4g * 5台（k8s01-03: 3master / k8s04-05: 2worker）  
节点系统：Centos 7.6  
K8s版本：1.18.5  
容器运行时：docker 19.03  
网络插件：calico

节点的ip信息如下：
```
k8s01: 10.211.55.48
k8s02: 10.211.55.49
k8s03: 10.211.55.50
k8s04: 10.211.55.51
k8s05: 10.211.55.52
```

{{< hint warning >}}
生产环境一般会有以下改动：
- 多master节点，`etcd`服务满足高可用
- `kubeadm init`时使用config配置文件而不是命令行参数
- 无法访问公网，需要指定内部使用的镜像仓库
- `kubectl`等客户端访问的apiserver endpoint有被Load Balancer负载，满足高可用
{{< /hint >}}

### 2.1 安装容器运行时
在所有5个节点上安装容器运行时，参看1.1章节

### 2.2 安装kubelet、kubeadm、kubectl
在所有5个节点上安装kubelet、kubeadm、kubectl，参看1.2章节

### 2.3 kubeadm初始化集群
准备以下名为`kubeadm-config.yaml`的init配置文件，参数含义参看官方文档[https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta3/](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta3/)
```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.211.55.48
  bindPort: 6443
nodeRegistration:
  name: k8s01
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
  criSocket: /var/run/dockershim.sock
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
clusterName: cluster.local
etcd:
  local:
    imageRepository: "k8s.gcr.io"
    imageTag: "3.4.3-0"
    extraArgs:
      metrics: basic
      election-timeout: "5000"
      heartbeat-interval: "250"
      auto-compaction-retention: "8"
      snapshot-count: "10000"
    serverCertSANs:
      - etcd.kube-system.svc.cluster.local
      - etcd.kube-system.svc
      - etcd.kube-system
      - etcd
    peerCertSANs:
      - etcd.kube-system.svc.cluster.local
      - etcd.kube-system.svc
      - etcd.kube-system
      - etcd
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.233.0.0/18
  podSubnet: 10.222.0.0/18
kubernetesVersion: v1.18.5
controlPlaneEndpoint: 10.211.55.48 # 这里可以换成Load Balancer之类的地址
certificatesDir: /etc/kubernetes/pki
imageRepository: k8s.gcr.io # 这里可以替换成实际的镜像仓库
apiServer:
  extraArgs:
    alsologtostderr: "true"
    logtostderr: "false"
    anonymous-auth: "True"
    authorization-mode: Node,RBAC
    bind-address: 0.0.0.0
    insecure-port: "0"
    apiserver-count: "3"
    endpoint-reconciler-type: lease
    service-node-port-range: 30000-32767
    profiling: "False"
    request-timeout: "1m0s"
    enable-aggregator-routing: "False"
    storage-backend: etcd3
    allow-privileged: "true"
    feature-gates: RotateKubeletServerCertificate=True
  extraVolumes:
  - name: basic-time
    hostPath: /etc/localtime
    mountPath: /etc/localtime
    pathType: File
  certSANs:
  - kubernetes
  - kubernetes.default
  - kubernetes.default.svc
  - kubernetes.default.svc.cluster.local
  - 10.233.0.1
  - localhost
  - 127.0.0.1
  - localhost6
  - ::1
  - k8s01 # 这里修改实际的主机名
  - k8s02
  - k8s03
  - lb-apiserver.kubernetes.local
  - 10.211.55.48 # 这里修改实际的ip
  - 10.211.55.49
  - 10.211.55.50
  timeoutForControlPlane: 5m0s
controllerManager:
  extraArgs:
    alsologtostderr: "true"
    logtostderr: "false"
    experimental-cluster-signing-duration: 87600h0m0s
    node-monitor-grace-period: 40s
    node-monitor-period: 5s
    pod-eviction-timeout: 5m0s
    profiling: "False"
    terminated-pod-gc-threshold: "12500"
    bind-address: 0.0.0.0
    feature-gates: RotateKubeletServerCertificate=True
    configure-cloud-routes: "false"
  extraVolumes:
  - name: basic-time
    hostPath: /etc/localtime
    mountPath: /etc/localtime
    pathType: File
scheduler:
  extraArgs:
    alsologtostderr: "true"
    logtostderr: "false"
    bind-address: 0.0.0.0
    feature-gates: RotateKubeletServerCertificate=True
  extraVolumes:
  - name: basic-time
    hostPath: /etc/localtime
    mountPath: /etc/localtime
    pathType: File
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
bindAddress: 0.0.0.0
clientConnection:
 acceptContentTypes: 
 burst: 10
 contentType: application/vnd.kubernetes.protobuf
 kubeconfig: 
 qps: 5
clusterCIDR: 10.222.0.0/18
configSyncPeriod: 15m0s
conntrack:
 maxPerCore: 32768
 min: 131072
 tcpCloseWaitTimeout: 1h0m0s
 tcpEstablishedTimeout: 24h0m0s
enableProfiling: False
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: k8s01
iptables:
 masqueradeAll: False
 masqueradeBit: 14
 minSyncPeriod: 0s
 syncPeriod: 30s
metricsBindAddress: 0.0.0.0:10249
mode: iptables
nodePortAddresses: []
oomScoreAdj: -999
portRange: 
udpIdleTimeout: 250ms
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
clusterDNS:
- 10.233.0.10
```
执行以下命令初始化集群
```bash
[root@k8s01 ~]# kubeadm init --config kubeadm-config.yaml --upload-certs

...
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 10.211.55.48:6443 --token 0s796x.et4cftvejauqdgl9 \
    --discovery-token-ca-cert-hash sha256:f9b349c27ce2e7f4e7a6e4f2be418831e42b2af848e043202368175116a780c1 \
    --control-plane --certificate-key 1386988aa396f30129170fd3eb3215d4ff0ef27f94afb03aaddd873f667ca7a1
...
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.211.55.48:6443 --token 0s796x.et4cftvejauqdgl9 \
    --discovery-token-ca-cert-hash sha256:f9b349c27ce2e7f4e7a6e4f2be418831e42b2af848e043202368175116a780c1
```
{{< hint warning >}}
命令行参数的含义如下：
- `--upload-certs`: 的含义是让kubeadm管理证书，会将证书加密作为Secret上传至kube-system  
这样可以省去手动将ca证书、sa密钥传到其他control plane节点的操作
> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#manual-certs

- `--token`: 是一个低权限的token，kubelet使用bootstrap token连接到kube-apiserver向其申请证书，也是省去了手动给kubelet签署证书的步骤
然后kube-controller-manager给kubelet动态签署证书，后续kubelet都将通过动态签署的证书与kube-apiserver进行双向认证TLS通信，证书存在kubelet的kubeconfig文件中
> https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/

- `--discovery-token-ca-cert-hash`: CA根证书中公钥的摘要值，用于验证control plane提供CA根证书

- `--certificate-key`: 就是用于解密`--upload-certs`上传的加密证书的密钥，此密钥有效期2小时
{{< /hint >}}

### 2.4 其他节点加入集群

#### 2.4.1 control plane节点
准备以下名为`kubeadm-controlplane.yaml`的join文件，用init时命令的输出替换下面内容中的token/caCertHashes/certificateKey
```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: 10.211.55.48:6443
    token: yc9xno.aw40wwrnzjbv49mh # 实际替换
    caCertHashes:
    - sha256:8a6dcb0070e1acf6b9b42ab16fbd288e2f068b3d05d528bd128b9469df68a862 # 实际替换
  timeout: 5m0s
controlPlane:
  localAPIEndpoint:
    advertiseAddress: 10.211.55.49
    bindPort: 6443
  certificateKey: 63430f0e38dbed8fe8523ba7a8f11b708fcde4c4cbcf39109512e7304b4bbc9d # 实际替换
nodeRegistration:
  name: k8s02
  criSocket: /var/run/dockershim.sock
```
执行以下命令以control plane节点加入集群
```bash 
[root@k8s02 ~]# kubeadm join --config kubeadm-controlplane.yaml
...
This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.
```

#### 2.4.2 worker节点
准备以下名为`kubeadm-worker.yaml`的join文件，用init时命令的输出替换下面内容中的token/caCertHashes/certificateKey
```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: 10.211.55.48:6443
    token: yc9xno.aw40wwrnzjbv49mh # 实际替换
    caCertHashes:
    - sha256:8a6dcb0070e1acf6b9b42ab16fbd288e2f068b3d05d528bd128b9469df68a862 # 实际替换
  timeout: 5m0s
nodeRegistration:
  name: k8s04
  criSocket: /var/run/dockershim.sock
```
执行以下命令以worker节点加入集群
```bash 
[root@k8s04 ~]# kubeadm join --config kubeadm-worker.yaml
...
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
```

### 2.4 安装网络插件

参看1.4章节，注意修改`custom-resources.yaml`中cidr: 10.222.0.0/18

## 3 其他操作

### 3.1 修改主机名重新加入集群
修改主机名会使node变为NotReady状态

需要`kubeadm reset`撤销`kubeadm join`的操作，然后修改`kubeadm-worker.yaml`中的`nodeRegistration.name`

如果修改主机名涉及到证书修改，参看以下命令重新生成证书后重新执行`kubeadm join`：
```bash
# 重新生成证书
[root@k8s01 ~]# mv /etc/kubernetes/pki/apiserver.{crt,key} ~
[root@k8s01 ~]# kubeadm init phase certs apiserver --config kubeadm-config.yaml
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local localhost localhost6 k8s01 k8s02 k8s04 lb-apiserver.kubernetes.local] and IPs [10.233.0.1 10.211.55.48 10.211.55.48 10.233.0.1 127.0.0.1 ::1 10.211.55.48 10.211.55.49 10.211.55.50]

# 上传证书
[root@k8s01 ~]# kubeadm init phase upload-certs --upload-certs
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
b8bdfa93e2c229edb5df2f4d9227fceed392e8975c9f148e522ed088a59c34a7

# 顺带更新配置
[root@k8s01 ~]# kubeadm init phase upload-config all --config kubeadm-config.yaml
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
```

> https://blog.scottlowe.org/2019/07/30/adding-a-name-to-kubernetes-api-server-certificate/

### 3.2 升级集群
升级集群以满足新的特性，比如从`1.18.5`升级到`1.19.5`

一般升级的思路是：
- 先升级主control plane节点
- 再升级其他所有control plane节点
- 最后升级所有worker节点

详细参看[https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

#### 3.2.1 升级主control plane节点

##### 3.2.1.1 kubeadm upgrade
```bash
[root@k8s01 ~]# yum install -y kubeadm-1.19.5-0 --disableexcludes=kubernetes

[root@k8s01 ~]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.5", GitCommit:"e338cf2c6d297aa603b50ad3a301f761b4173aa6", GitTreeState:"clean", BuildDate:"2020-12-09T11:16:40Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}

[root@k8s01 ~]# kubeadm upgrade plan
Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
kubelet     2 x v1.18.5   v1.18.20

Upgrade to the latest version in the v1.18 series:

COMPONENT                 CURRENT   AVAILABLE
kube-apiserver            v1.18.5   v1.18.20
kube-controller-manager   v1.18.5   v1.18.20
kube-scheduler            v1.18.5   v1.18.20
kube-proxy                v1.18.5   v1.18.20
CoreDNS                   1.6.7     1.7.0
etcd                      3.4.3-0   3.4.3-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.18.20

_____________________________________________________________________

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
kubelet     2 x v1.18.5   v1.19.16

Upgrade to the latest stable version:

COMPONENT                 CURRENT   AVAILABLE
kube-apiserver            v1.18.5   v1.19.16
kube-controller-manager   v1.18.5   v1.19.16
kube-scheduler            v1.18.5   v1.19.16
kube-proxy                v1.18.5   v1.19.16
CoreDNS                   1.6.7     1.7.0
etcd                      3.4.3-0   3.4.13-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.19.16

Note: Before you can perform this upgrade, you have to update kubeadm to v1.19.16.

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________

[root@k8s01 ~]#  kubeadm upgrade apply v1.19.5
...
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.19.5". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

##### 3.2.1.2 驱逐节点安装kubelet
```bash
[root@k8s01 ~]# kubectl drain k8s01 --ignore-daemonsets

[root@k8s01 ~]# yum install -y kubelet-1.19.5-0 kubectl-1.19.5-0 --disableexcludes=kubernetes

[root@k8s01 ~]# kubectl uncordon k8s01
```

##### 3.2.1.3 更新网络插件
参看1.4章节，如果CNI提供的是daemonsets形式，不需要额外在其他节点执行，本文例子只需要在主control plane节点执行，其他节点不用


#### 3.2.2 升级其他control plane节点
```bash
[root@k8s02 ~]# yum install -y kubeadm-1.19.5-0 --disableexcludes=kubernetes

[root@k8s02 ~]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.5", GitCommit:"e338cf2c6d297aa603b50ad3a301f761b4173aa6", GitTreeState:"clean", BuildDate:"2020-12-09T11:16:40Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}

[root@k8s02 ~]# kubeadm upgrade plan

[root@k8s02 ~]# kubeadm upgrade node

[root@k8s02 ~]# kubectl drain k8s02 --ignore-daemonsets

[root@k8s02 ~]# yum install -y kubelet-1.19.5-0 kubectl-1.19.5-0 --disableexcludes=kubernetes

[root@k8s02 ~]# kubectl uncordon k8s02
```

#### 3.2.3 升级所有worker节点
```bash
[root@k8s04 ~]# yum install -y kubeadm-1.19.5-0 --disableexcludes=kubernetes

[root@k8s04 ~]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.5", GitCommit:"e338cf2c6d297aa603b50ad3a301f761b4173aa6", GitTreeState:"clean", BuildDate:"2020-12-09T11:16:40Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}

[root@k8s04 ~]# kubeadm upgrade node

[root@k8s04 ~]# kubectl drain k8s04 --ignore-daemonsets

[root@k8s04 ~]# yum install -y kubelet-1.19.5-0 kubectl-1.19.5-0 --disableexcludes=kubernetes

[root@k8s04 ~]# kubectl uncordon k8s04
```

{{< hint warning >}}
升级有以下几点注意：
- kubeadm不支持跨minor版本升级，即不能一下就从`1.18.5`升级到`1.20.5`，必须按照`1.18.x -> 1.19.x -> 1.20.x`这样的路径进行升级
- 如果`kubeadm upgrade plan`命令显示说明有组件的配置需要`MANUAL UPGRADE`，则需要在`kubeadm upgrade apply`时通过`--config`参数指定正确的替换配置（本文例子不涉及）
{{< /hint >}}

### 3.3 支持ipv6
关于k8s支持ipv6的历史情况，参看[https://kubernetes.io/blog/2021/12/08/dual-stack-networking-ga/](https://kubernetes.io/blog/2021/12/08/dual-stack-networking-ga/)

主要是几个维度的修改：
- 节点需要有ipv6地址，节点只能能过痛过ipv6访问
- 通过kubeadm修改serviceSubnet和podSubnet支持ipv6
- CNI插件ipv6修改
- Service和Pod的服务支持ipv6

> https://kubernetes.io/docs/concepts/services-networking/dual-stack/
> https://docs.tigera.io/calico/latest/networking/ipam/ipv6#enable-dual-stack
> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/dual-stack-support/


### 3.4 容器运行时切换
从1.20版本开始，kubelet对docker的支持deprecated，会在未来的版本移除掉
[https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#deprecation](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#deprecation)

可以将运行时从docker切换为containerd，详细参看[https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/change-runtime-containerd/](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/change-runtime-containerd/)

### 3.5 忘记了init时的token后续怎么join节点
可以使用存量token自行生成join命令
```bash
[root@k8s01 ~]# kubeadm token list
TOKEN                     TTL         EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS
4tf7yf.3wovx7pygqec6iyv   22h         2023-06-06T20:46:47+08:00   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
7836kw.phyyoeclft6uf9u2   1h          2023-06-05T23:46:54+08:00   <none>                   Proxy for managing TTL for the kubeadm-certs secret        <none>
```

也可以新生成token

针对worker节点
```bash
[root@k8s01 ~]# kubeadm token create --print-join-command
kubeadm join 10.211.55.48:6443 --token k0j9oh.uw4mr15fe0e197ls     --discovery-token-ca-cert-hash sha256:2a19ab5f53a90134832898f905ee85e5b9d238f6728d0186a7f9638a68df9b04
```
针对control plane节点
```bash
# 获取certificateKey
[root@k8s01 ~]# kubeadm init phase upload-certs --upload-certs
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
803fe75e5124f4ce603bf6760a29029c841f71f6953ecafd573b6b769daaab25

[root@k8s01 ~]# kubeadm token create --print-join-command --certificate-key 803fe75e5124f4ce603bf6760a29029c841f71f6953ecafd573b6b769daaab25
kubeadm join 10.211.55.48:6443 --token mjrr1t.x83pef77sp73vyhe     --discovery-token-ca-cert-hash sha256:2a19ab5f53a90134832898f905ee85e5b9d238f6728d0186a7f9638a68df9b04     --control-plane --certificate-key 803fe75e5124f4ce603bf6760a29029c841f71f6953ecafd573b6b769daaab25
```

## 4 内部细节

### 4.1 init phase

此部分讲解安装过程中涉及到的一些内部细节

`init`过程包含了很多个`phase`:
- `preflight`: 做了一些检测，如是否打开了防火墙、是否关闭了swap、容器运行时是否在运行、进行`kubeadm config images pull`拉取镜像等
- `kubelet-start`: 启动kubelet
- `certs`: 生成一堆证书
- `kubeconfig`: 生成了一堆kubeconfig，如`admin.conf`/`kubelet.conf`/`controller-manager.conf`/`scheduler.conf`
- `control-plane`: 生成了静态pod的yaml文件，如`kube-apiserver`/`kube-controller-manager`/`kube-scheduler`
- `etcd`: 生成`etcd`静态yaml文件
- `wait-control-plane`: 等待静态pod运行起来
- `kubelet-check`: 等待kubelet /healthz endpoint返回'ok'
- `apiclient`: 等待所有control plane components变为healthy
- `upload-config`: 将kubeadm的配置文件作为Configmap上传到集群中
- `kubelet`: 创建kubelet Configmap
- `upload-certs`: 将control-plane的证书加密作为kubeadm-certs Secret上传到集群中
- `mark-control-plane`: 把一个node作为control-plane，并打上污点
- `bootstrap-token`: 创建bootstrap token
- `kubelet-finalize`: 更新kubelet相关配置，比如更新kubeconfig kubelet.conf，指向rotatable certificate and key的路径
- `addons`: 启动`CoreDNS`和`kube-proxy`内置插件

> https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#init-workflow
> https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init-phase/

### 4.2 各个组件的作用及关系

- `etcd`: 高可用key/value数据库，用于存储k8s集群涉及到的所有状态数据
- `kube-apiserver`: 暴露API Server，作为frontend为其他组件提供了获取集群共享数据的交换方式，使用`etcd`作为backend
- `kube-controller-manager`: 管理多个controllers(control loops)，这些controller从API Server获取集群当前状态，试图将当前状态变成期望状态，如`replication controller`
- `kube-scheduler`: 将`scheduling queue`中的Pod调度到合适的节点上
- `kubelet`: 运行在每个节点上的agent，对接CNI创建Pod等
- `kube-proxy`: 已`daemonset addon`形式运行在每个节点上，将`Service`概念实现
- `CoreDNS`: 已`deployment addon`形式运行，为集群内部提供对`Pod`和`Service`的DNS解析，使用`etcd`作为backend
- `kubectl`: cli工具，对接API Server操作集群

以创建`Pod`为例，理解不同组件之间怎么通讯的
```
kubectl             ->  kube-apiserver  : 创建Pod
kube-apiserver      ->  etcd            : 存储object定义
controller-manager  ->  kube-apiserver  : 创建Pod资源，更新etcd中的信息，此时kubectl能显示出Pod信息，状态为Pending
scheduler           ->  kube-apiserver  : 为Pod选择节点，更新pod Spec中的nodeName信息，更新etcd中的信息
kubelet             ->  kube-apiserver  : 获取Pod模版信息
kubelet             ->  CRI(docker)     : 创建容器
kubelet             ->  kube-apiserver  : 更新Pod信息
```

> https://kubernetes.io/docs/concepts/overview/components/  
> https://kubernetes.io/docs/reference/command-line-tools-reference/  
> https://belowthemalt.com/2022/04/08/kubernetes-cluster-process-flow-of-a-pod-creation/

### 4.3 生成一堆证书的作用

本质为组件之间为双向认证，需要指定提供TLS服务的证书和密钥，以及自身作为client端的证书和密钥，以及验证对方证书的CA证书

详细参看:
> http://www.xuyasong.com/?p=2054#_kubeconfig  
> https://zhuanlan.zhihu.com/p/580400509  
> https://jvns.ca/blog/2017/08/05/how-kubernetes-certificates-work/  
> https://kubernetes.io/docs/setup/best-practices/certificates/  
> https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubeconfig-additional-users

后面会专门写一篇关于认证体系的文章

### 4.4 3 master的作用，master是不是越多越好
master节点为什么推荐3个或5个，主要是受etcd影响

主要是为了容错而不是性能

> https://zhuanlan.zhihu.com/p/567371786  
> https://etcd.io/docs/v3.3/faq/#why-an-odd-number-of-cluster-members  
> https://stackoverflow.com/questions/57667504/why-we-need-more-than-3-master-cluster-for-kubernetes-ha
