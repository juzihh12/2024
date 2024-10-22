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

# Pod

## 一、Pod是什么？

Pod是Kubernetes中的最小调度单元，k8s是通过定义一个Pod的资源，然后在Pod里面运行容器，容器需要指定一个镜像，这样就可以用来运行具体的服务。一个Pod封装一个容器（也可以封装多个容器），Pod里的容器共享存储、网络等。也就是说，应该把整个pod看作虚拟机，然后每个容器相当于运行在虚拟机的进程。

Pod中可以同时运行多个容器。同一个Pod中的容器会自动的分配到同一个 node 上。同一个Pod中的容器共享资源、网络环境，它们总是被同时调度，在一个Pod中同时运行多个容器是一种比较高级的用法，只有当你的容器需要紧密配合协作的时候才考虑用这种模式。例如，你有一个容器作为web服务器运行，需要用到共享的volume，有另一个“sidecar”容器来从远端获取资源更新这些文件。

## 二、创建简单pod示例

### 1、编写yaml文件

```
apiVersion: v1  #api版本
kind: Pod       #创建的资源
metadata:    
  name: tomcat-test  #Pod的名字
  namespace: default   #Pod所在的名称空间
  labels:
    app:  tomcat     #Pod具有的标签
spec:
  containers:
  - name:  tomcat-java   #Pod里容器的名字
    ports:
    - containerPort: 8080  #容器暴露的端口
    image: xianchao/tomcat-8.5-jre8:v1  #容器使用的镜像
  imagePullPolicy: IfNotPresent    #镜像拉取策略
```

### 2、pod的工作流程

<figure><img src="../../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

### 3、pod的基础操作

\#更新资源清单文件

```
kubectl apply -f pod-first.yaml
```

\#查看pod是否创建成功

```
kubectl get pods -l app=tomcat
```

\#查看pod的ip和pod调度到哪个节点上

```
kubectl get pods -owide
```

\#查看pod日志

```
kubectl logs tomcat-test
```

\#进入到刚才创建的pod，刚才创建的pod名字是web

```
kubectl exec -it tomcat-test  -- /bin/bash
```

\#假如pod里有多个容器，进入到pod里的指定容器，按如下命令：

```
kubectl exec -it tomcat-test -c tomcat-java -- /bin/bash
```

\#查看pod详细信息

```
kubectl describe pods tomcat-test
```

\#查看pod具有哪些标签：

```
kubectl get pods --show-labels
```

\#删除pod

```
kubectl delete pods tomcat-test
或者
kubectl delete -f pod-first.yaml
```

### 4、通过kubectl run创建Pod

```
kubectl run tomcat --image=xianchao/tomcat-8.5-jre8:v1  --image-pull-policy='IfNotPresent'  --port=8080
```

## 三、标签选择器

### 1、nodeName

```
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: default
  labels:
    app: myapp
    env: dev
spec:
  nodeName: node1  #通过节点名称进行资源的调度
  containers:
  - name:  tomcat-pod-java
    ports:
    - containerPort: 8080
    image: tomcat:8.5-jre8-alpine
    imagePullPolicy: IfNotPresent
  - name: busybox
    image: busybox:latest
    command:
    - "/bin/sh"
    - "-c"
- "sleep 3600"
```

### 2、nodeSelector

#### 2.1 在调度的node节点上打上标签

```
kubectl label nodes node2 disk=ceph
```

#### 2.2 编写yaml文件指定调度到具有标签disk=ceph节点上

```
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod-1
  namespace: default
  labels:
    app: myapp
    env: dev
spec:
  nodeSelector:   # 指定pod调度到具有哪些标签的node节点上
    disk: ceph   
  containers:
  - name:  tomcat-pod-java
    ports:
    - containerPort: 8080
    image: tomcat:8.5-jre8-alpine
    imagePullPolicy: IfNotPresent
```

#### 2.3 删除node节点打的标签

```
kubectl label nodes node2 disk-
```

### 3、如果两个调度同时存在的话

`同一个yaml文件里定义pod资源，如果同时定义了nodeName和NodeSelector，那么条件必须都满足才可以，有一个不满足都会调度失败`

## 四、污点、容忍度、亲和性

### 1、node节点亲和性

`node节点亲和性调度：nodeAffinity`

&#x20;  `preferredDuringSchedulingIgnoredDuringExecution ： 尽量满足这个定义的亲和性，软亲和性`

&#x20;    `matchExpressions：匹配表达式的`

&#x20;    `matchFields： 匹配字段的`

&#x20;  `requiredDuringSchedulingIgnoredDuringExecution ： 必须有节点满足这个定义的亲和性，硬亲和性`

&#x20;    `key：检查label`

