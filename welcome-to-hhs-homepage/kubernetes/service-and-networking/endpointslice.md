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

# EndpointSlice

_**创建Service服务时系统会自动创建对应的EndpointSlice服务(Pod 的地址、端口和状态信息会被记录在**** ****`EndpointSlice`**** ****中)，将请求发往Service服务，最后路由到对应的Pod上进行访问**_

```
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: example-abc  # EndpointSlice 的名称
  labels:
    kubernetes.io/service-name: example  # 关联的 Service 名称
addressType: IPv4  # 地址类型，支持 IPv4 或 IPv6
ports:
  - name: http  # 端口名称（服务通过这个端口提供 HTTP 访问）
    protocol: TCP  # 协议类型为 TCP
    port: 80  # 对应的服务端口为 80
endpoints:
  - addresses:
      - "10.1.2.3"  # Pod 或节点的 IP 地址，请求会被路由到这个地址
    conditions:
      ready: true  # 表示这个端点当前已准备好接收请求
    hostname: pod-1  # 对应的 Pod 名称
    nodeName: node-1  # 该 Pod 所在的节点名称
    zone: us-west2-a  # Pod 所在的可用区（区域信息）

```

表示服务的**一组端点（Endpoints）**，即可以访问该服务的 **Pod 或节点的 IP 地址和端口信息**

**`EndpointSlice` 与 Service 的关系**：

* 通过标签 **`kubernetes.io/service-name: example`** 将这个 `EndpointSlice` 关联到一个名为 `example` 的服务。
* 服务的请求将会转发到 `10.1.2.3` 上的 **80 端口**。
* **支持多端点**：
  * 一个 `EndpointSlice` 可能包含多个端点，这里列出了一个 Pod 的信息。
* **高效负载分发**：
  * 可以跨区域（`zone`）和节点（`nodeName`）进行负载均衡，并且仅将请求发送到状态为 **ready** 的 Pod。
