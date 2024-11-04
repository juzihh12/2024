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

# PersistentVolume

## 一、资源概述

_**PV是群集中的资源**_

PersistentVolume（PV）是群集中的一块存储，由管理员配置或使用存储类动态配置。 它是集群中的资源，就像pod是k8s集群资源一样。 PV是容量插件，如Volumes，其生命周期独立于使用PV的任何单个pod。

## 二、资源清单

> **capacity**: #容量\
> **volumeMode**: 存储卷模式 (默认值为 filesystem, 除了支持文件系统外（file system）也支持块设备（raw block devices）)\
> **accessModes**: 访问模式\
> **ReadWriteMany** :（RWO / 该 volume 只能被单个节点以读写的方式映射）,ReadOnlyMany (ROX / 该 volume 可以被多个节点以只读方式映射), ReadWriteMany (RWX / 该 volume 可以被多个节点以读写的方式映射)\
> **persistentVolumeReclaimPolicy**: 回收策略 ，Retain（保留）、 Recycle（回收）或者 Delete（删除）\
> **storageClassName**: 存储类 (通过设置 storageClassName 字段进行设置。如果设置了存储类，则此 PV 只能被绑定到也指定了此存储类的 PVC)\
> mountOptions: 挂接选项

## 三、更多内容查看[Persistent Storage](persistent-storage.md)
