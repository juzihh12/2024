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

# PersistentVolume

## 一、资源概述

_**PV是群集中的资源**_

PersistentVolume（PV）是群集中的一块存储，由管理员配置或使用存储类动态配置。 它是集群中的资源，就像pod是k8s集群资源一样。 PV是容量插件，如Volumes，其生命周期独立于使用PV的任何单个pod。