&#x20;    `operator：做等值选则还是不等值选则`

&#x20;    `values：给定值`

#### 1、requiredDuringSchedulingIgnoredDuringExecution&#x20;

```
apiVersion: v1
kind: Pod
metadata:
  name:  pod-node-affinity-demo
  namespace: default
  labels:
    app: myapp
    tier: frontend
spec:
  affinity:
    nodeAffinity:
     requiredDuringSchedulingIgnoredDuringExecution:
       nodeSelectorTerms:
       - matchExpressions:
         - key: zone
           operator: In
           values: 
           - foo
           - bar
  containers:
  - name: myapp
    image: docker.io/ikubernetes/myapp:v1
    imagePullPolicy: IfNotPresent
```

`当前节点中有任意一个节点拥有zone标签的值是foo或者bar，就可以把pod调度到这个node节点的foo或者bar标签上的节点上`

#### 2、preferredDuringSchedulingIgnoredDuringExecution&#x20;

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-node-affinity-demo-2
  namespace: default
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: myapp
    image: docker.io/ikubernetes/myapp:v1
    imagePullPolicy: IfNotPresent
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - preference:
          matchExpressions: 
          - key: zone1
            operator: In
            values:
            - foo1
            - bar1
        weight: 10
      - preference:
          matchExpressions:
          - key: zone2
            operator: In
            values:
            - foo2
            - bar2
        weight: 20
```

`软亲和性是可以运行这个pod的，尽管没有运行这个pod的节点定义的zone1标签`

`weight是相对权重，权重越高，pod调度的几率越大`

### 2、Pod节点亲和性

`podaffinity： pod和pod更倾向腻在一起，pod和pod之间更好通信，可以提高通信效率`

`podunaffinity： pod和pod更倾向不腻在一起，如果部署两套程序，那么这两套程序更倾向于反亲和性，这样相互之间不会有影响。`

`判定哪些节点是相同位置的标准：以节点名称为标准，这个节点名称相同的表示是同一个位置，节点名称不相同的表示不是一个位置。`

`requiredDuringSchedulingIgnoredDuringExecution： 硬亲和性`

&#x20; `topologyKey：位置拓扑的键，这个是必须字段`

&#x20;   `怎么判断是不是同一个位置：`

&#x20;   `rack=rack  #使用rack的键是同一个位置`

&#x20;   `row=row1  #使用row的键是同一个位置`

&#x20; `labelSelector：`

&#x20;   `我们要判断pod跟别的pod亲和，跟哪个pod亲和，需要靠labelSelector，通过labelSelector选则一组能作  为亲和对象的pod资源`

&#x20; `namespace：`

&#x20;   `labelSelector需要选则一组资源，定义这组资源是在哪个名称空间中`

`preferredDuringSchedulingIgnoredDuringExecution：软亲和性`

#### 1、requiredDuringSchedulingIgnoredDuringExecution

例一、pod节点亲和性

`定义两个pod，第一个pod做为基准，第二个pod跟着它走`

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-first
  labels:
    app2: myapp2
    tier: frontend
spec:
    containers:
    - name: myapp
      image: ikubernetes/myapp:v1
      imagePullPolicy: IfNotPresent
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-second
  labels:
    app: backend
    tier: db
spec:
    containers:
    - name: busybox
      image: busybox:latest
      imagePullPolicy: IfNotPresent
      command: ["sh","-c","sleep 3600"]
    affinity:
      podAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
         - labelSelector:
              matchExpressions:
              - {key: app2, operator: In, values: ["myapp2"]}
           topologyKey: kubernetes.io/hostname
```

例二、pod节点反亲和性

`定义两个pod，第一个pod做为基准，第二个pod跟它调度节点相反`

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-first
  labels:
    app1: myapp1
    tier: frontend
spec:
    containers:
    - name: myapp
      image: ikubernetes/myapp:v1
      imagePullPolicy: IfNotPresent
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-second
  labels:
    app: backend
    tier: db
spec:
    containers:
    - name: busybox
      image: busybox:latest
      imagePullPolicy: IfNotPresent
      command: ["sh","-c","sleep 3600"]
    affinity:
      podAntiAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
         - labelSelector:
              matchExpressions:
              - {key: app1, operator: In, values: ["myapp1"]}
           topologyKey: kubernetes.io/hostname
```

例三、更改topologyKey

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-first
  labels:
    app3: myapp3
    tier: frontend
spec:
    containers:
    - name: myapp
      image: ikubernetes/myapp:v1
      imagePullPolicy: IfNotPresent
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-second
  labels:
    app: backend
    tier: db
