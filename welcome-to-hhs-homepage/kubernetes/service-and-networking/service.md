---
layout:
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: false
  outline:
    visible: true
  pagination:
    visible: false
---

# Service

## 一、为什么要定义Service服务

在kubernetes中，Pod是有生命周期的，如果Pod重启它的IP很有可能会发生变化。如果我们的服务都是将Pod的IP地址写死，Pod挂掉或者重启，和刚才重启的pod相关联的其他服务将会找不到它所关联的Pod，为了解决这个问题，在kubernetes中定义了service资源对象，Service 定义了一个服务访问的入口，客户端通过这个入口即可访问服务背后的应用集群实例，service是一组Pod的逻辑集合，这一组Pod能够被Service访问到，通常是通过[Label Selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)实现的。

### 1、Service的概述

_**将在集群中运行的应用通过同一个面向外界的端点公开出去，即使工作负载分散于多个后端也完全可行**_

<figure><img src="../../../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

### 2、Service的工作原理

k8s在创建Service时，会根据标签选择器selector(lable selector)来查找Pod，据此创建与Service同名的endpoint对象，当Pod 地址发生变化时，endpoint也会随之发生变化，service接收前端client请求的时候，就会通过endpoint，找到转发到哪个Pod进行访问的地址。(至于转发到哪个节点的Pod，由负载均衡kube-proxy决定)

### 3、Service的三种网络模式

#### 1、Node Network（节点网络）

#### 2、Pod network（pod 网络）

#### 3、Cluster Network（集群地址）

### 4、service、ingress、gateway的关系

| 特性         | Service                         | Ingress                                      | Gateway                                     |
| ---------- | ------------------------------- | -------------------------------------------- | ------------------------------------------- |
| **作用范围**   | 在集群内提供稳定访问和负载均衡                 | 提供 HTTP/HTTPS 入口和路由（将外部请求路由到集群中的不同服务）        | 实现跨协议的高级流量管理和控制（限流、重试、超时）                   |
| **暴露方式**   | ClusterIP、NodePort、LoadBalancer | 需要 Ingress Controller（Nginx、Traefik 或 Istio） | 需要 Gateway Controller（Kong、API、Istio控制网格流量） |
| **支持协议**   | TCP/UDP                         | HTTP/HTTPS                                   | HTTP/HTTPS、TCP、gRPC 等多种协议                   |
| **负载均衡**   | Pod 内部负载均衡                      | 基于域名和路径的负载均衡                                 | 提供更灵活的路由和负载策略                               |
| **典型使用场景** | 服务之间或对外提供稳定的 IP                 | 对外 HTTP/HTTPS 流量路由                           | API 网关、跨集群服务调用                              |

## 二、Service的资源清单

### 1、spec字段

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

### 2、spec.ports字段

&#x20;  appProtocol     # 描述_**应用层**_协议（如 HTTP、HTTPS、gRPC 等）

&#x20;  name   #定义端口的名字

&#x20;  nodePort    #service在物理机映射的端口，默认在 30000-32767 之间

&#x20;  port      #service的端口，这个是k8s集群内部服务可访问的端口（必选参数）

&#x20;  protocol    # 指定端口使用的协议（例如 TCP、UDP、SCTP）_**传输层**_

&#x20;  targetPort   # targetPort是pod上的端口，从port和nodePort上来的流量，经过kube-proxy流入到后端pod的targetPort上，最后进入容器

## 三、Service的使用案例

### 1、clusterIP

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80  #pod中的容器需要暴露的端口
        startupProbe:
           periodSeconds: 5
           initialDelaySeconds: 60
           timeoutSeconds: 10
           httpGet:
             scheme: HTTP
             port: 80
             path: /
        livenessProbe:
           periodSeconds: 5
           initialDelaySeconds: 60
           timeoutSeconds: 10
           httpGet:
             scheme: HTTP
             port: 80
             path: /
        readinessProbe:
           periodSeconds: 5
           initialDelaySeconds: 60
           timeoutSeconds: 10
           httpGet:
             scheme: HTTP
             port: 80
             path: /
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: ClusterIP
  ports:
  - port: 80   #service的端口，暴露给k8s集群内部服务访问
    protocol: TCP
    targetPort: 80    #pod容器中定义的端口
  selector:
    run: my-nginx  #选择拥有run=my-nginx标签的pod
