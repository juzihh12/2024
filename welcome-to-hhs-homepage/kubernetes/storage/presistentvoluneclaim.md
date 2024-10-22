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

# PresistentVoluneClaim

## 一、资源概述

_**PVC是对群集中的资源的请求（pv）**_

PersistentVolumeClaim（PVC）是一个持久化存储卷，我们在创建pod时可以定义这个类型的存储卷。 它类似于一个pod。 Pod消耗节点资源，PVC消耗PV资源。 Pod可以请求特定级别的资源（CPU和内存）。 pvc在申请pv的时候也可以请求特定的大小和访问模式（例如，可以一次读写或多次只读）

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