spec:
    containers:
    - name: busybox
      image: busybox:latest
      imagePullPolicy: IfNotPresent
      command: ["sh","-c","sleep 3600"]
    affinity:
      podAntiAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
         - labelSelector:
              matchExpressions:
              - {key: app3 ,operator: In, values: ["myapp3"]}
           topologyKey:  zone
```

`第二个pod是pending，因为两个节点是同一个位置，现在没有不是同一个位置的了，而且我们要求反亲和性，所以就会处于pending状态，如果在反亲和性这个位置把required改成preferred，那么也会运行。`

`kubernetes.io/hostname： 表示节点的主机名，用于确保 Pod 在不同节点上运行。`

`zone： 表示可用区`

### 3、污点、容忍度

给节点打一个污点，不容忍的pod就运行不上来，污点就是定义在节点上的键值属性数据，可以定决定拒绝那些pod

`taints是键值数据，用在节点上，定义污点；`

&#x20; `effect用来定义对pod对象的排斥等级`

&#x20;   `NoSchedule：`

&#x20;   `仅影响pod调度过程，当pod能容忍这个节点污点，就可以调度到当前节点，后来这个节点的污点改了，加了一个新的污点，使得之前调度的pod不能容忍了，那这个pod会怎么处理，对现存的pod对象不产生影响`

&#x20;   `NoExecute：`

&#x20;   `既影响调度过程，又影响现存的pod对象，如果现存的pod不能容忍节点后来加的污点，这个pod就会被驱逐`

&#x20;   `PreferNoSchedule：`

&#x20;   `最好不，也可以，是NoSchedule的柔性版本`

`tolerations是键值数据，用在pod上，定义容忍度，能容忍哪些污点`

#### 1、taints

1.1 给node2创建一个污点

```
kubectl taint node node2 node-type=production:NoSchedule
```

1.2 创建一个示例pod

<pre><code>apiVersion: v1
kind: Pod
metadata:
  name: taint-pod
  namespace: default
  labels:
    tomcat:  tomcat-pod
spec:
  containers:
  - name:  taint-pod
    ports:
    - containerPort: 8080
    image: tomcat:8.5-jre8-alpine
<strong>    imagePullPolicy: IfNotPresent 
</strong></code></pre>

`我们在创建pod的时候没有容忍度，所以node2上不会有pod调度上去的`

1.3 给node1创建一个污点

```
kubectl taint node node1 node-type=dev:NoExecute
```

1.4  可以发现pod状态现在为termaitering，表示已经被驱逐

1.5 创建一个具有污点的Pod

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-deploy
  namespace: default
  labels:
    app: myapp
    release: canary
spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 80
      tolerations:
      - key: "node-type"
        operator: "Equal"  # 等值匹配（key和value，effect必须和node节点定义的污点完全匹配才可以）
        value: "production"
        effect: "NoExecute"
        tolerationSeconds: 3600

```

1.6 不定义value的值

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-deploy
  namespace: default
  labels:
    app: myapp
    release: canary
spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 80
      tolerations:
      - key: "node-type"
        operator: "Exists"  
        value: ""
        effect: "NoSchedule"

```

`只要对应的键是存在的，exists，其值被自动定义成通配符`

1.7 只定义key的值

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-deploy
  namespace: default
  labels:
    app: myapp
    release: canary
spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 80
      tolerations:
      - key: "node-type"
        operator: "Exists"  
        value: ""
        effect: ""

```

`有一个node-type的键，不管值是什么，不管是什么效果，都能容忍`

#### 2、删除污点

```
kubectl taint nodes node1 node-type:NoExecute-
kubectl taint nodes node2 node-type-
```

## 五、pod的状态及其重启策略

### 1、pod的状态

#### 第一阶段：

1、挂起（Pending）：

1.1 正在创建Pod但是Pod中的容器还没有全部被创建完成，处于此状态的Pod应该检查Pod依赖的存储是否有权限挂载、镜像是否可以下载、调度是否正常等

1.2 我们在请求创建pod时，条件不满足，调度没有完成，没有任何一个节点能满足调度条件，已经创建了pod但是没有适合它运行的节点叫做挂起，调度没有完成。&#x20;

2、失败（Failed）：Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。也就是说，容器以非0状态退出或者被系统终止。

3、未知（Unknown）：未知状态，所谓pod是什么状态是apiserver和运行在pod节点的kubelet进行通信获取状态信息的，如果节点之上的kubelet本身出故障，那么apiserver就连不上kubelet，得不到信息了，就会看Unknown，通常是由于与pod所在的node节点通信错误。

4、Error 状态：Pod 启动过程中发生了错误

5、成功（Succeeded）：Pod中的所有容器都被成功终止，即pod里所有的containers均已terminated。&#x20;

