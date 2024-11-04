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

# Persistent Storage

## 一、emptyDir

### 1、资源概述

emptyDir类型的Volume是在Pod分配到Node上时被创建，Kubernetes会在Node上自动分配一个目录，因此无需指定宿主机Node上对应的目录文件。 这个目录的初始内容为空，当Pod从Node上移除时，emptyDir中的数据会被永久删除。emptyDir Volume主要用于某些应用程序无需永久保存的临时目录，多个容器的共享目录等。

### 2、资源示例

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-empty
spec:
  containers:
  - name: container-empty
    image: nginx
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - emptyDir:
     {}
    name: cache-volume
```

查看资源存储在什么位置

```
kubectl get pods -o wide | grep emptydir     # 查看资源分布在哪一个节点
kubectl get pods pod-emptydir -o yaml | grep uid      # 获取资源的uid
tree /var/lib/kubelet/pods/2f386c85-6960-4d75-8e4a-8bccfd83c9b5    # 查看资源存储在什么位置
由上可知临时数据存储在/var/lib/kubelet/pods/2f386c85-6960-4d75-8e4a-8bccfd83c9b5/volumes/kubernetes.io~empty-dir/cache-volume
```

## 二、hostPath

### 1、资源概述

hostPath Volume是指Pod挂载宿主机上的目录或文件。 hostPath Volume使得容器可以使用宿主机的文件系统进行存储，hostpath（宿主机路径）：节点级别的存储卷，在pod被删除，这个存储卷还是存在的，不会被删除，所以只要同一个pod被调度到同一个节点上来，在pod被删除重新被调度到这个节点之后，对应的数据依然是存在的。

### 2、资源示例

```
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpath
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: test-nginx
    volumeMounts:
    - mountPath: /test-nginx
      name: test-volume
  - image: tomcat:8.5-jre8-alpine
    imagePullPolicy: IfNotPresent
    name: test-tomcat
    volumeMounts:
    - mountPath: /test-tomcat
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /data1
      type: DirectoryOrCreate   # 在node节点创建/data1用来永久化存储
```

type的资源

<figure><img src="../../../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

hostpath存储卷缺点：

单节点，pod删除之后重新创建必须调度到同一个node节点，数据才不会丢失

可以用分布式存储：nfs，cephfs，glusterfs

## 三、NFS

### 1、资源概述

网络文件系统，英文Network File System(NFS)，是由[SUN](https://baike.baidu.com/item/SUN/69463?fromModule=lemma\_inlink)公司研制的[UNIX](https://baike.baidu.com/item/UNIX/219943?fromModule=lemma\_inlink)[表示层](https://baike.baidu.com/item/%E8%A1%A8%E7%A4%BA%E5%B1%82/4329716?fromModule=lemma\_inlink)协议(presentation layer protocol)，能使使用者访问网络上别处的文件就像在使用自己的计算机一样

### 2、资源示例

以k8s的控制节点作为NFS服务端

#### 1、安装nfs服务及后端存储目录

<pre><code>yum install nfs-utils -y
mkdir /data/volumes -pv 
systemctl start nfs-server.service
vim /etc/exports 
   /data/volumes *(rw,no_root_squash)
#rw 该主机对该共享目录有读写权限
# no_root_squash 登入 NFS 主机使用分享目录的使用者，如果是 root 的话，那么对于这个分享的目录来说，他就具有 root 的权限
exportfs -arv  # 使配置生效
<strong>systemctl restart nfs-server.service &#x26;&#x26; systemctl enable nfs-server.service  # 重启服务并设置为开机自启动
</strong>
yum install nfs-utils -y  # 在node节点安装nfs服务
systemctl restart nfs-server.service &#x26;&#x26; systemctl enable nfs-server.service  # 重启服务并设置为开机自启动
mkdir /test &#x26;&#x26; mount 192.168.100.10:/data/volumes /test/ 创建挂载目录，并挂载
umount /test # 查看是否正常挂载，正常挂载后取消挂载
</code></pre>

> exportfs 命令常用选项\
> \-a 全部挂载或者全部卸载\
> \-r 重新挂载\
> \-u 卸载某一个目录\
> \-v 显示共享目录 以下操作在服务端上

#### 2、编写nfs服务

```
apiVersion: v1
kind: Pod
metadata:
  name: test-nfs-volume
