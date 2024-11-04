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

# Service and endpoint

## 映射外部服务案例

### 1、在node节点安装mariadb服务，并且启动

```
yum install mariadb-server -y
systemctl start mariadb
```

### 2、在master节点编写service服务的yaml文件

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  type: ClusterIP
  ports:
  - port: 3306
```

可以发现  endpoints为none，没有绑定endpoint

```
❯ kubectl describe service mysql
Name:              mysql
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          <none>
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.194.174
IPs:               10.96.194.174
Port:              <unset>  3306/TCP
TargetPort:        3306/TCP
Endpoints:         <none>
Session Affinity:  None
Events:            <none>
```

### 3、在master节点编写endpoint服务

```
apiVersion: v1
kind: Endpoints
metadata:
  name: mysql
subsets:
- addresses:
  - ip: 192.168.40.182
  ports:
  - port: 3306
```

之后可以发现已经绑定外部数据库

```
❯ kubectl describe service mysql
Name:              mysql
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          <none>
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.194.174
IPs:               10.96.194.174
Port:              <unset>  3306/TCP
TargetPort:        3306/TCP
Endpoints:         192.168.100.30:3306
Session Affinity:  None
Events:            <none>
```

_**将外部IP地址和服务引入到k8s集群内部，由service作为一个代理来达到能够访问外部服务的目的。**_
