# Service

## 一、Service的概述

_**将在集群中运行的应用通过同一个面向外界的端点公开出去，即使工作负载分散于多个后端也完全可行**_

### service、ingress、gateway的关系

| 特性         | Service                         | Ingress                                      | Gateway                                     |
| ---------- | ------------------------------- | -------------------------------------------- | ------------------------------------------- |
| **作用范围**   | 在集群内提供稳定访问和负载均衡                 | 提供 HTTP/HTTPS 入口和路由（将外部请求路由到集群中的不同服务）        | 实现跨协议的高级流量管理和控制（限流、重试、超时）                   |
| **暴露方式**   | ClusterIP、NodePort、LoadBalancer | 需要 Ingress Controller（Nginx、Traefik 或 Istio） | 需要 Gateway Controller（Kong、API、Istio控制网格流量） |
| **支持协议**   | TCP/UDP                         | HTTP/HTTPS                                   | HTTP/HTTPS、TCP、gRPC 等多种协议                   |
| **负载均衡**   | Pod 内部负载均衡                      | 基于域名和路径的负载均衡                                 | 提供更灵活的路由和负载策略                               |
| **典型使用场景** | 服务之间或对外提供稳定的 IP                 | 对外 HTTP/HTTPS 流量路由                           | API 网关、跨集群服务调用                              |

## 二、Service的资源清单

### spec字段

allocateLoadBalancerNodePorts   # 当服务类型是 LoadBalancer 时，决定是否自动为该服务分配 NodePort。默认为true

clusterIP   # 为服务分配的集群内 IP 地址，通常由系统随机分配（"None"：创建无虚拟 IP 的 "Headless Service"，即直接连接到后端 Pod）

clusterIPs    # 服务分配的 IP 地址列表,最多只能有两个 IP 地址，且必须与 ipFamilies 配置对应

externalIPs   # 使集群节点接受该 IP 的流量

externalName    # 设置一个外部引用名称（如 DNS 的 CNAME 记录）不会代理流量

externalTrafficPolicy   # 定义服务通过外部地址（如 NodePort、ExternalIP 或 LoadBalancer IP）分发流量的策略 。"Local"：只路由到本节点上的后端端点，不会改变客户端源 IP，但如果没有本地端点则丢弃流量。"Cluster"（默认）：路由到所有可用端点，可能通过代理改变客户端源 IP。

healthCheckNodePort    # 当externalTrafficPolicy 为 "Local" 时，指定健康检查的 NodePort 端口

internalTrafficPolicy   # 决定集群内 Pod 通过 ClusterIP 访问服务时的流量策略。"Local"：只路由到与客户端 Pod 同一节点上的端点。"Cluster"（默认）：路由到所有端点

ipFamilies    # 指定服务的 IP 协议族，如 IPv4 或 IPv6。用于双栈网络的集群

ipFamilyPolicy   # 定义服务在单栈或双栈网络中的行为。SingleStack：使用单个 IP 协议族。PreferDualStack：优先使用双栈（如果集群支持）。RequireDualStack：强制使用双栈，否则创建失败

loadBalancerClass   # 指定服务所属的负载均衡器实现类（例如 "internal-vip"）只能用于 LoadBalancer 类型的服务

loadBalancerIP   # 指定 LoadBalancer 类型服务的 IP 地址

loadBalancerSourceRanges   # 限制通过云提供商负载均衡器进入的流量，仅允许指定的客户端 IP 访问

ports  # 定义服务暴露的端口列表

publishNotReadyAddresses   # 表示服务是否发布未就绪的地址，常用于 StatefulSet 的 Headless 服务

selector   # 绑定后端Pod

sessionAffinity   # 支持 "ClientIP" 或 "None"，用于会话保持（基于客户端 IP 的连接保持）

sessionAffinityConfig   # 包含会话保持的详细配置

type    # 定义服务的类型。ClusterIP：默认类型，仅限集群内部访问。NodePort：在每个节点上分配端口。LoadBalancer：通过外部负载均衡器暴露服务。ExternalName：将服务映射为外部 DNS 名称。

## 三、Service的使用案例

### 1、clusterIP

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostnames
spec:
  selector:
    matchLabels:
      app: hostnames
  replicas: 3
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: k8s.gcr.io/serve_hostname
        ports:
        - containerPort: 9376
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
```

我们使用的 webservice 镜像为 `k8s.gcr.io/serve_hostname`，其主要提供输出当前服务器的 `hostname` 的功能，这里声明的`Pod`份数是 `3` 份，此时如果我们依次访问 `curl 10.0.1.175:80` 的话，会发现每次响应内容不一样，说明后端请求了不同的 pod 。这是因为 Service 提供的负载均衡方式是 `Round Robin`

### 2、**NodePort**

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 3
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: nginx
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - nodePort: 8080
    targetPort: 80
    protocol: TCP
    name: http
  - nodePort: 443
    protocol: TCP
    name: https
  selector:
    run: my-nginx
```

如果你未指定 `spec.ports.nodePort` 的话，则系统会随机选择一个范围为 `30000-32767` 的端口号使用。

### 3、**LoadBalancer**

<pre><code>apiVersion: apps/v1
kind: Deployment
metadata:
  name: example
spec:
  selector:
    matchLabels:
      app: example
  replicas: 3
  template:
    metadata:
      labels:
        app: example
    spec:
      containers:
      - name: example
        image: nginx
<strong>---
</strong><strong>kind: Service
</strong>apiVersion: v1
metadata:
  name: example-service
spec:
  ports:
  - port: 8765
    targetPort: 9376
  selector:
    app: example
  type: LoadBalancer
</code></pre>

使用云提供商的负载均衡器向外部暴露服务。 外部负载均衡器可以将流量路由到自动创建的 `NodePort` 服务和 `ClusterIP` 服务上，创建这个服务后，系统会自动创建一个外部负载均衡器，其端口为8765, 并且把被代理的地址添加到公有云的负载均衡当中

### 4、**ExternalName**

```
 apiVersion: v1
 kind: Service
 metadata:
   name: my-service
 spec:
   type: ExternalName
   externalName: my.database.example.com
```

指定了一个 `externalName=my.database.example.com` 的字段。而且你应该会注意到，这个 YAML 文件里不需要指定 selector。 这时候，当你通过 Service 的 DNS 名字访问它的时候，比如访问：`my-service.default.svc.cluster.local`。 那么，Kubernetes 为你返回的就是 `my.database.example.com`。所以说，`ExternalName` 类型的 Service，其实是在 `kube-dns` 里为你添加了一条 CNAME 记录。 这时访问 `my-service.default.svc.cluster.local` 就和访问 `my.database.example.com` 这个域名是一个效果了。

_**访问**** ****`my-service`**** ****的方式与访问其他 Service 的方式相同， 主要区别在于重定向发生在 DNS 级别，而不是通过代理或转发来完成。**_
