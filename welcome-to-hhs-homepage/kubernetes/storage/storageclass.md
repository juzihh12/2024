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

# StorageClass

## 一、为什么使用storageclass

PV和PVC模式都是需要先创建好PV，然后定义好PVC和pv进行一对一的Bond，但是如果PVC请求成千上万，那么就需要创建成千上万的PV，对于运维人员来说维护成本很高，Kubernetes提供一种自动创建PV的机制，叫StorageClass，它的作用就是创建PV的模板。k8s集群管理员通过创建storageclass可以动态生成一个存储卷pv供k8s pvc使用。

StorageClass会定义以下两部分：

1、PV的属性 ，比如存储的大小、类型等；

2、创建这种PV需要使用到的存储插件，比如Ceph、NFS等

## 二、storageclass资源清单

&#x20;  allowVolumeExpansion      # 是否允许 PersistentVolumeClaim (PVC) 动态扩容存储卷，此功能仅用于扩容卷，不能用于缩小卷

&#x20;  allowedTopologies         # 指定存储卷必须位于的拓扑范围

&#x20;  mountOptions       # 挂载存储卷时使用的选项

&#x20;  parameters       # 存储类的配置参数，由具体的 存储提供商（provisioner） 定义

&#x20;  provisioner        # 指定存储类使用的 存储提供商（例如 AWS EBS、GCE PD、Ceph 等）_**必选参数**_

&#x20;  reclaimPolicy               # 存储卷被释放后的回收策略  &#x20;

&#x20; volumeBindingMode        # 存储卷的绑定行为 。Immediate：立即绑定存储卷，WaitForFirstConsumer：延迟绑定，直到有 Pod 需要此卷

## 三、storageclass使用案例

### 1、创建运行nfs-provisioner需要的sa账号

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-provisioner
```

`serviceaccount是为了方便Pod里面的进程调用Kubernetes API或其他外部服务而设计的。`

`指定了serviceaccount之后，我们把pod创建出来了，我们在使用这个pod时，这个pod就有了我们指定的账户的权限了`

### 2、对sa授权

```
kubectl create clusterrolebinding nfs-provisioner-clusterrolebinding --clusterrole=cluster-admin --serviceaccount=default:nfs-provisioner
```

创建了一个 **ClusterRoleBinding**，将 **cluster-admin** 超级管理员角色绑定到某个 ServiceAccount 上。

扩展：

此操作风险很大，建议创建一个用来管理pv和pvc的角色，然后将其绑定到sa上

#### 1、创建ClusterRole角色用来管理pv和pvc

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-pv-manager
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes", "persistentvolumeclaims"]
    verbs: ["get", "list", "create", "delete", "watch"]
```

#### 2、创建 ClusterRoleBinding，将角色绑定到sa上

```
kubectl create clusterrolebinding nfs-pv-manager-binding \
--clusterrole=nfs-pv-manager \
--serviceaccount=default:nfs-provisioner
```

### 3、安装nfs-provisioner程序

<pre><code>mkdir /data/nfs_pro -p  # 创建共享目录
vim /etc/exports
<strong>  /data/nfs_pro 192.168.100.10/24(rw,no_root_squash)
</strong>exportfs -arv      # 配置生效
</code></pre>

### 4、创建deployment服务

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-provisioner
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector: 
    matchLabels:
      app: nfs-provisioner
  template:
    metadata:
      labels:
        app: nfs-provisioner
    spec:
      containers:
      - name: nfs-provisioner
        image: registry.cn-beijing.aliyuncs.com/mydlq/nfs-subdir-external-provisioner:v4.0.0
        imagePullPolicy: IfNotPresent
        env:
          - name: PROVISIONER_NAME
            value: example.com/nfs
          - name: NFS_SERVER
            value: 192.168.100.10
          - name: NFS_PATH
            value: /data/nfs_pro
        volumeMounts:
        - name: nfs-client-root
          mountPath: /persistentvolumes
      volumes:
      - name: nfs-client-root
        nfs:
          path: /data/nfs_pro
          server: 192.168.100.10
```

### 5、创建storageclass，动态供给pv

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs
provisioner: example.com/nfs
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-provisioner
```

`provisioner处写的example.com/nfs应该跟安装nfs provisioner时候的env下的PROVISIONER_NAME的value值保持一致，如下：`

`env:`

&#x20;   `- name: PROVISIONER_NAME`

&#x20;    `value: example.com/nfs`

### 6、创建pvc，通过storageclass动态生成pv

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests: 
      storage: 1Gi
  storageClassName: nfs
```

### 7、查看是否动态生成了pv，pvc是否创建成功，并和pv绑定

```
❯ kubectl get pvc
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-pvc   Bound    pvc-9daf7917-7020-4e30-ad48-5e64f0e5607d   1Gi        RWO            nfs            12m
```

### 8、步骤总结：

#### 1、供应商：创建一个nfs provisioner

#### 2、创建storageclass，storageclass指定刚才创建的供应商

#### 3、创建pvc，这个pvc指定storageclass

### 9、创建pod，挂载storageclass动态生成的pvc：test-pvc

```
kind: Pod
apiVersion: v1
metadata:
  name: read-pod
spec:
  containers:
  - name: read-pod
    image: nginx
    imagePullPolicy: IfNotPresent
    volumeMounts:
      - name: nfs-pvc
        mountPath: /usr/share/nginx/html
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-pvc
```

