---
layout:
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: false
---

# bookinfo

在提供的kubernetes镜像中，使用 project/istio/istio-1.17.2/services/bookinfo.yaml部署bookinfo应用，将Bookinfo应用部署到default命名空间下，使用Istio Gateway可以实现应用程序从外部访问，请为Bookinfo应用创建一个名为bookinfo-gateway的网关，指定所有HTTP流量通过80端口流入网格，然后将网关绑定到虚拟服务bookinfo上。

## 一、创建bookinfo服务

### 1、解压istio服务

<pre><code><strong>tar -zxvf istio-1.17.2-linux-amd64.tar.gz -C .  
</strong></code></pre>

### 2、进入存放bookinfo的目录

```
cd istio-1.17.2/samples/bookinfo/networking/platform/kube
```

### 3、安装bookinfo服务

```
kubectl apply -f bookinfo.yaml
```

### 4、安装bookinfo的gateway服务

```
cd /root/istio-1.17.2/samples/bookinfo/networking
```

### 5、安装gateway服务

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

```
 kubectl apply -f bookinfo-gateway.yaml 
```

## 二、服务网格

在我们部署好的Bookinfo服务中，访问Bookinfo应用发现，其中一个微服务 reviews 的三个不同版本已经部署并同时运行 ，在浏览器中访问 Bookinfo 应用程序的 /productpage 并刷新几次，您会注意到，有时书评的输出包含星级评分，有时则不包含。这是因为没有明确的默认服务版本可路由，Istio将以循环方式将请求路由到所有可用版本。

（1）将default命名空间下的pod全部删除，并添加自动注入Sidecar并进行重新调度管理。

（2）请为Bookinfo应用创建DestinationRule规则，然后创建VirtualService服务，将所有流量路由到微服务的 v1 版本。

完成这些内容，我们再次访问Bookinfo网页就会发现，流量只会访问到我们的v1版本中。

### 1、添加自动注入

```
kubectl label namespace default  istio-injection=enabled
```

### 2、创建DestinationRule规则

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  subsets:
  - name: v1
    labels:
      version: v1

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v2-mysql
    labels:
      version: v2-mysql
  - name: v2-mysql-vm
    labels:
      version: v2-mysql-vm

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### 3、创建VirtualService服务

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
spec:
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: v1

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
spec:
  hosts:
  - details
  http:
  - route:
    - destination:
        host: details
        subset: v1
```

`再次访问Bookinfo网页就会发现，流量只会访问到我们的v1版本中`