#### 第二阶段：

1、Unschedulable：Pod不能被调度， scheduler没有匹配到合适的node节点

2、PodScheduled：pod正处于调度中，在scheduler刚开始调度的时候，还没有将pod分配到指定的node，在筛选出合适的节点后就会更新etcd数据，将pod分配到指定的node

3、Initialized：所有pod中的初始化容器已经完成了

4、ImagePullBackOff：Pod所在的node节点下载镜像失败

5、Running：Pod内部的容器已经被创建并且启动。

#### 扩展：还有其他状态，如下：

1、Evicted状态：出现这种情况，多见于系统内存或硬盘资源不足，可df-h查看docker存储所在目录的资源使用情况，如果百分比大于85%，就要及时清理下资源，尤其是一些大文件、docker镜像。

2、CrashLoopBackOff：容器曾经启动了，但可能又异常退出了



### 2、Pod的重启策略（restartPolicy）

`Always：只要容器异常退出，kubelet就会自动重启该容器。（这个是默认的重启策略）`

`OnFailure：当容器终止运行且退出码不为0时，由kubelet自动重启该容器。`

`Never：不论容器运行状态如何，kubelet都不会重启该容器。`

#### 1、Always策略

<pre><code>apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: default
  labels:
    app: myapp
spec:
  restartPolicy: Always
  containers:
  - name:  tomcat-pod-java
    ports:
    - containerPort: 8080
    image: xianchao/tomcat-8.5-jre8:v1
<strong>    imagePullPolicy: IfNotPresent
</strong></code></pre>

正常停止容器里的tomcat服务，容器重启了一次，但是pod又恢复正常了

#### 2、OnFailure策略

<pre><code>apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: default
  labels:
    app: myapp
spec:
  restartPolicy: OnFailure
  containers:
  - name:  tomcat-pod-java
    ports:
    - containerPort: 8080
    image: xianchao/tomcat-8.5-jre8:v1
<strong>    imagePullPolicy: IfNotPresent	
</strong></code></pre>

2.1 正常重启tomcat服务

```
kubectl exec -it demo-pod -- /bin/bash /usr/local/tomcat/bin/shutdown.sh
```

发现正常停止容器里的tomcat服务，退出码是0，pod里的容器不会重启

2.2 非正常重启tomcat服务

```
kubectl exec -it tomcat-pod -- /bin/bash kill 1
```

可以发现非正常停止pod里的容器，容器退出码不是0，那就会重启容器

#### 3、Nerver策略

<pre><code>apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: default
  labels:
    app: myapp
spec:
  restartPolicy: Never
  containers:
  - name:  tomcat-pod-java
    ports:
    - containerPort: 8080
    image: xianchao/tomcat-8.5-jre8:v1
<strong>    imagePullPolicy: IfNotPresent	
</strong></code></pre>

3.1 正常重启tomcat服务

```
kubectl exec -it demo-pod -- /bin/bash /usr/local/tomcat/bin/shutdown.sh
```

发现正常停止容器里的tomcat服务，pod正常运行，容器没有重启

3.2 非正常重启tomcat服务

```
kubectl exec -it tomcat-pod -- /bin/bash kill 1
```

可以发现最终容器状态是error，并且没有重启，这说明重启策略是never，那么pod里容器服务无论如何终止，都不会重启

## 六、Pod的生命周期

### 1、Pod的生命周期

`pod从开始创建到终止退出的时间范围称为Pod生命周期`

#### `1、生命周期包含以下几个重要流程：`

`创建主容器（containers）是必须的操作，初始化容器（initContainers），容器启动后钩子，启动探测、存活性探测，就绪性探测，容器停止前钩子。`

<figure><img src="../../../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

#### 2、pod在整个生命周期的过程中总会处于以下几个状态：

1、Pending：创建了pod资源并存入etcd中，但尚未完成调度。

2、ContainerCreating：Pod 的调度完成，被分配到指定 Node 上。处于容器创建的过程中。通常是在拉取镜像的过程中。

3、Running：Pod 包含的所有容器都已经成功创建，并且成功运行起来。

4、Succeeded：Pod中的所有容器都已经成功终止并且不会被重启

5、Failed：所有容器都已经终止，但至少有一个容器终止失败，也就是说容器返回了非0值的退出状态或已经被系统终止。

6、Unknown：因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败。

#### 3、pod生命周期的重要行为：

1、在启动任何容器之前，先创建pause基础容器，它初始化Pod的环境并为后续加⼊的容器提供共享的名称空间。

2.初始化容器（initcontainer）：

一个pod可以拥有任意数量的init容器。init容器是按照顺序以此执行的，并且仅当最后一个init容器执行完毕才会去启动主容器。

