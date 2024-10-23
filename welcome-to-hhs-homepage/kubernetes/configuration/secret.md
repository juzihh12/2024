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

# Secret

## 一、Secret的概述

### 1、Secret是什么？

Configmap一般是用来存放明文数据的，如配置文件，对于一些敏感数据，如密码、私钥等数据时，要用secret类型。

Secret解决了密码、token、秘钥等敏感数据的配置问题，而不需要把这些敏感数据暴露到镜像或者Pod Spec中。Secret可以以Volume或者环境变量的方式使用。

_**要使用 secret，pod 需要引用 secret**_。

### 2、使用 secret的两种方式

作为 volume 中的文件被挂载到 pod 中的一个或者多个容器里，或者当 kubelet 为 pod 拉取镜像时使用。

### 3、secret三种可选参数

generic: 通用类型，通常用于存储密码数据。&#x20;

tls：此类型仅用于存储私钥和证书。&#x20;

docker-registry: 若要保存docker仓库的认证信息的话，就必须使用此种类型来创建。

### 4、Secret类型

Service Account：用于被 serviceaccount 引用。serviceaccout 创建时 Kubernetes 会默认创建对应的 secret。Pod 如果使用了 serviceaccount，对应的 secret 会自动挂载到 Pod 的 /run/secrets/kubernetes.io/serviceaccount 目录中。

Opaque：base64编码格式的Secret，用来存储密码、秘钥等。可以通过base64 --decode解码获得原始数据，因此安全性弱

kubernetes.io/dockerconfigjson：用来存储私有docker registry的认证信息。

## 二、Secret使用案例&#x20;

### 1、通过环境变量引入Secret

#### 1、把mysql的root用户的password创建成secret

```
 kubectl create secret generic mysql-password --from-literal=password=1111
```

`password的值是加密的，但secret的加密是一种伪加密，它仅仅是将数据做了base64的编码.`

#### 2、创建一个pod进行引用secret

<pre><code>apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
  labels:
    app: myapp
spec:
  containers:
    name: myapp
<strong>    image: ikubernetes/myapp:v1
</strong>    ports:
    - name: http
      containerPort: 80
    env:
    - name: MYSQL_ROOT_PASSWORD #它是Pod启动成功后,Pod中容器的环境变量名.
      valueFrom:
<strong>        secretKeyRef:
</strong>          name: mysql-password  #这是secret的对象名
          key: password      #它是secret中的key名
</code></pre>

3、验证服务是否正常运行

```
kubectl exec -it pod-secret -- /bin/sh
MYSQL_PASSWORD=1111
```

### 2、通过volume挂载Secret

#### 1、创建Secret

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
  labels: 
    app: secret
type: Opaque      # 必选--不然服务对接不上
data:
  username: juzihh12
  password: anV6aWhoMTIK
```

#### 2、创建pod，将Secret挂载到Volume中

```
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume
spec:
  containers:
  - name: myapp
    image: registry.cn-beijing.aliyuncs.com/google_registry/myapp:v1
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: secret-volume
      mountPath: /opt/secret
  volumes:
  - name: secret-volume
    secret:
      secretName: mysecret
```

3、验证服务是否正常运行

```
kubectl exec -it secret -- /bin/sh
cat /opt/secret/password 
juzihh12      # 密码已经被解密
```
