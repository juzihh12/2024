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

# StatefulSet

## 一、StatefulSet的概念

_**StatefulSet是为了管理有状态服务的问题而设计的**_

有状态服务？

StatefulSet是有状态的集合，管理有状态的服务，它所管理的_**Pod的名称不能随意变化**_。数据持久化的目录也是不一样，每一个Pod都有自己独有的数据持久化存储目录。比如MySQL主从、redis集群等。

无状态服务？

RC、Deployment、DaemonSet都是管理无状态的服务，它们所管理的Pod的IP、名字，启停顺序等都是随机的。个体对整体无影响，所有pod都是共用一个数据卷的，部署的tomcat就是无状态的服务，tomcat被删除，在启动一个新的tomcat，加入到集群即可，跟tomcat的名字无关。

## 二、Statefulet资源清单

### spec字段

&#x20;  podManagementPolicy   #pod管理策略

&#x20;  replicas   #副本数

&#x20;  revisionHistoryLimit   #保留的历史版本

&#x20;  selector   #标签选择器，选择它所关联的pod（必选参数）

&#x20;  serviceName   #headless service的名字（必选参数）

&#x20;  template     #生成pod的模板（必选参数）

&#x20;  updateStrategy  #更新策略

&#x20;  volumeClaimTemplates  #存储卷申请模板

## 三、StatefulSet使用案例

### 部署web站点

#### 1、statefulset和service

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: statefulset
spec:
  replicas: 3
  selector: 
    matchLabels:
      app: nginx
  serviceName: "nginx"   #headless service的名字
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
        - name: web
          containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/local/nginx/www
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "nfs"
      resources:
        requests: 
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports: 
  - port: 80
    name: nginx
  clusterIP: None
  selector: 
    app: nginx
```

#### 2、pv和pvc

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: www-web-0
spec:
  accessModes:
    - ReadWriteOnce
    - ReadWriteMany
  storageClassName: nfs-web
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: www-web-1
spec:
  accessModes:
    - ReadWriteOnce
    - ReadWriteMany
  storageClassName: nfs-web
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-www1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
    - ReadWriteMany
  storageClassName: nfs-web
  nfs:
    path: /mnt/nfs/www1   # NFS 共享路径
    server: 192.168.100.10  # 替换为 NFS 服务器的 IP 地址
  persistentVolumeReclaimPolicy: Delete
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-www2
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
    - ReadWriteMany
  storageClassName: nfs-web
  nfs:
    path: /mnt/nfs/www2   # NFS 共享路径
    server: 192.168.100.10  # 替换为 NFS 服务器的 IP 地址
  persistentVolumeReclaimPolicy: Delete
```

Headless service?

Headless service不分配clusterIP，headless service可以通过解析service的DNS,返回所有Pod的dns和ip地址 (statefulSet部署的Pod才有DNS)，普通的service,只能通过解析service的DNS返回service的ClusterIP。headless service会为service分配一个域名

`K8s中资源的全局FQDN格式:` 　　

`Service_NAME.NameSpace_NAME.Domain.LTD.` 　　

`Domain.LTD.=svc.cluster.local.　　　　 #这是默认k8s集群的域名。`

为什么要用volumeClaimTemplate？

对于有状态应用都会用到持久化存储，比如mysql主从，由于主从数据库的数据是不能存放在一个目录下的，每个mysql节点都需要有自己独立的存储空间。而在deployment中创建的存储卷是一个共享的存储卷，多个pod使用同一个存储卷，它们数据是同步的，而statefulset定义中的每一个pod都不能使用同一个存储卷，这就需要使用volumeClainTemplate，当在使用statefulset创建pod时，volumeClainTemplate会自动生成一个PVC，从而请求绑定一个PV，每一个pod都有自己专用的存储卷。Pod、PVC和PV对应的关系图如下：

<figure><img src="../../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

使用kubectl run运行一个提供nslookup命令的容器的，这个命令来自于dnsutils包，通过对pod主机名执行nslookup，可以检查它们在集群内部的DNS地址：

```
 kubectl run busybox --image docker.io/library/busybox:1.28  --image-pull-policy=IfNotPresent --restart=Never --rm -it busybox -- sh
```

### 1、nslookup的使用

