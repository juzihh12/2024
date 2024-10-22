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

# Gateway

## 一、gateway的概述

Gateway API 具有三种稳定的 API 类别：

* **GatewayClass：** 定义一组具有配置相同的网关，由实现该类的控制器管理。
* **Gateway：** 定义流量处理基础设施（例如云负载均衡器）的一个实例。
* **HTTPRoute：** 定义特定于 HTTP 的规则，用于将流量从网关监听器映射到后端网络端点的表示。 这些端点通常表示为 [Service](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/)。

&#x20;一个 Gateway 对象只能与一个 GatewayClass 相关联；GatewayClass 描述负责管理此类 Gateway 的网关控制器。 各个（可以是多个）路由类别（例如 HTTPRoute）可以关联到此 Gateway 对象。 Gateway 可以对能够挂接到其 `listeners` 的路由进行过滤，从而与路由形成双向信任模型。

<figure><img src="../../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

一个 Kubernetes 集群管理员在 `Infra` 命名空间中部署了一个名为 `shared-gw` 的 `Gateway`，供不同的应用团队使用，以便将其应用暴露在集群之外。团队 A 和团队 B（分别在命名空间 "A" 和 "B" 中）将他们的 `Route` 绑定到这个 `Gateway`。它们互不相识，只要它们的 `Route` 规则互不冲突，就可以继续隔离运行。团队 C 有特殊的网络需求（可能是性能、安全或关键性），他们需要一个专门的 `Gateway` 来代理他们的应用到集群外。团队 C 在 "C" 命名空间中部署了自己的 `Gateway` `specialive-gw`，该 Gateway 只能由 "C" 命名空间中的应用使用。

<figure><img src="../../../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

## 二、yaml详解

通过网关将 HTTP 流量路由到集群内部的服务。两个资源组合起来控制**外部流量如何进入集群，并路由到具体的服务**。

```
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway  # Gateway 名称
spec:
  selector:
    istio: ingressgateway  # 选择使用哪个 Istio Ingress Gateway 组件
  servers:
  - hosts: 
    - "*"  # 接受来自所有主机名（域名）的请求
    port:
      name: http  # 端口的名称
      number: 80  # 使用 HTTP 的 80 端口
      protocol: HTTP  # 协议为 HTTP
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: bookinfo-vs  # VirtualService 名称
spec:
  hosts:
  - "*"  # 接受所有主机的请求
  gateways:
  - bookinfo-gateway  # 绑定到上面定义的 bookinfo-gateway
  http:
  - match:
    - uri: 
        exact: /productpage  # 精确匹配 /productpage 的请求
    - uri: 
        prefix: /static  # 匹配以 /static 开头的所有请求
    - uri:
        exact: /login  # 精确匹配 /login 的请求
    - uri: 
        exact: /logout  # 精确匹配 /logout 的请求
    - uri:
        prefix: /api/v1/products  # 匹配以 /api/v1/products 开头的请求
    route:
      - destination:
          host: productpage  # 请求会被路由到名为 productpage 的服务
          port:
            number: 9080  # 发送到该服务的 9080 端口

```
