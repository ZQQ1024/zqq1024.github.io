---
title: "K8s Basic"
weight: 1
bookToc: true
---

## 1 K8s概述

### 1.1 容器

下图展示了**传统部署、虚拟化部署和容器部署**的架构演进：

![](/data/image/container/k8s/image.png)

- **传统部署**：应用直接运行在操作系统之上，所有应用共享同一套操作系统环境。这种方式结构简单，但应用间缺乏隔离，容易因依赖冲突或资源竞争导致系统不稳定，同时扩展和迁移也较为困难。
- **虚拟化部署**：在宿主操作系统上通过 `Hypervisor` 运行多个虚拟机，每个虚拟机包含完整的操作系统和所需运行环境，实现了应用间的强隔离，并能在同一物理服务器上运行不同类型的操作系统，虚拟化技术能够更好地利用物理服务器的资源。然而，虚拟机启动较慢且资源开销大，每台虚拟机需要独立的操作系统，占用较多内存和存储空间。

- **容器部署**：容器直接运行在宿主操作系统的容器运行时之上，共享内核但隔离文件系统、网络和进程空间。与虚拟机相比，容器更加轻量化，启动速度快、资源利用率高，便于快速扩展和迁移，非常适合微服务和云原生应用的部署。但容器的隔离性较虚拟机弱，不适合运行完全不同内核需求的应用场景。

容器使用了 `linux namespace`、`cgroup `等技术，在同一个操作系统上为不同应用创建了各自独立的运行环境，使得每个应用从自身角度看起来独立拥有了操作系统。

![](/data/image/container/k8s/image-1.png)

### 1.2 容器编排

容器是打包和运行应用程序的好方式，越来越多的业务系统利用容器来搭建部署。随着容器技术越来越多的使用，也出现了很多问题，如：

- 成百上千的容器管理问题
- 容器如何通信
- 如何协调和调度这些容器
- 如何在升级应用程序时不会中断服务
- 如何监视应用程序的运行状况

于是乎，市场上就出现了一批容器编排工具，典型的是 `Docker Swarm` 和 `K8s`。最后，`K8s`几乎成了当前容器编排的事实标准。

![](/data/image/container/k8s/image-2.png)

> `Kubernetes` 这个名字源于希腊语，意为`“舵手”`或`“飞行员”`。K8s 这个缩写是因为 K 和 s 之间有 8 个字符的关系。

K8s与Docker的关系是：K8s 是编排层，Docker 是运行层。K8s在很长一段时间内开发了 **dockershim** 作为适配层，使用 **docker** 在不兼容 **Container Runtime Interface（CRI）** 的情况下作为运行层使用。CRI 的存在是让 `kubelet` 与不同容器运行时进行解耦。K8s 社区在 **1.20 版本宣布弃用 dockershim**，并在 **1.24 完全移除**，推荐使用 containerd 或 CRI-O。

![](/data/image/container/k8s/image-3.png)

### 1.3 K8s 组件

K8s架构图如下：

![](/data/image/container/k8s/image-4.png)


K8s 的架构上传统可以分为 **Master 节点（控制平面）** 和 **Worker 节点（工作节点）**，但在新版本的官方文档中，这种叫法逐渐被替换为 **Control Plane 节点** 和 **Node 节点**。



**Control Plane 节点** 负责整个集群的管理和调度，维护集群的期望状态。主要组件包括：

- **kube-apiserver**：提供 Kubernetes API，是集群的统一入口。
- **etcd**：分布式键值存储，保存集群的所有配置信息和状态数据。
- **kube-scheduler**：负责将未调度的 Pod 分配到合适的 Worker 节点。
- **kube-controller-manager**：运行一系列控制器，维持集群实际状态与期望状态一致。
- （可选）**cloud-controller-manager**：处理与云平台相关的控制逻辑。

> Control Plane  节点通常不运行应用工作负载，只运行控制平面组件（但也可以配置让其参与运行 Pod）。

**Node 节点** 实际运行容器化应用工作负载。主要组件包括：

- **kubelet**：节点代理，负责与 API Server 通信并管理本节点的 Pod 生命周期。
- **kube-proxy**：负责网络代理和负载均衡，实现 Service 的虚拟 IP 和流量转发。
- **容器运行时（containerd/CRI-O/Docker）**：负责运行具体的容器。

> Worker 节点通过 kubelet 将运行状态汇报给控制平面，并接收调度过来的 Pod

