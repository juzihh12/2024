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

# ConfigMap

## 一、ConfigMap的概述

### 1、什么是ConfigMap？

Configmap是k8s中的资源对象，用于保存非机密性的配置的，数据可以用key/value键值对的形式保存，也可通过文件的形式保存

### 2、ConfigMap解决的问题

我们在部署服务的时候，每个服务都有自己的配置文件，如果一台服务器上部署多个服务：nginx、tomcat、apache等，那么这些配置都存在这个节点上，假如一台服务器不能满足线上高并发的要求，需要对服务器扩容，扩容之后的服务器还是需要部署多个服务：nginx、tomcat、apache，新增加的服务器上还是要管理这些服务的配置，如果有一个服务出现问题，需要修改配置文件，每台物理节点上的配置都需要修改，这种方式肯定满足不了线上大批量的配置变更要求。 所以，k8s中引入了Configmap资源对象，可以当成volume挂载到pod中，实现统一的配置管理。

### 3、Configmap应用场景

1、使用k8s部署应用，当你将应用配置写进代码中，更新配置时也需要打包镜像，configmap可以将配置信息和docker镜像解耦，以便实现镜像的可移植性和可复用性，因为一个configMap其实就是一系列配置信息的集合，可直接注入到Pod中给容器使用。configmap注入方式有两种，一种将configMap做为存储卷，一种是将configMap通过env中configMapKeyRef注入到容器中。

2、使用微服务架构的话，存在多个服务共用配置的情况，如果每个服务中单独一份配置的话，那么更新配置就很麻烦，使用configmap可以友好的进行配置共享。

### 4、局限性

ConfigMap在设计上不是用来保存大量数据的。在ConfigMap中保存的数据不可超过1 MiB。如果你需要保存超出此尺寸限制的数据，可以考虑挂载存储卷或者使用独立的数据库或者文件服务。

## 二、ConfigMap的创建方法

### 1、命令行直接创建

```
kubectl create configmap tomcat-config --from-literal=tomcat_port=8080 --from-literal=server_name=myapp.tomcat.com
```

### 2、通过文件进行创建

```
1、创建一个nginx.conf文件
vim nginx.conf
server {
  server_name www.nginx.com;
  listen 80;
  root /home/nginx/www
}
2、定义一个key为www，值为nginx.conf的内容
kubectl create configmap www-nginx --from-file=www=./nginx.conf
```

### 3、指定目录进行创建

```
mkdir test-configmap && cd test-configmap     # 创建一个目录并且进入
vim my-server.cnf
 server-id=1
vim my-slave.cnf
  slave-id=2
kubectl create configmap test-configmap --from-file=/root/test-configmap
```

## 三、ConfigMap的资源清单

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-configmap
  labels:
    app: mysql
data:
  master.cnf: |
    [mysqld]
    log-bin
    log_bin_trust_function_creators=1
    lower_case_table_names=1
  slave.cnf: |
    [mysqld]
    super-read-only
    log_bin_trust_function_creators=1
```

## 四、实验案例（使用ConfigMap）

### 1、通过环境变量引入：使用configMapKeyRef

```
1、定义configmap
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-configmap
  labels:
    app: mysql
data:
  log: "1"
  lower: "1"
2、创建一个pod进行引用创建的configmap
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
spec:
  containers:
  - name: mysql
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh","-c","sleep 3600"]
    env:
    - name: log_bin
      valueFrom:
        configMapKeyRef:
          name: mysql-configmap
          key: log
    - name: lower
      valueFrom:
        configMapKeyRef:
          name: mysql-configmap     # 定义的configmap名称
          key: lower
3、查看是否正常导入环境变量
kubectl exec -it mysql-pod -- /bin/sh
/ # printenv 
    log_bin=1
    lower=1
```

### 2、通过环境变量引入：使用envfrom

```
1、定义configmap
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-configmap
  labels:
    app: mysql
data:
  log: "1"
  lower: "1"
2、创建pod进行引用configmap
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-pod
spec:
  containers:
  - name: mysql
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh","-c","sleep 3600"]
    envFrom:
    - configMapRef:
        name: mysql-configmap
  restartPolicy: Never
3、查看是否正常导入环境变量
kubectl exec -it envfrom-pod -- /bin/sh
/ # printenv 
    log_bin=1
    lower=1
```

### 3、把configmap做成volume，挂载到pod

```
1、定义configmap
apiVersion: v1
kind: ConfigMap
metadata:
  name: volume-configmap
  labels:
    app: volume
data:
  log: "1"
  lower: "1"
  master.cnf: |
    [mysqld]
    welcome=111
2、创建一个pod进行引用configmap
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: volume
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh","-c","sleep 3600"]
    volumeMounts:
    - name: volume-configmap
      mountPath: /tmp/config
  volumes:
  - name: volume-configmap
    configMap:
      name: volume-configmap
  restartPolicy: Never
3、验证是否正常运行
kubectl exec -it volume-pod -- /bin/sh
   ls /tmp/config/
log         lower       master.cnf
```

### 4、configmap的热更新

```
1、更改configmap，使其进行更新
kubectl edit configmaps volume-configmap
将log: "1"改为log: "2"
2、再次进入容器并查看log文件
kubectl exec -it volume-pod -- /bin/sh
  cat /tmp/config/log
  2                 # 这个过程大概会进行10s左右，才会更新过来
```

`注意：`

`更新 ConfigMap 后：`

`使用该 ConfigMap 挂载的 Env 不会同步更新`
