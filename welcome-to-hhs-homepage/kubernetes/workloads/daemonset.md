# DaemonSet

## 一、DaemonSet的概念

_**DaemonSet控制器能够确保k8s集群所有的节点都运行一个相同的pod副本**_，当向k8s集群中增加node节点时，这个node节点也会自动创建一个pod副本，当node节点从集群移除，这些pod也会自动删除；删除Daemonset也会删除它们创建的pod

### 1、工作原理

daemonset的控制器会监听kuberntes的daemonset对象、pod对象、node对象，这些被监听的对象之变动，就会触发syncLoop循环让kubernetes集群朝着daemonset对象描述的状态进行演进。

### 2、应用场景

在集群的每个节点上运行存储，比如：glusterd 或 ceph。 在每个节点上运行日志收集组件，比如：flunentd 、 logstash、filebeat等。 在每个节点上运行监控组件，比如：Prometheus、 Node Exporter 、collectd等。

### 3、DaemonSet 与 Deployment 的区别

#### Deployment 部署的副本 Pod 会分布在各个 Node 上，每个 Node 都可能运行好几个副本。 DaemonSet 的不同之处在于：每个 Node 上最多只能运行一个副本。

## 二、DaemonSet资源清单

### spec字段

&#x20;  minReadySeconds   #当新的pod启动几秒种后，再kill掉旧的pod。

&#x20;  revisionHistoryLimit  #历史版本

&#x20;  selector  #用于匹配pod的标签选择器（必选参数）

&#x20;  template  #定义Pod的模板（必选参数）

&#x20;  updateStrategy  #daemonset的升级策略

## 三、DaemonSet使用案例

### 部署日志收集组件fluentd

```
apiVersion: apps/v1  #DaemonSet使用的api版本
kind: DaemonSet     # 资源类型
metadata:
  name: fluentd-elasticsearch   #资源的名字
  namespace: kube-system      #资源所在的名称空间
  labels:
    k8s-app: fluentd-logging    #资源具有的标签
spec:
  selector:         #标签选择器
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:  #基于这回模板定义的pod具有的标签
        name: fluentd-elasticsearch
    spec:
      tolerations:   #定义容忍度
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:    #定义容器
      - name: fluentd-elasticsearch
        image: xianchao/fluentd:v2.5.1
        resources:  #资源配额
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts: 
        - name: varlog
          mountPath: /var/log   #把本地/var/log目录挂载到容器
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers  
#把/var/lib/docker/containers/挂载到容器里
          readOnly: true  #挂载目录是只读权限
      terminationGracePeriodSeconds: 30  #优雅的关闭服务
      volumes:
      - name: varlog
        hostPath:
          path: /var/log  #基于本地目录创建一个卷
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers  #基于本地目录创建一个卷
```

## 四、Daemonset滚动更新

rollingUpdate更新策略只支持maxUnavailabe，先删除在更新；因为我们不支持一个节点运行两个pod，因此需要先删除一个，在更新一个。

#### 更新镜像版本（将xianchao/fluentd:v2.5.1更新为ikubernetes/filebeat:5.6.6-alpine） <a href="#id-3-geng-xin-jing-xiang-ban-ben-jiang-nginx-geng-xin-wei-nginx1.16.1" id="id-3-geng-xin-jing-xiang-ban-ben-jiang-nginx-geng-xin-wei-nginx1.16.1"></a>

```
kubectl set image daemonsets fluentd-elasticsearch fluentd-elasticsearch=ikubernetes/filebeat:5.6.6-alpine
```