参考以下了解K8s中的相关术语概念

> https://kubernetes.io/docs/reference/glossary/?fundamental=true

### 1.4 航海隐喻

云原生领域相关词语都或多或少与航海有关，用现实中的航海运输体系比喻抽象的容器与云原生技术架构。

![](/data/image/container/k8s/image-5.png)

- **Container 与集装箱**

  - Container 最早指 **海运集装箱**，用于把货物标准化封装后放到船上运输，不用关心内部装了什么，极大提升了运输效率

  - 容器技术借鉴了这一概念：**应用及其依赖被封装在标准化镜像中**，可以跨环境（开发、测试、生产）快速运输和部署

  - 这种跨平台、标准化的特性与航运集装箱高度相似

- **Docker 与码头工人**

  - Docker 的 logo 是一只 **鲸鱼驮着集装箱**，象征将容器装载运输

  - Docker 的角色类似“码头工人”，负责**打包、搬运和运行容器**

- **Harbor 与港口**
  - Harbor 作为镜像仓库，用于**集中存放和分发容器镜像**，就像港口存放和分发集装箱
  - “港口”象征镜像的中转站和安全存放点

- **Kubernetes 与舵手/舰队管理**

  - Kubernetes 名称源自希腊语 **“κυβερνήτης” (kybernētēs)**，意为“舵手”或“船长”

  - 它负责**调度和管理容器集群**，如同指挥舰队航行

- **Helm 与船舵**

  - Helm 直译为“舵轮”，是 Kubernetes 的包管理工具

  - 作用类似船舵，**简化航向控制**，方便部署和管理复杂应用

- **Istio 与帆船**

  - Istio 名字灵感来自希腊语 **ἵστιον (ístion)**，意为“帆”

  - 帆是驱动帆船在海上航行的关键工具，Istio 是 **服务网格（Service Mesh）**，其功能就像“帆”赋予舰船灵活的航行能力

### 1.5 声明式与控制器

![](/data/image/container/k8s/image-6.png)

K8s 的核心理念就是 **声明式 API（Declarative API）** 与 **控制器（Controller）** 的结合，声明式API 不关心执行过程，只定义最终的**期望状态**。控制器是 K8s 的核心组件之一，通过 **控制循环（Control Loop）** 监控集群实际状态，根据 **期望状态（Desired State）** 和 **实际状态（Current State）** 之间的差异，自动执行调整操作，使两者保持一致。

---

类似在机器人和自动化领域，控制循环（Control Loop）是一种调节系统状态的非终止回路。下图是给鹦鹉用的恒温箱，如设置预期状态为 30度，恒温器会朝着维持在30度而努力，当温度大于32度灭灯，当温度小于28度亮灯。

![](/data/image/container/k8s/image-7.png)

Kubernetes 中默认内置的一些控制器示例包括：**ReplicaSet Controller、Endpoints Controller、Namespace Controller 和 ServiceAccounts Controller**

各个 Controller 的简要解释如下：

- `ReplicaSet Controller`：确保指定数量的 Pod 副本始终运行。Deployment 会创建和管理 ReplicaSet，由 ReplicaSet Controller 具体执行副本数维护
- `Endpoints Controller`：负责根据当前集群中 Service 和 Pod 的匹配关系，动态生成和更新 Endpoints 资源（即某个 Service 对应的 Pod IP 列表）
- `Namespace Controller`：管理 Namespace 生命周期，比如当你删除一个 namespace 时，它会递归删除其中的所有资源
- `ServiceAccounts Controller`：自动为每个 namespace 创建 default 的 ServiceAccount，并确保每个 Pod 都能关联到合适的账户（用于 API 鉴权）

---

以下yaml通过声明式的方式定义了一个 Deployment 对象，名字为 `nginx-deployment`，通过 `replicas: 3` 指定期望状态：始终运行 3 个 Nginx Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

![](/data/image/container/k8s/image-8.png)

### 1.6 总结

了解以上概念能够帮助我们更好的在日常工作中使用K8s，我们尝试给日常能够基本使用K8s的本质做个总结：

**使用 K8s的本质，是通过kubectl等客户端操作 API 资源，通过配置文件如yaml声明你想要的状态，而K8s通过调度、控制器、kubelet 等组件自动帮你达到预期状态**。具体对应以下几点：

![](/data/image/container/k8s/image-9.png)