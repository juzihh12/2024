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

# 基于Jenkins+k8s+Git+DockerHub等技术链构建企业级DevOps容器云平台

## 一、安装jenkins

1、安装nfs

```
yum install nfs-utils -y
```

2、创建一个共享目录

```
mkdir /data/v2  -p
```

3、定义目录的访问及权限

<pre><code>vim /etc/exports
<strong>    /data/v2 *(rw,no_root_squash)
</strong>exportfs -arv
</code></pre>

4、创建工作目录及命名空间

```
mkdir jenkins-k8s && cd jenkins-k8s
kubectl create namespace jenkins-k8s
```

5、创建pv存储（使用nfs）

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
  namespace: jenkins-k8s
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  nfs:
    path: /data/v2
    server: 192.168.100.10
```

6、创建pvc对接pv

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: jenkins-k8s
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

7、创建sa证书

```
kubectl create sa jenkins-sa -n jenkins-k8s
```

8、对sa账号做rbac授权

```
kubectl create clusterrolebinding jenkins-sa-cluster -n jenkins-k8s --clusterrole=cluster-admin --serviceaccount=jenkins-k8s:jenkins-sa
```

9、[通过deployment部署jenkins](jenkins-yi-zhi-chu-yu-crashloopbackoff-zhuang-tai.md)

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins-k8s
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccount: jenkins-sa
      containers:
      - name: jenkins
        image:  jenkins/jenkins:2.394
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: web
          protocol: TCP
        - containerPort: 50000
          name: agent
          protocol: TCP
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        volumeMounts:
        - name: jenkins-volume
          subPath: jenkins-home
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-volume
        persistentVolumeClaim:
          claimName: jenkins-pvc
```

10、创建service，提供外部网络访问

```
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: jenkins-k8s
  labels:
    app: jenkins
spec:
  selector:
    app: jenkins
  type: NodePort
  ports:
  - name: web
    port: 8080
    targetPort: web
    nodePort: 30002
  - name: agent
    port: 50000
    targetPort: agent
```

11、获取管理员密码

```
cat  /data/v2/jenkins-home/secrets/initialAdminPassword
5224486613f4475ca3bc3d2f54d69065
```

