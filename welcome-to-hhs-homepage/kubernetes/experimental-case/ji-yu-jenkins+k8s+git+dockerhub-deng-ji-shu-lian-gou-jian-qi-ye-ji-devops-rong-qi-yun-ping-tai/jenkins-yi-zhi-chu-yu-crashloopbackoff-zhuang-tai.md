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

# jenkins一直处于CrashLoopBackOff状态

```
jenkins-7688f99f75-q4478   0/1     CrashLoopBackOff   3 (19s ago)   77s  
```

```
[root@master ~]# kubectl describe pod jenkins-59bcc77965-rcnfv -n jenkins-k8s
Name:             jenkins-59bcc77965-rcnfv
Namespace:        jenkins-k8s
Priority:         0
Service Account:  jenkins-sa
Node:             node2/192.168.100.30
Start Time:       Thu, 24 Oct 2024 03:39:22 -0400
Labels:           app=jenkins
                  pod-template-hash=59bcc77965
Annotations:      cni.projectcalico.org/containerID: 2acaf2c063f5bf13c8ac9dde0d2799720b380b4be6ae8626caeabb61201b1085
                  cni.projectcalico.org/podIP: 10.244.104.9/32
                  cni.projectcalico.org/podIPs: 10.244.104.9/32
Status:           Running
IP:               10.244.104.9
IPs:
  IP:           10.244.104.9
Controlled By:  ReplicaSet/jenkins-59bcc77965
Containers:
  jenkins:
    Container ID:   containerd://ac5ce878285db877e4dd5fccb056d7280a2f83ad945da2dee79b6bda7c69a4ba
    Image:          jenkins/jenkins:2.394
    Image ID:       docker.io/jenkins/jenkins@sha256:3e01cf833f85f9843d76b5222fe2751595da34b565b55c75923c13cca5bf7633
    Ports:          8080/TCP, 50000/TCP
    Host Ports:     0/TCP, 0/TCP
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
      Started:      Thu, 24 Oct 2024 03:39:47 -0400
      Finished:     Thu, 24 Oct 2024 03:39:47 -0400
    Ready:          False
    Restart Count:  2
    Limits:
      cpu:     1
      memory:  1Gi
    Requests:
      cpu:        500m
      memory:     512Mi
    Liveness:     http-get http://:8080/login delay=60s timeout=5s period=10s #success=1 #failure=12
    Readiness:    http-get http://:8080/login delay=60s timeout=5s period=10s #success=1 #failure=12
    Environment:  <none>
    Mounts:
      /var/jenkins_home from jenkins-volume (rw,path="jenkins-home")
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-gw9bj (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  jenkins-volume:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  jenkins-pvc
    ReadOnly:   false
  kube-api-access-gw9bj:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  37s                default-scheduler  Successfully assigned jenkins-k8s/jenkins-59bcc77965-rcnfv to node2
  Normal   Pulled     12s (x3 over 36s)  kubelet            Container image "jenkins/jenkins:2.394" already present on machine
  Normal   Created    12s (x3 over 36s)  kubelet            Created container jenkins
  Normal   Started    12s (x3 over 35s)  kubelet            Started container jenkins
  Warning  BackOff    6s (x7 over 34s)   kubelet            Back-off restarting failed container jenkins in pod jenkins-59bcc77965-rcnfv_jenkins-k8s(239577ca-0624-4436-847e-c6c7b1faacbb)
  [root@master ~]# kubectl logs jenkins-59bcc77965-rcnfv -n jenkins-k8s   # 没有权限访问
touch: cannot touch '/var/jenkins_home/copy_reference_file.log': Permission denied
Can not write to /var/jenkins_home/copy_reference_file.log. Wrong volume permissions?
```

错误原因：

&#x20;由于/data/v2共享目录的权限没有正确配置

解决办法：

因为在创建jenkins时，初始化需要访问/var/jenkins\_home/secrets/initialAdminPassword目录，需要用到jenkins用户去访问，但是文件的所有者为root，所以导致服务一直重启

```
[root@master kubernetes]# ls -ld /data/v2      # 查看/data/v2目录的权限及所有者
drwxrwxrwx 13 root root 4096 Oct 24 03:28 /data/v2
[root@master kubernetes]# docker run --rm jenkins/jenkins:2.394 id    # 查看jenkins的uid
uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins)
[root@master kubernetes]# chown -R 1000:1000 /data/v2   # 将/data/v2目录的所有者改为jenkins
[root@master kubernetes]# ls -ld /data/v2
drwxrwxrwx 13 1000 1000 4096 Oct 24 03:28 /data/v2
```
