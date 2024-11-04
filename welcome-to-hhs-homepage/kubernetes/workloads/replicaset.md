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

# ReplicaSet

## 一、Replicaset的概念

_**ReplicaSet是kubernetes中的一种副本控制器**_，简称rs，主要作用是控制由其管理的pod，使pod副本的数量始终维持在预设的个数。它的主要作用就是保证一定数量的Pod能够在集群中正常运行，它会持续监听这些Pod的运行状态，在Pod发生故障时重启pod，pod数量减少时重新运行新的 Pod副本。官方推荐不要直接使用ReplicaSet，用Deployments取而代之，Deployments是比ReplicaSet更高级的概念，它会管理ReplicaSet并提供很多其它有用的特性，最重要的是Deployments支持声明式更新，声明式更新的好处是不会丢失历史变更。所以Deployment控制器不直接管理Pod对象，而是由 Deployment 管理ReplicaSet，再由ReplicaSet负责管理Pod对象。

1、工作原理： 确保pod副本一直处于满足用户期望的数量，具有自动扩容、缩容的机制

2、组成部分：期望副本数、标签选择器、pod资源模板

## 二、Replicaset使用案例

### 部署guestbook示例

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  namespace: guestbook
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: docker.io/yecc/gcr.io-google_samples-gb-frontend:v3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        startupProbe:
          failureThreshold: 3
          initialDelaySeconds: 20
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 10
          httpGet: 
            scheme: HTTP
            port: 80
            path: /
        livenessProbe:
          failureThreshold: 3
          initialDelaySeconds: 20
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 10
          httpGet: 
            scheme: HTTP
            port: 80
            path: /
        readinessProbe:
          failureThreshold: 3
          initialDelaySeconds: 20
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 10
          httpGet: 
            scheme: HTTP
            port: 80
            path: /
```

## 三、Replicaset的扩容、缩容和回滚

### 1、扩容（将三个副本数扩容到五个副本数） <a href="#id-1-kuo-rong-jiang-liang-ge-fu-ben-shu-kuo-rong-dao-san-ge-fu-ben-shu" id="id-1-kuo-rong-jiang-liang-ge-fu-ben-shu-kuo-rong-dao-san-ge-fu-ben-shu"></a>

```
kubectl scale replicaset frontend --replicas=5
```

### 2、缩容（将五个副本数缩容到三个副本数） <a href="#id-2-suo-rong-jiang-san-ge-fu-ben-shu-suo-rong-dao-liang-ge-fu-ben-shu" id="id-2-suo-rong-jiang-san-ge-fu-ben-shu-suo-rong-dao-liang-ge-fu-ben-shu"></a>

```
kubectl scale replicaset frontend --replicas=3
```

### 3、更新镜像版本（将gcr.io-google\_samples-gb-frontend:v3更新为myapp:v2） <a href="#id-3-geng-xin-jing-xiang-ban-ben-jiang-nginx-geng-xin-wei-nginx1.16.1" id="id-3-geng-xin-jing-xiang-ban-ben-jiang-nginx-geng-xin-wei-nginx1.16.1"></a>

```
kubectl set image replicaset frontend php-redis=myapp:v2
```

* **`frontend`** 是 ReplicaSet 的名称。
* **`php-redis`**是容器的名称，必须与 ReplicaSet 中定义的容器名一致。
* **`myapp:v2`** 是要替换的新镜像。

## 四、总结

生产环境如果升级，可以删除一个pod，观察一段时间之后没问题再删除另一个pod，但是这样需要人工干预多次；实际生产环境一般采用蓝绿发布，原来有一个rs1，再创建一个rs2（控制器），通过修改service标签，修改service可以匹配到rs2的控制器，这样才是蓝绿发布，这个也需要我们精心的部署规划，我们有一个控制器就是建立在rs之上完成的，叫做Deployment
