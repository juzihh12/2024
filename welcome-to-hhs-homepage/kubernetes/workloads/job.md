---
layout:
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: false
  pagination:
    visible: false
---

# Job

## 一、Job的概述

_**负责**_[_**批量处理**_](https://so.csdn.net/so/search?q=%E6%89%B9%E9%87%8F%E5%A4%84%E7%90%86\&spm=1001.2101.3001.7020)_**(一次要处理指定数量任务)短暂的一次性(每个任务仅运行一次就结束)任务**_

Job 会创建一个或者多个 Pod，并将继续重试 Pod 的执行，直到指定数量的 Pod 成功终止。 随着 Pod 成功结束，Job 跟踪记录成功完成的 Pod 个数。 当数量达到指定的成功个数阈值时，任务（即 Job）结束。 删除 Job 的操作会清除所创建的全部 Pod。 挂起 Job 的操作会删除 Job 的所有活跃 Pod，直到 Job 被再次恢复执行。

## 二、Job的资源清单

### sepc字段

activeDeadlineSeconds    # 设置 Job 最多可以运行的时间（以秒为单位）

backoffLimit    # 设置 Job 失败后重试的最大次数

completionMode   # 定义 Job 如何追踪任务完成情况，有两种模式：[`NonIndexe`](#user-content-fn-1)[^1]`d` 和 [`Indexed`](#user-content-fn-2)[^2]

**注意**：`Indexed` 模式要求 `.spec.completions` 必须设置，并且 `.spec.parallelism` 不能超过 100,000。

completions   # 指定 Job 需要成功运行的 Pod 数量

manualSelector  # 控制 Pod 标签和选择器是否由用户手动设置 true or false

parallelism   # 定义 Job 同时运行的最大 Pod 数量

podFailurePolicy   # 定义处理 Pod 失败的策略。可以设置触发条件和操作

selector     # 定义 Job 的 Pod 选择器

suspend  # 控制 Job 是否处于挂起状态 ，挂起true ，正常运行 false

template    #  创建的 Pod 模板（必选）

ttlSecondsAfterFinished    # 限制 Job 完成后的存活时间

## 三、Job使用案例

```
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob 
spec:
  backoffLimit: 4
  completions: 12
  parallelism: 3
  suspend: true    # 挂起job
  activeDeadlineSeconds: 100  # job的清理（设置 Job 最多可以运行的时间）
  ttlSecondsAfterFinished: 100   # 自动清理已完成 Job （状态为 Complete 或 Failed）
  template:
    spec:
      containers:
      - name: myjob 
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  podFailurePolicy:     # 定义job的失效策略
    rules:
    - action: FailJob
      onExitCodes:
        containerName: main      # 可选
        operator: In             # In 和 NotIn 二选一
        values: [42]
    - action: Ignore             # Ignore、FailJob、Count 其中之一
      onPodConditions:
      - type: DisruptionTarget   # 表示 Pod 失效
  completionMode: Indexed # 对成功策略是必需的
  successPolicy:         # 定义job的成功策略
    rules:
      - succeededIndexes: 0,2-3
        succeededCount: 1
```

## 四、job的挂起

### 1、挂起一个活跃的 Job：

```shell
kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":true}}'
```

### 2、恢复一个挂起的 Job：

```shell
kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":false}}'
```

### 3、Job 的 `status` 可以用来确定 Job 是否被挂起，或者曾经被挂起。

```shell
kubectl get jobs/myjob -o yaml
```



[^1]: 任务只需达到指定数量的成功 Pod 即完成

[^2]: 每个 Pod 都有一个唯一的索引，必须按索引完成所有 Pod
