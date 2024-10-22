---
layout:
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: false
  pagination:
    visible: false
---

# Replication Controller

## 一、Reolication Controller的概念

_**ReplicationController 类似于进程管理器，但是 ReplicationController 不是监控单个节点上的单个进程，而是监控跨多个节点的多个 Pod。**_

Replication Controller为Kubernetes的一个核心内容，应用托管到Kubernetes之后，需要保证应用能够持续的运行，Replication Controller就是这个保证的key，主要的功能如下：

* **确保pod数量**：它会确保Kubernetes中有指定数量的Pod在运行。如果少于指定数量的pod，Replication Controller会创建新的，反之则会删除掉多余的以保证Pod数量不变。
* **确保pod健康**：当pod不健康，运行出错或者无法提供服务时，Replication Controller也会杀死不健康的pod，重新创建新的。
* [**弹性伸缩**](https://cloud.tencent.com/product/as?from\_column=20065\&from=20065) ：在业务高峰或者低峰期的时候，可以通过Replication Controller动态的调整pod的数量来提高资源的利用率。同时，配置相应的监控功能（Hroizontal Pod Autoscaler），会定时自动从监控平台获取Replication Controller关联pod的整体资源使用情况，做到自动伸缩。
* **滚动升级**：滚动升级为一种平滑的升级方式，通过逐步替换的策略，保证整体系统的稳定，在初始化升级的时候就可以及时发现和解决问题，避免问题不断扩大。

## 二、Replication Controller使用案例

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
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
        ports:
        - containerPort: 80 
```

## 三、Replication Controller的扩容、缩容、更新 <a href="#id-1-kuo-rong-jiang-liang-ge-fu-ben-shu-kuo-rong-dao-san-ge-fu-ben-shu" id="id-1-kuo-rong-jiang-liang-ge-fu-ben-shu-kuo-rong-dao-san-ge-fu-ben-shu"></a>

### 1、扩容（将三个副本数扩容到五个副本数） <a href="#id-1-kuo-rong-jiang-liang-ge-fu-ben-shu-kuo-rong-dao-san-ge-fu-ben-shu" id="id-1-kuo-rong-jiang-liang-ge-fu-ben-shu-kuo-rong-dao-san-ge-fu-ben-shu"></a>

```
kubectl scale rc nginx --replicas=5
```

### 2、缩容（将五个副本数缩容到三个副本数） <a href="#id-2-suo-rong-jiang-san-ge-fu-ben-shu-suo-rong-dao-liang-ge-fu-ben-shu" id="id-2-suo-rong-jiang-san-ge-fu-ben-shu-suo-rong-dao-liang-ge-fu-ben-shu"></a>

```
kubectl scale rc nginx --replicas=3
```

### 3、更新镜像版本（将nginx更新为myapp:v1） <a href="#id-3-geng-xin-jing-xiang-ban-ben-jiang-nginx-geng-xin-wei-nginx1.16.1" id="id-3-geng-xin-jing-xiang-ban-ben-jiang-nginx-geng-xin-wei-nginx1.16.1"></a>

```
kubectl set image rc nginx nginx=myapp:v1
```

* **`nginx`** 是Replication Controller的名称
* **`nginx`**是容器的名称，必须与 Replication Controller中定义的容器名一致。
* **`myapp:v1`** 是要替换的新镜像。