3.生命周期钩子：

pod允许定义两种类型的生命周期钩子，启动后(post-start)钩子和停止前(pre-stop)钩子

这些生命周期钩子是基于每个容器来指定的，和init容器不同的是，init容器是应用到整个pod。而这些钩子是针对容器的，是在容器启动后和停止前执行的。

4.容器探测：

对Pod健康状态诊断。分为三种： Startupprobe、Livenessprobe(存活性探测)， Readinessprobe(就绪性检测)

Startup(启动探测):：探测容器是否正常运行

Liveness(存活性探测)：判断容器是否处于runnning状态，根据重启策略决定是否重启容器

Readiness(就绪性检测)：判断容器是否准备就绪并对外提供服务，将容器设置为不可用，不接受service转发的请求

三种探针用于Pod检测：

&#x20;ExecAction：在容器中执行一个命令，并根据返回的状态码进行诊断，只有返回0为成功

&#x20;TCPSocketAction：通过与容器的某TCP端口尝试建立连接

&#x20;HTTPGetAction：通过向容器IP地址的某指定端口的path发起HTTP GET请求。

5.容器的重启策略：

定义是否重启Pod对象

Always：但凡Pod对象终止就重启，默认设置

OnFailure：仅在Pod出现错误时才重启

Never：从不

注：一旦Pod绑定到一个节点上，就不会被重新绑定到另一个节点上，要么重启，要么终止

6.pod的终止过程

终止过程主要分为如下几个步骤：

(1)用户发出删除 pod 命令：kubectl delete pods ，kubectl delete -f yaml

(2)Pod 对象随着时间的推移更新，在宽限期（默认情况下30秒），pod 被视为“dead”状态

(3)将 pod 标记为“Terminating”状态

(4)第三步同时运行，监控到 pod 对象为“Terminating”状态的同时启动 pod 关闭过程

(5)第三步同时进行，endpoints 控制器监控到 pod 对象关闭，将pod与service匹配的 endpoints 列表中删除

(6)如果 pod 中定义了 preStop 钩子处理程序，则 pod 被标记为“Terminating”状态时以同步的方式启动执行；若宽限期结束后，preStop 仍未执行结束，第二步会重新执行并额外获得一个2秒的小宽限期

(7)Pod 内对象的容器收到 TERM 信号

(8)宽限期结束之后，若存在任何一个运行的进程，pod 会收到 SIGKILL 信号

(9)Kubelet 请求 API Server 将此 Pod 资源宽限期设置为0从而完成删除操作

### 2、init容器（initContainers）

`按照定义的顺序依次执行，先执行初始化容器1，再执行初始化容器2，等初始化容器执行完具体操作之后初始化容器就退出了，只有所有的初始化容器执行完后，主容器才启动。`

1、使用案例

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']

apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', "sleep 2"]
  - name: init-mydb
    image: busybox:1.28
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', "sleep 2"]
  containers:
  - name: myapp-container
    image: busybox:1.28
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', 'echo containers start running! && sleep 3600']
```

2、生产案例

主容器运行nginx服务，初始化容器用来给主容器生成index.html文件

```
apiVersion: v1
kind: Pod
metadata:
  name: initnginx
spec:
  initContainers:
  - name: install
    image: docker.io/library/busybox:1.28
    imagePullPolicy: IfNotPresent
    command:
    - wget
    - "-O"
    - "/work-dir/index.html"
    - "https://www.baidu.com"
    volumeMounts:
    - name: workdir
      mountPath: /work-dir
  containers:
  - name: nginx
    image: docker.io/xianchao/nginx:v1
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  dnsPolicy: Default
  volumes:
  - name: workdir
    emptyDir: {}
```

### 3、主容器

#### 1、 容器钩子

初始化容器启动之后，开始启动主容器，在主容器启动之后有一个post start hook（容器启动后钩子）和pre stop hook（容器结束前钩子），无论启动后还是结束前所做的事我们可以把它放两个钩子，这个钩子就表示用户可以用它来钩住一些命令，非必须选项

1.1 postStart：该钩子在容器被创建后立刻执行，用于资源部署、环境准备等，如果该钩子对应的探测执行失败，则该容器会被杀死，并根据该容器的重启策略决定是否要重启该容器，这个钩子不需要传递任何参数。

1.2 preStop：该钩子在容器被删除前执行，主要用于释放资源、优雅关闭程序和通知其他系统等

#### 2、示例1

容器创建成功后，复制/sample.war到/app文件夹中。而在容器终止之前，发送HTTP请求到http://monitor.com:8080/waring，即向监控系统发送警告。

```
containers:
- image: sample:v2  
     name: war
     lifecycle：
      postStart:
       exec:
         command:
          - “cp”
          - “/sample.war”
          - “/app”
      prestop:
       httpGet:
        host: monitor.com
        path: /waring
        port: 8080
        scheme: HTTP