```
root@web-1:/# nslookup web-0.nginx.default.svc.cluster.local  
Server:		10.96.0.10
Address:	10.96.0.10#53
Name:	web-0.nginx.default.svc.cluster.local 

#statefulset创建的pod也是有dns记录的
Address: 10.244.209.154  #解析的是pod的ip地址

root@web-1:/# nslookup nginx.default.svc.cluster.local
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	nginx.default.svc.cluster.local  #查询service dns，会把对应的pod ip解析出来
Address: 10.244.209.139

Name:	nginx.default.svc.cluster.local
Address: 10.244.209.140

root@web-1:/# dig -t A nginx.default.svc.cluster.local @10.96.0.10
; <<>> DiG 9.11.5-P4-5.1+deb10u3-Debian <<>> -t A nginx.default.svc.cluster.local @10.96.0.10
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 27207
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: f2f17492cf38ad44 (echoed)
;; QUESTION SECTION:
;nginx.default.svc.cluster.local. IN	A

;; ANSWER SECTION:
nginx.default.svc.cluster.local. 30 IN	A	10.244.209.139
nginx.default.svc.cluster.local. 30 IN	A	10.244.209.140

;; Query time: 0 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Fri Apr 09 15:10:40 UTC 2021
;; MSG SIZE  rcvd: 166
```

### 2、dig的使用

dig -t A nginx.default.svc.cluster.local @10.96.0.10

格式如下：&#x20;

@来指定域名服务器

A 为解析类型 ，A记录是解析域名到IP

\-t 指定要解析的类型

## 四、service和headless service的区别

```
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
---
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
        image: busybox
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        command:
          - sleep
          - "3600"
```

可以发现创建的pod是随机的

```
 kubectl exec -it web-1 -- /bin/bash
 nslookup my-nginx.default.svc.cluster.local  # 解析的是service的ip地址
```

## 五、Statefulset总结：

1、Statefulset管理的pod，pod名字是有序的，由statefulset的名字-0、1、2这种格式组成

2、创建statefulset资源的时候，必须事先创建好一个service,如果创建的service没有ip，那对这个service做dns解析，会找到它所关联的pod ip，如果创建的service有ip，那对这个service做dns解析，会解析到service本身ip。

3、statefulset管理的pod，删除pod，新创建的pod名字跟删除的pod名字是一样的

4、statefulset具有volumeclaimtemplate这个字段，这个是卷申请模板，会自动创建pv，pvc也会自动生成，跟pv进行绑定，那如果创建的statefulset使用了volumeclaimtemplate这个字段，那创建pod，数据目录是独享的

5、ststefulset创建的pod，是域名的（域名组成：pod-name.svc-name.svc-namespace.svc.cluster.local）

## 六、Statefulset的扩容、缩容、更新

### 1、扩容（将三个副本数扩容到五个副本数） <a href="#id-1-kuo-rong-jiang-liang-ge-fu-ben-shu-kuo-rong-dao-san-ge-fu-ben-shu" id="id-1-kuo-rong-jiang-liang-ge-fu-ben-shu-kuo-rong-dao-san-ge-fu-ben-shu"></a>

```
kubectl scale statefulset nginx --replicas=5
```

### 2、缩容（将五个副本数缩容到三个副本数） <a href="#id-2-suo-rong-jiang-san-ge-fu-ben-shu-suo-rong-dao-liang-ge-fu-ben-shu" id="id-2-suo-rong-jiang-san-ge-fu-ben-shu-suo-rong-dao-liang-ge-fu-ben-shu"></a>

```
kubectl scale statefulset nginx --replicas=3
```

### 3、更新镜像版本（将nginx更新为myapp:v2） <a href="#id-3-geng-xin-jing-xiang-ban-ben-jiang-nginx-geng-xin-wei-nginx1.16.1" id="id-3-geng-xin-jing-xiang-ban-ben-jiang-nginx-geng-xin-wei-nginx1.16.1"></a>

```
kubectl set image statefulset nginx nginx=myapp:v2
```

* **`nginx`** 是ststefulset的名称。
* **`nginx`**是容器的名称，必须与 statefulset 中定义的容器名一致。
* **`myapp:v2`** 是要替换的新镜像。

如果更新策略是OnDelete，那不会自动更新pod，需要手动删除，重新常见的pod才会实现更新