```

创建一个 Service，具有标签run=my-nginx的Pod，目标TCP端口 80，并且在一个抽象的Service端口（targetPort：容器接收流量的端口；port：抽象的 Service 端口，可以使任何其它 Pod访问该 Service 的端口）上暴露。

工作原理：

service可以对外提供统一固定的ip地址，并将请求重定向至集群中的pod。其中“将请求重定向至集群中的pod”就是通过endpoint与selector协同工作实现。selector是用于选择pod，由selector选择出来的pod的ip地址和端口号，将会被记录在endpoint中。endpoint便记录了所有pod的ip地址和端口号。当一个请求访问到service的ip地址时，就会从endpoint中选择出一个ip地址和端口号，然后将请求重定向至pod中。具体把请求代理到哪个pod，需要的就是kube-proxy的轮询实现的。service不会直接到pod，service是直接到endpoint资源，就是地址加端口，再由endpoint再关联到pod。

service只要创建完成，我们就可以直接解析它的服务名，每一个服务创建完成后都会在集群dns中动态添加一个资源记录，添加完成后我们就可以解析了，资源记录格式是：

SVC\_NAME.NS\_NAME.DOMAIN.LTD.

服务名.命名空间.域名后缀

集群默认的域名后缀是svc.cluster.local.

就像我们上面创建的my-nginx这个服务，它的完整名称解析就是my-nginx.default.svc.cluster.local

### 2、**NodePort**

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx-nodeport
spec:
  selector:
    matchLabels:
      run: my-nginx-nodeport
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx-nodeport
    spec:
      containers:
      - name: my-nginx-nodeport-container
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-nodeport
  labels:
    run: my-nginx-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30380
  selector:
    run: my-nginx-nodeport
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

default名称空间下的client 服务想要访问nginx-ns名称空间下的nginx-svc服务

#### 1、创建nginx服务

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: nginx-ns
spec: 
  replicas: 1
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
       image: nginx
       imagePullPolicy: IfNotPresent
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: nginx-ns
spec:
  selector:
    app: nginx
  ports:
   - name: http
     protocol: TCP
     port: 80
     targetPort: 80
```

#### 2、创建client服务

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client
spec: 
  replicas: 1
  selector:
    matchLabels:
      app: busybox
  template:
   metadata:
    labels:
      app: busybox
   spec:
     containers:
     - name: busybox
       image: busybox
       imagePullPolicy: IfNotPresent
       command: ["/bin/sh","-c","sleep 36000"]
---
apiVersion: v1
kind: Service
metadata:
  name: client-svc
spec:
  type: ExternalName
  externalName: nginx-svc.nginx-ns.svc.cluster.local
  ports:
  - name: http
    port: 80
    targetPort: 80
```

#### 3、验证结果

登录到client pod

```
 kubectl exec -it  client-76b6556d97-xk7mg -- /bin/sh
/ # wget -q -O - client-svc.default.svc.cluster.local
    wget -q -O - nginx-svc.nginx-ns.svc.cluster.local
```

## 四、Service服务发现：coredns组件详解

### 1、DNS是什么？

DNS全称是Domain Name System：域名系统，是整个互联网的电话簿，它能够将可被人理解的域名翻译成可被机器理解IP地址，使得互联网的使用者不再需要直接接触很难阅读和理解的IP地址。域名系统在现在的互联网中非常重要，因为服务器的 IP 地址可能会经常变动，如果没有了 DNS，那么可能 IP 地址一旦发生了更改，当前服务器的客户端就没有办法连接到目标的服务器了，如果我们为 IP 地址提供一个『别名』并在其发生变动时修改别名和 IP 地址的关系，那么我们就可以保证集群对外提供的服务能够相对稳定地被其他客户端访问。DNS 其实就是一个分布式的树状命名系统，它就像一个去中心化的分布式数据库，存储着从域名到 IP 地址的映射。

### 2、CoreDNS？

CoreDNS 其实就是一个 DNS 服务，而 DNS 作为一种常见的服务发现手段，所以很多开源项目以及工程师都会使用 CoreDNS 为集群提供服务发现的功能，Kubernetes 就在集群中使用 CoreDNS 解决服务发现的问题。 作为一个加入 CNCF（Cloud Native Computing Foundation）的服务， CoreDNS 的实现非常简单。

### 3、案例

```
apiVersion: v1
kind: Pod
metadata:
  name: dig
  namespace: default
spec:
  containers:
  - name: dig
    image:  xianchao/dig:latest
    imagePullPolicy: IfnotPresent
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

访问结果

```
❯ kubectl exec -it dig -- nslookup kubernetes
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   kubernetes.default.svc.cluster.local   # FQDN 那么k8s集群内部的服务就可以通过FQDN访问
Address: 10.96.0.1 
```