```

#### 3、优雅的删除资源对象

当用户请求删除含有pod的资源对象时（如RC、deployment等），K8S为了让应用程序优雅关闭（即让应用程序完成正在处理的请求后，再关闭软件），K8S提供两种信息通知：

1）、默认：K8S通知node执行docker stop命令，docker会先向容器中PID为1的进程发送系统信号SIGTERM，然后等待容器中的应用程序终止执行，如果等待时间达到设定的超时时间，或者默认超时时间（30s），会继续发送SIGKILL的系统信号强行kill掉进程。

2）、使用pod生命周期（利用PreStop回调函数），它执行在发送终止信号之前。

默认情况下，所有的删除操作的优雅退出时间都在30秒以内。kubectl delete命令支持--grace-period=的选项，以运行用户来修改默认值。0表示删除立即执行，并且立即从API中删除pod。在节点上，被设置了立即结束的的pod，仍然会给一个很短的优雅退出时间段，才会开始被强制杀死。

4、示例2

在容器启动后执行一条 Shell 命令，将字符串 `"lifecycle hookshandler"` 写入 `/usr/share/nginx/html/test.html` 文件中，在容器停止前执行命令 `nginx -s stop`，优雅地关闭 Nginx 服务

```
apiVersion: v1
kind: Pod
metadata:
  name: life-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: docker.io/xianchao/nginx:v1
    imagePullPolicy: IfNotPresent
    lifecycle:
      postStart:
         exec:
           command: ["/bin/sh", "-c","echo 'lifecycle hookshandler' > /usr/share/nginx/html/test.html"]
      preStop:
         exec:
           command:
           - "/bin/sh"
           - "-c"
           - "nginx -s stop"
```

### 4、初始化容器与主容器区别

1、初始化容器不支持 Readinessprobe,因为它们必须在Pod就绪之前运行完成

2、每个Init容器必须运行成功,下一个才能够运行

## 七、Pod的健康探测

### 1、为什么要对容器做探测

在 Kubernetes 中 Pod 是最小的计算单元，而一个 Pod 又由多个容器组成，相当于每个容器就是一个应用，应用在运行期间，可能因为某些意外情况致使程序挂掉。那么如何监控这些容器状态稳定性，保证服务在运行期间不会发生问题，发生问题后进行重启等机制，就成为了重中之重的事情，考虑到这点 kubernetes 推出了活性探针机制。有了存活性探针能保证程序在运行中如果挂掉能够自动重启，但是还有个经常遇到的问题，比如说，在Kubernetes 中启动Pod，显示明明Pod已经启动成功，且能访问里面的端口，但是却返回错误信息。还有就是在执行滚动更新时候，总会出现一段时间，Pod对外提供网络访问，但是访问却发生404，这两个原因，都是因为Pod已经成功启动，但是 Pod 的的容器中应用程序还在启动中导致，考虑到这点Kubernetes推出了就绪性探针机制。

k8s 提供了三种来实现容器探测的方法，分别是：

1、startupProbe：探测容器中的应用是否已经启动。如果提供了启动探测(startup probe)，则禁用所有其他探测，直到它成功为止。如果启动探测失败，kubelet 将杀死容器，容器服从其重启策略进行重启。如果容器没有提供启动探测，则默认状态为成功Success。

2、livenessprobe：用指定的方式（exec、tcp、http）检测pod中的容器是否正常运行，如果检测失败，则认为容器不健康，那么Kubelet将根据Pod中设置的 restartPolicy策略来判断Pod 是否要进行重启操作，如果容器配置中没有配置 livenessProbe，Kubelet 将认为存活探针探测一直为success（成功）状态。

3、readnessprobe：就绪性探针，用于检测容器中的应用是否可以接受请求，当探测成功后才使Pod对外提供网络访问，将容器标记为就绪状态，可以加到pod前端负载，如果探测失败，则将容器标记为未就绪状态，会把pod从前端负载移除。

`真正的启动顺序为startupProbe>readinessProbe和livenessProbe，readinessProbe和livenessProbe是并发关系`

目前LivenessProbe和ReadinessProbe、startupprobe探测都支持下面三种探针：

1、exec：在容器中执行指定的命令，如果执行成功，退出码为 0 则探测成功。

2、TCPSocket：通过容器的 IP 地址和端口号执行 TCP 检 查，如果能够建立 TCP 连接，则表明容器健康。

3、HTTPGet：通过容器的IP地址、端口号及路径调用 HTTP Get方法，如果响应的状态码大于等于200且小于400，则认为容器健康

探针探测结果有以下值：

1、Success：表示通过检测。

2、Failure：表示未通过检测。

3、Unknown：表示检测没有正常进行。

Pod探针相关的属性：

探针(Probe)有许多可选字段，可以用来更加精确的控制Liveness和Readiness两种探针的行为

&#x20;   initialDelaySeconds：容器启动后要等待多少秒后探针开始工作，单位“秒”，默认是 0 秒，最小值是 0

&#x20;   periodSeconds： 执行探测的时间间隔（单位是秒），默认为 10s，单位“秒”，最小值是1

&#x20;   timeoutSeconds： 探针执行检测请求后，等待响应的超时时间，默认为1，单位“秒”。

&#x20;   successThreshold：连续探测几次成功，才认为探测成功，默认为 1，在 Liveness 探针中必须为1，最小值为1。

&#x20;   failureThreshold： 探测失败的重试次数，重试一定次数后将认为失败，在 readiness 探针中，Pod会被标记为未就绪，默认为 3，最小值为 1

ReadinessProbe 和 livenessProbe探针区别：

ReadinessProbe 和 livenessProbe 可以使用相同探测方式，只是对 Pod 的处置方式不同：

readinessProbe 当检测失败后，将 Pod 的 IP:Port 从对应的 EndPoint 列表中删除。

livenessProbe 当检测失败后，将杀死容器并根据 Pod 的重启策略来决定作出对应的措施。

### &#x20;2、启动探测（startupProbe）

#### 1、exec模式

```
apiVersion: v1
kind: Pod
metadata:
  name: startupprobe
