# Deployment

## 一、Deployment的概述

Deployment是kubernetes中最常用的资源对象，为ReplicaSet和Pod的创建提供了一种声明式的定义方法，在Deployment对象中描述一个期望的状态，Deployment控制器就会按照一定的控制速率把实际状态改成期望状态，_**通过定义一个Deployment控制器会创建一个新的ReplicaSet控制器，通过ReplicaSet创建pod，删除Deployment控制器，也会删除Deployment控制器下对应的ReplicaSet控制器和pod资源。**_

使用Deployment而不直接创建ReplicaSet是因为Deployment对象拥有许多ReplicaSet没有的特性，例如滚动升级、金丝雀发布、蓝绿部署和回滚。

## 二、Deployment的工作原理

Deployment控制器是建立在rs之上的一个控制器，可以管理多个rs，每次更新镜像版本，都会生成一个新的rs，把旧的rs替换掉，多个rs同时存在，但是只有一个rs运行。

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption><p>replicaset</p></figcaption></figure>

rs v1控制三个pod，删除一个pod，在rs v2上重新建立一个，依次类推，直到全部都是由rs v2控制，如果rs v2有问题，还可以回滚，Deployment是建构在rs之上的，多个rs组成一个Deployment，但是只有一个rs处于活跃状态.

Deployment可以使用声明式定义[^1]，直接在命令行通过纯命令的方式完成对应资源版本的内容的修改，也就是通过打补丁的方式进行修改；Deployment能提供滚动式自定义自控制的更新；对Deployment来讲，我们在实现更新时还可以实现控制更新节奏和更新逻辑[^2]。

## 三、Deployment资源清单

### 1、spec字段

minReadySeconds： Kubernetes在等待设置的时间后才进行升级 ，如果没有设置该值，Kubernetes会假设该容器启动起来后就提供服务了

paused： 暂停，当我们更新的时候创建pod先暂停，不是立即更新

progressDeadlineSeconds： k8s 在升级过程中有可能由于各种原因升级卡住（这个时候还没有明确的升级失败），比如在拉取被墙的镜像，权限不够等错误。那么这个时候就需要有个 deadline ，在 deadline 之内如果还卡着，那么就上报这个情况，这个时候这个 Deployment 状态就被标记为 False，并且注明原因。但是它并不会阻止 Deployment 继续进行卡住后面的操作。完全由用户进行控制。

replicas： 副本数

revisionHistoryLimit： 保留的历史版本，默认为10

selector： 标签选择器，选择关联的pod（必选参数）

strategy： 更新策略

template： 定义pod的模板（必选参数）

### 2、spec.strategy字段

rollingUpdate： 定义滚动更新

type： 支持两种更新，Recreate和RollingUpdate，Recreate是重建式更新，删除一个更新一个

### 3、spec.strategy.rollingUpdate字段

maxSurge： 我们更新的过程当中最多允许超出的指定的目标副本数有几个，它有两种取值方式，第一种直接给定数量，第二种根据百分比，和期望的副本数比，超过期望副本数最大比例（或最大值），这个值调的越大，副本更新速度越快。

maxUnavailable： 最多允许几个不可用，和期望的副本数比，不可用副本数最大比例（或最大值），这个值越小，越能保证服务稳定，更新越平滑maxUnavailable：和期望的副本数比，不可用副本数最大比例（或最大值），这个值越小，越能保证服务稳定，更新越平滑。

## 四、Deployment使用案例

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: janakiramm/myapp:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80   # 容器内应用暴露的端口号
        startupProbe:
          failureThreshold: 3
          initialDelaySeconds: 20
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 10
          httpGet:
            scheme: HTTP
            port: 80
            path: /
        livenessProbe:
          failureThreshold: 3
          initialDelaySeconds: 20
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 10
          httpGet:
            scheme: HTTP
            port: 80
            path: /
        readinessProbe:
          failureThreshold: 3
          initialDelaySeconds: 20
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 10
          httpGet:
            scheme: HTTP
            port: 80
            path: /
```

## 五、Deployment的扩容、缩容和回滚

### 1、扩容（将两个副本数扩容到三个副本数）

```
kubectl scale deployment myapp-v1 --replicas=3
```

### 2、缩容（将三个副本数缩容到两个副本数）

```
kubectl scale deployment myapp-v1 --replicas=2
```

### 3、更新镜像版本（将nginx更新为nginx:1.16.1）

```
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
```

### 4、回滚（上一版本）

```
kubectl rollout undo deployment myapp-v1
```

### 5、回滚（指定版本）

```
kubectl rollout history deployment myapp-v1 --revision=1 
```

### 6、暂停

```
kubectl rollout pause deployment myapp-v1
```

### 7、恢复暂停

```
kubectl rollout resume deployment myapp-v1
```

## 六、蓝绿发布

### 1、什么是蓝绿部署？

蓝绿部署中，一共有两套系统：一套是正在提供服务系统，标记为“绿色”；另一套是准备发布的系统，标记为“蓝色”。两套系统都是功能完善的、正在运行的系统，只是系统版本和对外服务情况不同。

开发新版本，要用新版本替换线上的旧版本，在线上的系统之外，搭建了一个使用新版本代码的全新系统。 这时候，一共有两套系统在运行，正在对外提供服务的老系统是绿色系统，新部署的系统是蓝色系统。

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

蓝色系统不对外提供服务，用来做什么呢？

用来做发布前测试，测试过程中发现任何问题，可以直接在蓝色系统上修改，不干扰用户正在使用的系统。（注意，两套系统没有耦合的时候才能百分百保证不干扰）

蓝色系统经过反复的测试、修改、验证，确定达到上线标准之后，直接将用户切换到蓝色系统。

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

切换后的一段时间内，依旧是蓝绿两套系统并存，但是用户访问的已经是蓝色系统。这段时间内观察蓝色系统（新系统）工作状态，如果出现问题，直接切换回绿色系统。

当确信对外提供服务的蓝色系统工作正常，不对外提供服务的绿色系统已经不再需要的时候，蓝色系统正式成为对外提供服务系统，成为新的绿色系统。 原先的绿色系统可以销毁，将资源释放出来，用于部署下一个蓝色系统。

### 2、蓝绿部署的优势和缺点

`优点：`

`1、更新过程无需停机，风险较少`

`2、回滚方便，只需要更改路由或者切换DNS服务器，效率较高`

`缺点：`

`1、成本较高，需要部署两套环境。如果新版本中基础服务出现问题，会瞬间影响全网用户；如果新版本有问题也会影响全网用户。`

`2、需要部署两套机器，费用开销大`

`3、在非隔离的机器（Docker、VM）上操作时，可能会导致蓝绿环境被摧毁风险`

`4、负载均衡器/反向代理/路由/DNS处理不当，将导致流量没有切换过来情况出现`

### 3、案例实现

#### 1、创建蓝色部署环境

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
  namespace: deployment
spec:
  replicas: 3
  selector:
   matchLabels:
    app: myapp
    version: v1
  template:
   metadata:
    labels:
     app: myapp
     version: v1
   spec:
    containers:
    - name: myapp
      image: janakiramm/myapp:v1
      imagePullPolicy: IfNotPresent
      ports:
      - containerPort: 80
```