spec:
  containers:
  - name: test-nfs
    image: nginx
    imagePullPolicy: IfNotPresent
    ports:
    - name: http
      containerPort: 80
      protocol: TCP
    volumeMounts:
    - name: nfs-volumes
      mountPath: /usr/share/nginx/html
  volumes:
  - name: nfs-volumes
    nfs:
      path: /data/volumes
      server: 192.168.100.10
```

#### 3、验证服务是否搭建成功

```
echo "11111" > /data/volumes/index.html   
kubectl exec -it test-nfs-volume  -- /bin/bash 
   cat /usr/share/nginx/html/index.html
   11111
```

上面说明挂载nfs存储卷成功了，nfs支持多个客户端挂载，可以创建多个pod，挂载同一个nfs服务器共享出来的目录；但是nfs如果宕机了，数据也就丢失了，所以需要使用分布式存储，常见的分布式存储有glusterfs和cephfs

## 四、PVC

### 1、资源概述

#### （1）pv的供应方式

可以通过两种方式配置PV：静态或动态。

静态的：

集群管理员创建了许多PV。它们包含可供群集用户使用的实际存储的详细信息。它们存在于Kubernetes API中，可供使用。

动态的：

当管理员创建的静态PV都不匹配用户的PersistentVolumeClaim时，群集可能会尝试为PVC专门动态配置卷。此配置基于StorageClasses，PVC必须请求存储类，管理员必须创建并配置该类，以便进行动态配置。

#### （2）绑定

用户创建pvc并指定需要的资源和访问模式。在找到可用pv之前，pvc会保持未绑定状态

#### （3）使用

a）需要找一个存储服务器，把它划分成多个存储空间；

b）k8s管理员可以把这些存储空间定义成多个pv；

c）在pod中使用pvc类型的存储卷之前需要先创建pvc，通过定义需要使用的pv的大小和对应的访问模式，找到合适的pv；

d）pvc被创建之后，就可以当成存储卷来使用了，我们在定义pod时就可以使用这个pvc的存储卷

e）pvc和pv它们是一一对应的关系，pv如果被pvc绑定了，就不能被其他pvc使用了；

f）我们在创建pvc的时候，应该确保和底下的pv能绑定，如果没有合适的pv，那么pvc就会处于pending状态。

#### （4）回收策略

当我们创建pod时如果使用pvc做为存储卷，那么它会和pv绑定，当删除pod，pvc和pv绑定就会解除，解除之后和pvc绑定的pv卷里的数据需要怎么处理，目前，卷可以保留，回收或删除：

1、Retain

当删除pvc的时候，pv仍然存在，处于released状态，但是它不能被其他pvc绑定使用，里面的数据还是存在的，当我们下次再使用的时候，数据还是存在的，这个是默认的回收策略

2、Delete

删除pvc时即会从Kubernetes中移除PV，也会从相关的外部设施中删除存储资产

3、Recycle （不推荐使用，1.15可能被废弃了）

### 2、资源示例

#### 1、创建nfs共享目录

```
mkdir /data/volume_test/v{1,2,3,4,5,6,7,8,9,10} -p 
vim /etc/exports
exporting 192.168.100.10/24:/data/volume_test/v10
exporting 192.168.100.10/24:/data/volume_test/v9
exporting 192.168.100.10/24:/data/volume_test/v8
exporting 192.168.100.10/24:/data/volume_test/v7
exporting 192.168.100.10/24:/data/volume_test/v6
exporting 192.168.100.10/24:/data/volume_test/v5
exporting 192.168.100.10/24:/data/volume_test/v4
exporting 192.168.100.10/24:/data/volume_test/v3
exporting 192.168.100.10/24:/data/volume_test/v2
exporting 192.168.100.10/24:/data/volume_test/v1
exportfs -arv   # 使其生效
```

#### 2、编写配置

#### 1、创建pv

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: v1
  labels:
    app: v1
spec:
  nfs:
    path: /data/volume_test/v1
    server: 192.168.100.10
  accessModes: ["ReadWriteOnce"]
  capacity: 
    storage: 1Gi
---
apiVersion: v1
kind: PersistentVolume
metadata: 
  name: v2
  labels:
    app: v2
spec:
  nfs:
    path: /data/volume_test/v2
    server: 192.168.100.10
  accessModes: ["ReadOnlyMany"]
  capacity:
    storage: 2Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: v3
  labels:
    app: v3
spec:
  nfs:
    path: /data/volume_test/v3
    server: 192.168.100.10
  accessModes: ["ReadWriteMany"]
  capacity:
    storage: 3Gi
```