spec:
  containers:
  - name: startup
image: xianchao/tomcat-8.5-jre8:v1
imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 8080
    startupProbe:
     exec:
       command:
       - "/bin/sh"
       - "-c"
       - "aa ps aux | grep tomcat"
     initialDelaySeconds: 20 #容器启动后多久开始探测
     periodSeconds: 20 #执行探测的时间间隔
     timeoutSeconds: 10 #探针执行检测请求后，等待响应的超时时间
     successThreshold: 1 #成功多少次才算成功
     failureThreshold: 3 #失败多少次才算失败
```

#### 2、tcpsocket模式

```
apiVersion: v1
kind: Pod
metadata:
  name: startupprobe
spec:
  containers:
  - name: startup
image: xianchao/tomcat-8.5-jre8:v1
imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 8080
    startupProbe:
     tcpSocket:
       port: 8080
     initialDelaySeconds: 20 #容器启动后多久开始探测
     periodSeconds: 20 #执行探测的时间间隔
     timeoutSeconds: 10 #探针执行检测请求后，等待响应的超时时间
     successThreshold: 1 #成功多少次才算成功
     failureThreshold: 3 #失败多少次才算失败
```

#### 3、httpget模式

```
apiVersion: v1
kind: Pod
metadata:
  name: startupprobe
spec:
  containers:
  - name: startup
image: xianchao/tomcat-8.5-jre8:v1
imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 8080
    startupProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 20 #容器启动后多久开始探测
      periodSeconds: 20 #执行探测的时间间隔
      timeoutSeconds: 10 #探针执行检测请求后，等待响应的超时时间
      successThreshold: 1 #成功多少次才算成功
      failureThreshold: 3 #失败多少次才算失败
```

### 3、存活性探测（livenessProbe）

#### 1、exec模式

容器在初始化后，首先创建一个 /tmp/healthy 文件，然后执行睡眠命令，睡眠 30 秒，到时间后执行删除 /tmp/healthy 文件命令。而设置的存活探针检检测方式为执行 shell 命令，用 cat 命令输出 healthy 文件的内容，如果能成功执行这条命令，存活探针就认为探测成功，否则探测失败。在前 30 秒内，由于文件存在，所以存活探针探测时执行 cat /tmp/healthy 命令成功执行。30 秒后 healthy 文件被删除，所以执行命令失败，Kubernetes 会根据 Pod 设置的重启策略来判断，是否重启 Pod。

```
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
  labels:
    app: liveness
spec:
  containers:
  - name: liveness
image: busybox:1.28
imagePullPolicy: IfNotPresent
    args:                       #创建测试探针探测的文件
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      initialDelaySeconds: 10   #延迟检测时间
      periodSeconds: 5          #检测时间间隔
      exec:
        command:
        - cat
        - /tmp/healthy