#### 2、创建service服务进行访问myapp服务

```
apiVersion: v1
kind: Service
metadata:
  name: myapp-lan-v1
  namespace: deployment
  labels:
    app: myapp
spec:
  ports:
  - name: http
    port: 80
    nodePort: 30062
  selector:
    app: myapp
    version: v1
  type: NodePort
```

#### 3、创建绿色部署环境

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
  namespace: blue-green
spec:
  replicas: 3
  selector:
   matchLabels:
    app: myapp
    version: v2
  template:
   metadata:
    labels:
     app: myapp
     version: v2
   spec:
    containers:
    - name: myapp
      image: janakiramm/myapp:v2
      imagePullPolicy: IfNotPresent
      ports:
      - containerPort: 80
```

#### 4、更新service服务，进行流量的转移

```
apiVersion: v1
kind: Service
metadata:
  name: myapp-lan-v1
  namespace: deployment
  labels:
    app: myapp
spec:
  ports:
  - name: http
    port: 80
    nodePort: 30062
  selector:
    app: myapp
    version: v2
  type: NodePort
```

## 七、金丝雀发布

金丝雀发布（又称灰度发布、灰度更新）：金丝雀发布一般先发1台，或者一个小比例，例如2%的服务器，主要做流量验证用，也称为金丝雀 (Canary) 测试 （国内常称灰度测试）。

### 1、什么是金丝雀发布？

简单的金丝雀测试一般通过手工测试验证，复杂的金丝雀测试需要比较完善的监控基础设施配合，通过监控指标反馈，观察金丝雀的健康状况，作为后续发布或回退的依据。 如果金丝测试通过，则把剩余的V1版本全部升级为V2版本。如果金丝雀测试失败，则直接回退金丝雀，发布失败。

### 2、金丝雀发布的优势和缺点

`优点：灵活，策略自定义，可以按照流量或具体的内容进行灰度(比如不同账号，不同参数)，出现问题不会影响全网用户`

`缺点：没有覆盖到所有的用户导致出现问题不好排查`

### 3、案例实现

#### 1、创建绿色服务

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
  namespace: blue-green
spec:
  replicas: 3
  selector:
   matchLabels:
    app: myapp
    version: v2
  template:
   metadata:
    labels:
     app: myapp
     version: v2
   spec:
    containers:
    - name: myapp
      image: janakiramm/myapp:v2
      imagePullPolicy: IfNotPresent
      ports:
      - containerPort: 80
```

#### 2、监测更新过程

```
kubectl get pods -l app=myapp -w 
```

#### 3、逐步更新镜像

```
kubectl set image deployment myapp-v1 myapp=nginx:v1  && kubectl rollout pause deployment myapp-v1
```

`上面的解释说明把myapp这个容器的镜像更新到nginx:latest版本 更新镜像之后，创建一个新的pod就立即暂停，这就是我们说的金丝雀发布；如果暂停几个小时之后没有问题，那么取消暂停，就会依次执行后面步骤，把所有pod都升级`

#### 4、恢复一个已暂停的 Deployment 滚动更新

```
kubectl rollout resume deployment myapp-v1
```

#### 5、查看replicaset

```
kubectl get replicaset
```

`可以发现多创建了一个replicaset服务，进行pod的更新`



[^1]: 扩展：声明式定义是指直接修改资源清单yaml文件，然后通过kubectl apply -f 资源清单yaml文件，就可以更改资源

[^2]: 比如说Deployment控制5个pod副本，pod的期望值是5个，但是升级的时候需要额外多几个pod，那我们控制器可以控制在5个pod副本之外还能再增加几个pod副本；比方说能多一个，但是不能少，那么升级的时候就是先增加一个，再删除一个，增加一个删除一个，始终保持pod副本数是5个；还有一种情况，最多允许多一个，最少允许少一个，也就是最多6个，最少4个，第一次加一个，删除两个，第二次加两个，删除两个，依次类推，可以自己控制更新方式，这种滚动更新需要加readinessProbe和livenessProbe探测，确保pod中容器里的应用都正常启动了才删除之前的pod。

    启动第一步，刚更新第一批就暂停了也可以；假如目标是5个，允许一个也不能少，允许最多可以10个，那一次加5个即可；这就是我们可以自己控制节奏来控制更新的方法。
