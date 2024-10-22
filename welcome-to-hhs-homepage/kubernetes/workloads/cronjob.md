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

# CronJob

## 一、CronJob的概述

_**CronJob控制器以Job控制器资源为其管控对象，并借助它管理pod资源对象**_，Job控制器定义的作业任务在其控制器资源创建之后便会立即执行，但CronJob可以以类似于Linux操作系统的周期性任务作业计划的方式控制其运行时间点及重复运行的方式。也就是说，CronJob可以在特定的时间点(反复的)去运行job任务。&#x20;

## 二、CronJob的使用案例

```
apiVersion: batch/v1beta1 # 版本号
kind: CronJob # 类型 
metadata: # 元数据
  name: # rs名称
  namespace: # 所属命名空间
  labels: #标签
    controller: cronjob
spec: # 详情描述
  schedule: # cron格式的作业调度运⾏时间点,⽤于控制任务在什么时间执⾏
  concurrencyPolicy: # 并发执⾏策略，⽤于定义前⼀次作业运⾏尚未完成时是否以及如何运⾏后⼀次的作业
  failedJobHistoryLimit: # 为失败的任务执⾏保留的历史记录数，默认为1
  successfulJobHistoryLimit: # 为成功的任务执⾏保留的历史记录数，默认为3
  startingDeadlineSeconds: # 启动作业错误的超时时⻓
  timeZone:  "Etc/UTC"  # 时区
  jobTemplate: # job控制器模板，⽤于为cronjob控制器⽣成job对象;下⾯其实就是job的定义
    metadata:
    spec:
      completions: 1
      parallelism: 1
      activeDeadlineSeconds: 30
      backoffLimit: 6
      manualSelector: true
      selector:
        matchLabels:
          app: counter-pod
        matchExpressions: 规则
        - {key: app, operator: In, values: [counter-pod]}
      template:
        metadata:
          labels:
            app: counter-pod
       spec:
         restartPolicy: Never
         containers:
         - name: counter
           image: busybox:1.30
           command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 20;done"]
```

### 1、Cron 时间表语法 <a href="#cron-schedule-syntax" id="cron-schedule-syntax"></a>

<pre><code><strong># CronJob的spec.schedule的语法
</strong><strong># ┌───────────── 分钟 (0 - 59)
</strong># │ ┌───────────── 小时 (0 - 23)
# │ │ ┌───────────── 月的某天 (1 - 31)
# │ │ │ ┌───────────── 月份 (1 - 12)
# │ │ │ │ ┌───────────── 周的某天 (0 - 6)（周日到周六）
# │ │ │ │ │                          或者是 sun，mon，tue，web，thu，fri，sat
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
</code></pre>

### 2、并发性规则

```cobol
concurrencyPolicy: 
Allow: 允许Jobs并发运⾏(默认) 
Forbid: 禁⽌并发运⾏，如果上⼀次运⾏尚未完成，则跳过下⼀次运⾏ 
Replace: 替换，取消当前正在运⾏的作业并⽤新作业替换它
```