```

#### 2、tcpsocket模式

启动的容器是一个 SpringBoot 应用，其中引用了 Actuator 组件，提供了 /actuator/health 健康检查地址，存活探针可以使用 HTTPGet 方式向服务发起请求，请求 8081 端口的 /actuator/health 路径来进行存活判断：

任何大于或等于200且小于400的代码表示探测成功。

任何其他代码表示失败。

如果探测失败，则会杀死 Pod 进行重启操作。

<pre><code>apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
  labels:
    test: liveness
spec:
  containers:
  - name: liveness
image: mydlqclub/springboot-helloworld:0.0.1
imagePullPolicy: IfNotPresent
    livenessProbe:
      initialDelaySeconds: 20   #延迟加载时间
      periodSeconds: 5          #重试时间间隔
      timeoutSeconds: 10        #超时时间设置
      httpGet:
        scheme: HTTP
        port: 8081
        path: /actuator/health
#     httpGet探测方式有如下可选的控制字段:
<strong>#       scheme: 用于连接host的协议，默认为HTTP。
</strong>#       host：要连接的主机名，默认为Pod IP，可以在http request head中设置host头部。
<strong>#       port：容器上要访问端口号或名称。
</strong>#       path：http服务器上的访问URI。
<strong>#       httpHeaders：自定义HTTP请求headers，HTTP允许重复headers。
</strong></code></pre>

#### 3、httpget模式

在容器启动 initialDelaySeconds 参数设定的时间后，kubelet 将发送第一个 livenessProbe 探针，尝试连接容器的 80 端口，如果连接失败则将杀死 Pod 重启容器。

<pre><code>apiVersion: v1
kind: Pod
metadata:
  name: liveness-tcp
  labels:
    app: liveness
spec:
  containers:
  - name: liveness
image: docker.io/xianchao/nginx:v1
imagePullPolicy: IfNotPresent
    livenessProbe:
      initialDelaySeconds: 15
      periodSeconds: 20
      tcpSocket:
<strong>        port: 80
</strong></code></pre>

### 4、就绪性探测（readinessProbe）

Pod 的ReadinessProbe 探针使用方式和 LivenessProbe 探针探测方法一样，也是支持三种，只是一个是用于探测应用的存活，一个是判断是否对外提供流量的条件。这里用一个 Springboot 项目，设置 ReadinessProbe 探测 SpringBoot 项目的 8081 端口下的 /actuator/health 接口，如果探测成功则代表内部程序以及启动，就开放对外提供接口访问，否则内部应用没有成功启动，暂不对外提供访问，直到就绪探针探测成功。

```
apiVersion: v1
kind: Service
metadata:
  name: springboot
  labels:
    app: springboot
spec:
  type: NodePort
  ports:
  - name: server
    port: 8080
    targetPort: 8080
    nodePort: 31180
  - name: management
    port: 8081
    targetPort: 8081
    nodePort: 31181
  selector:
    app: springboot
---
apiVersion: v1
kind: Pod
metadata:
  name: springboot
  labels:
    app: springboot
spec:
  containers:
  - name: springboot
image: mydlqclub/springboot-helloworld:0.0.1
imagePullPolicy: IfNotPresent
    ports:
    - name: server
      containerPort: 8080
    - name: management
      containerPort: 8081
    readinessProbe:
      initialDelaySeconds: 20   
      periodSeconds: 5          
      timeoutSeconds: 10   
      httpGet:
        scheme: HTTP
        port: 8081
        path: /actuator/health
```

### 5、ReadinessProbe + LivenessProbe +startupProbe配合使用示例

```
apiVersion: v1
kind: Service
metadata:
  name: springboot-live
  labels:
    app: springboot
spec:
  type: NodePort
  ports:
  - name: server
    port: 8080
    targetPort: 8080
    nodePort: 31180
  - name: management
    port: 8081
    targetPort: 8081
    nodePort: 31181
  selector:
    app: springboot
---
apiVersion: v1
kind: Pod
metadata:
  name: springboot-live
  labels:
    app: springboot
spec:
  containers:
  - name: springboot
    image: mydlqclub/springboot-helloworld:0.0.1
    imagePullPolicy: IfNotPresent
    ports:
    - name: server
      containerPort: 8080
    - name: management
      containerPort: 8081
    readinessProbe:
      initialDelaySeconds: 20   
      periodSeconds: 5          
      timeoutSeconds: 10   
      httpGet:
        scheme: HTTP
        port: 8081
        path: /actuator/health
    livenessProbe:
      initialDelaySeconds: 20
      periodSeconds: 5
      timeoutSeconds: 10
      httpGet:
        scheme: HTTP
        port: 8081
        path: /actuator/health
    startupProbe:
      initialDelaySeconds: 20
      periodSeconds: 5
      timeoutSeconds: 10
      httpGet:
        scheme: HTTP
        port: 8081
        path: /actuator/health

```