#### 2、创建pvc使其对接到pv

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: v1
spec:
  accessModes: ["ReadWriteOnce"]
  selector:
    matchLabels:
      app: v1
  resources:
    requests: 
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: v2
spec:
  accessModes: ["ReadOnlyMany"]
  selector:
    matchLabels:
      app: v2
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: v3
spec:
  accessModes: ["ReadWriteMany"]
  selector:
    matchLabels:
      app: v3
  resources:
    requests:
      storage: 3Gi
```

`accessModes：`

`RWO：readwariteonce： 单路读写，允许同一个node节点上的pod访问`

`ROX： readonlymany:    多路只读，允许不通node节点的pod以只读方式访问`

`RWX： readwritemany：  多路读写，允许不同的node节点的pod以读写方式访问`&#x20;

#### 3、创建deployment服务

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-pvc
spec: 
  replicas: 3
  selector: 
    matchLabels:
      app: pvc
  template: 
    metadata:
      labels:
        app: pvc
    spec:
      containers:
      - name: pod-pvc
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          protocol: TCP
          containerPort: 80
        volumeMounts:
        - name: nginx-pvc
          mountPath: /usr/share/nginx/html
      volumes:
      - name: nginx-pvc
        persistentVolumeClaim:
          claimName: v1
```

#### 4、验证是否配置成功

```
cd /data/volume_test/v1 && touch 11  # 切换到nfs存储目录，并创建11文件进行验证
kubectl exec -it deployment-pvc-6dd4fc75dd-79h2g -- /bin/bash   # 进入容器并查看目录下是否有11文件
ls /usr/share/nginx/html/
  11
```

### 3、使用pvc和pv的注意事项

1、我们每次创建pvc的时候，需要事先有划分好的pv，这样可能不方便，那么可以在创建pvc的时候直接动态创建一个pv这个存储类，pv事先是不存在的

2、pvc和pv绑定，如果使用默认的回收策略retain，那么删除pvc之后，pv会处于released状态，我们想要继续使用这个pv，需要手动删除pv，kubectl delete pv pv\_name，删除pv，不会删除pv里的数据，当我们重新创建pvc时还会和这个最匹配的pv绑定，数据还是原来数据，不会丢失。

经过测试，如果回收策略是Delete，删除pv，pv后端存储的数据也不会被删除

3、示例

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: v4
  labels: 
    app: v4
spec:
  accessModes: 
  - ReadWriteOnce
  capacity: 
    storage: 1Gi
  nfs:
    path: /data/volume_test/v4
    server: 192.168.100.10
  persistentVolumeReclaimPolicy: Delete
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: v4
spec:
  accessModes: 
  - ReadWriteOnce
  resources:
    requests: 
      storage: 1Gi
  selector:
    matchLabels:
      app: v4
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-v4
spec:
  replicas: 3
  selector: 
    matchLabels: 
      app: v4
  template:
    metadata:
      labels:
        app: v4
    spec:
      containers:
      - name: pod-v4
        image: nginx
        imagePullPolicy: IfNotPresent
        ports: 
        - containerPort: 80
          name: http
          protocol: TCP
        volumeMounts:
        - name: pvc-v4
          mountPath: /usr/share/nginx/html
      volumes:
      - persistentVolumeClaim:
          claimName: v4
        name: pvc-v4 
```

验证服务是否正常运行

```
touch /data/volume_test/v4/11   # 在nfs存储目录创建11文件进行
kubectl exec -it deployment-v4-867ddf9bc5-dsvmz -- /bin/bash    # 进入容器并查看目录下是否有11文件
ls /usr/share/nginx/html/
11
```
