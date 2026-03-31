---
title: "K8s 认证授权准入控制"
weight: 3
bookToc: true
---

## K8s 认证授权准入控制

![](/data/image/container/k8s/image-46.png)

### 1.1 认证

认证的本质是在提取`username/group`等信息，且认证阶段对`username/group`透明，交由后续的授权模块处理，认证失败的视为`anonymous user`

![](/data/image/container/k8s/image-47.png)

K8s 中有2类用户：

- 一种是`serviceaccount`：由k8s管理，每个命名空间都有一个默认的 default 服务账户，sa会关联一个secrets（JWT token形式），这些凭据会以挂载的方式注入到 Pod 中的`/var/run/secrets/kubernetes.io/serviceaccount` 路径，使得容器内的程序能够与 Kubernetes API 通信，叫做pod in-cluster的通讯

  > 一般使用SA做in-cluster的通讯，但是在集群外部使用sa token 也可以调用API Server，只是API Server的一种认证方式

- 另外一种是`normal users`：正常用户在k8s中并没有实际的对象存在，无法通过API添加正常用户，是由外部合法的证书管理的。任何提供由集群证书颁发机构 (CA) 签名的有效证书的用户都被视为合法的、认证通过的用户，k8s从证书中的`CN common name`，`O organization` 提取`username`和`group`

  > 一般使用证书搭配kubectl，于外部操作集群

---

kubectl 可以用于本地连接远程K8s API Server，需要指定kubeconfig，kubeconfig文件中包含API Server地址证书文件等信息

![](/data/image/container/k8s/image-48.png)

我们可以通过以下命令验证，`kubeconfig` 文件中的客户端证书是由集群ca颁发的合法证书：

```bash
# 查看证书的organization和common name信息
openssl x509 -inform pem -noout -text -in '.minikube/profiles/minikube/client.crt'

# client证书中的颁发者对应集群ca证书的subject
openssl x509 -in client.crt -noout -issuer
openssl x509 -in ca.crt -noout -subject

# 使用ca证书中的公钥进行验证，验证client.crt是否有ca颁发
openssl verify -CAfile ca.crt client.crt
```

![](/data/image/container/k8s/image-49.png)

![](/data/image/container/k8s/image-50.png)

我们也可以使用ca证书及其对应的私钥，自行颁发证书，用于访问集群

```bash
openssl genrsa -out zqq.key 2048
openssl req -new -key zqq.key -out zqq.csr -subj "/O=system:masters/CN=zqq"
openssl x509 -req -in zqq.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out zqq.crt -days 365
```

![](/data/image/container/k8s/image-51.png)

![](/data/image/container/k8s/image-52.png)

> 注意这里指定的organization即用户对应的群组为system:masters，这个群组为默认的K8s管理员群组，**拥有整个集群的管理权**限。实际生成环境中应通过RBAC等方式赋予合适的权限，而避免将用户直接加入到该群组。

---

K8s会把关联的 ServiceAccount 的身份信息（JWT Token、CA 证书）自动挂载进 Pod（默认路径为 `/var/run/secrets/kubernetes.io/serviceaccount/`），如果 Pod 没有显式关联 ServiceAccount，K8s 会自动将其绑定到期命名空间默认的 `default` ServiceAccount。

该 Token 可以被用于访问 Kubernetes API Server，身份为：`system:serviceaccount:(NAMESPACE):(SERVICEACCOUNT)`，并自动加入群组`system:serviceaccounts`和`system:serviceaccounts:<namespace>`

```bash
kubectl exec -it minio-2 -- bash

cat /var/run/secrets/kubernetes.io/serviceaccount/token
# 将token复制出来
```

![](/data/image/container/k8s/image-53.png)

