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

# pvc一直处于peding状态



```
pod/es-cluster-0                       0/1     Pending             0          5s
statefulset.apps/es-cluster   0/3     5s
```

```
[root@master contest]# kubectl get pvc
NAME                              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-data-es-cluster-0   Pending                                      storage        32m
```

## 1、逐步排查错误原因

### 1、查看statefulset的日志

```
  kubectl describe statefulset.apps/es-cluster  # 查看statefulset可以看出没有什么大问题
  
  Type    Reason            Age                  From                    Message
  ----    ------            ----                 ----                    -------
  Normal  SuccessfulCreate  2m27s                statefulset-controller  create Pod es-cluster-0 in StatefulSet es-cluster successful
  Normal  SuccessfulCreate  111s (x2 over 2m1s)  statefulset-controller  create Claim elasticsearch-data-es-cluster-0 Pod es-cluster-0 in StatefulSet es-cluster success
```

### 2、查看pod的日志

```
kubectl describe pod/es-cluster-0   # 显示pod具有未绑定的pvc

Warning  FailedScheduling  2m2s (x3 over 2m46s)   default-scheduler  0/3 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod..
```

### 3、查看pvc的日志

```
kubectl describe pvc elasticsearch-data-es-cluster-0  # 显示一直等待创建卷也就是pv，但是我们已经做了pvc绑定，所以就不需要手动去创建
  
Normal  ExternalProvisioning  105s (x26 over 7m56s)  persistentvolume-controller  waiting for a volume to be created, either by external provisioner "example.com/nfs" or manually created by system administrator
```

### 4、查看的 Pod 的全部日志

```
kubectl logs -l app=nfs-provisioner  # 查看与此标签匹配的 Pod 的日志 ，可以发现它一直说 ServiceAccount 没有权限列出 StorageClass 资源，这导致它无法获取存储类信息，从而无法动态创建 PV，最终导致 PVC 处于 Pending 状态。

E1029 07:47:11.043320       1 reflector.go:178] pkg/mod/k8s.io/client-go@v0.18.0/tools/cache/reflector.go:125: Failed to list *v1.StorageClass: storageclasses.storage.k8s.io is forbidden: User "system:serviceaccount:default:nfs-client-provisioner" cannot list resource "storageclasses" in API group "storage.k8s.io" at the cluster scope
```

### 2、找到错误并修正

因为显示 ServiceAccount 没有权限列出 StorageClass资源，可能是在创建ServiceAccount 时权限搞错了

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-provisioner-runner
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "update"]
- apiGroups: [""]
  resources: ["storageclasses"]     
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "update", "patch"]
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get"]
- apiGroups: ["policy"]
  resources: ["podsecuritypolicies"]
  verbs: ["use"]

```

找到StorageClass，可以发现权限并没有问题，但是apiGroups为空，但是pod的错误日志显示这里的权限有问题，可以发现后面说的in API group "storage.k8s.io"，在这个组里面，但是我们的组时空的，所以问题出在这里，添加storage.k8s.io后，再次查看pvc是否对接

## 3、问题解决

### 1、查看pvc的状态

```
[root@master contest]# kubectl get pvc
NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-data-es-cluster-0   Bound    pvc-c66abbda-c3c5-4ed5-8fdc-62dd5527260a   5Gi        RWO            storage        27m
```

### 2、查看的 Pod 的全部日志（已经成功，没有权限问题）

```
kubectl logs -l app=nfs-provisioner

I1029 07:57:05.944098       1 controller.go:869] Started provisioner controller example.com/nfs_nfs-provisioner-6f67ccdd94-jtszd_46c819b4-fe78-4e11-be25-66f51f7c0f5d!
I1029 07:57:05.944842       1 controller.go:1317] provision "default/elasticsearch-data-es-cluster-0" class "storage": started
I1029 07:57:05.950236       1 event.go:278] Event(v1.ObjectReference{Kind:"PersistentVolumeClaim", Namespace:"default", Name:"elasticsearch-data-es-cluster-0", UID:"c66abbda-c3c5-4ed5-8fdc-62dd5527260a", APIVersion:"v1", ResourceVersion:"1269801", FieldPath:""}): type: 'Normal' reason: 'Provisioning' External provisioner is provisioning volume for claim "default/elasticsearch-data-es-cluster-0"
I1029 07:57:05.956139       1 controller.go:1420] provision "default/elasticsearch-data-es-cluster-0" class "storage": volume "pvc-c66abbda-c3c5-4ed5-8fdc-62dd5527260a" provisioned
I1029 07:57:05.956194       1 controller.go:1437] provision "default/elasticsearch-data-es-cluster-0" class "storage": succeeded
I1029 07:57:05.956205       1 volume_store.go:212] Trying to save persistentvolume "pvc-c66abbda-c3c5-4ed5-8fdc-62dd5527260a"
I1029 07:57:05.967724       1 volume_store.go:219] persistentvolume "pvc-c66abbda-c3c5-4ed5-8fdc-62dd5527260a" saved
I1029 07:57:05.968325       1 event.go:278] Event(v1.ObjectReference{Kind:"PersistentVolumeClaim", Namespace:"default", Name:"elasticsearch-data-es-cluster-0", UID:"c66abbda-c3c5-4ed5-8fdc-62dd5527260a", APIVersion:"v1", ResourceVersion:"1269801", FieldPath:""}): type: 'Normal' reason: 'ProvisioningSucceeded' Successfully provisioned volume pvc-c66abbda-c3c5-4ed5-8fdc-62dd5527260a
```