可以通过 [https://jwt.io/](https://jwt.io/) 网站验证Pod中的token来自于集群`sa.key`颁发（在网站中复制粘贴公钥`sa.pub`文件内容，用于验证签名）

![](/data/image/container/k8s/image-54.png)

`token`复制到网站左侧

![](/data/image/container/k8s/image-55.png)

`sa.pub`文件内容复制到右下角

![](/data/image/container/k8s/image-56.png)

可以看出pod中的token为`sa.key`签名

---

我们可以使用SA token在Pod内部访问集群API Server

```bash
kubectl exec -it minio-2 -- bash

NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

echo $NAMESPACE

# 默认无任何权限
curl -s --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
     -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
     https://kubernetes.default.svc/api/v1/namespaces/${NAMESPACE}/pods

# 另外起一个窗口，添加查看权限
kubectl create clusterrolebinding default-view --clusterrole=view --serviceaccount=default:default

# 有权限后
curl -s --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
     -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
     https://kubernetes.default.svc/api/v1/namespaces/${NAMESPACE}/pods
```

![](/data/image/container/k8s/image-57.png)


> 默认情况下，默认的 ServiceAccount（default）没有任何权限，除非你显式地给它配置了 RBAC 绑定。这是 K8s 的默认安全策略设计，用于最小化权限。
>
> 这里只是做测试，实际涉及权限的时候，你要很清楚地知道背后发生了什么。

### 1.2 授权

![](/data/image/container/k8s/image-58.png)

> RBAC（Role-Based Access Control）模块在 Kubernetes 中会返回 Allow 或 NoOpinion，不会返回 Deny，无法在 RBAC 中明确说“禁止某某操作”。

---

我们这里重点介绍RBAC的授权方式。

基于角色的访问控制 (Role-based access control, RBAC) 是一种根据组织内各个用户的角色来规范对计算机或网络资源的访问的方法。K8s实现RBAC涉及以下4种API资源：

![](/data/image/container/k8s/image-59.png)

> RoleBinding 可以引用 ClusterRole ，只是告诉 API Server：这个权限只在这个 namespace 里生效，相当于复用权限了。

---

看2个具体的例子：

以下yaml，赋予用户zqq，在 default 命名空间中读取 Pods 的权限

![](/data/image/container/k8s/image-60.png)

以下yaml，赋予manager群组，在整个集群中读取 Secrets 的权限

![](/data/image/container/k8s/image-61.png)

再看一道CNCF CKA认证关于RBAC的真题：

![](/data/image/container/k8s/image-62.png)

相信你应该能解答出来吧

---

如果ClusterRole包含的资源是cluster-scoped，如nodes，则RoleBinding 只能限制 **namespace-scoped** 权限的作用范围，仍然可以访问所有节点（nodes）资源。这个注意事项，我们通过实验详细说明

以下yaml：

1. 创建一个 ClusterRole，允许 list 所有节点（nodes）
2. 在 default 命名空间创建一个 ServiceAccount
3. 用 RoleBinding（注意是 RoleBinding，不是 ClusterRoleBinding）将这个 ServiceAccount 绑定到上述 ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: list-nodes
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-list-node
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-list-nodes
  namespace: default
subjects:
- kind: ServiceAccount
  name: sa-list-node
  namespace: default
roleRef:
  kind: ClusterRole
  name: list-nodes
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rbac.yaml

# node无namespace概念
kubectl auth can-i list nodes --as=system:serviceaccount:default:sa-list-node

# pod有namespace概念
kubectl auth can-i list pods --namespace=default     --as=system:serviceaccount:default:sa-list-node
kubectl auth can-i list pods --namespace=kube-system --as=system:serviceaccount:default:sa-list-node
```

![](/data/image/container/k8s/image-63.png)

>  虽然使用的是 RoleBinding（限定了 namespace），但绑定的是 ClusterRole 中的 cluster-scoped 资源（nodes）；所以这个绑定实际上还是授予了该 SA 访问整个集群中所有 Node 的 list 权限！

### 1.3 准入控制

![](/data/image/container/k8s/image-64.png)

![](/data/image/container/k8s/image-65.png)


> Mutating 插件先于 Validating 插件调用，因为 Mutating 插件可能会改变 Validating 插件的判断结果，如果顺序反了，系统校验的对象就不是最终要落地的版本

有一些默认的准入控制器，实现了一些逻辑比如AlwaysPullImages，更新Pod的 `imagePullPolicy`为`Always`

这里的Webhook 是指按照一定规范调用外部HTTP API，可以通过编写外部服务实现以下需求，涉及到编写具体的代码，这里就不展开讲：

- 镜像验证签名
- 添加sidecar容器
- `replicas = 1` 不允许准入