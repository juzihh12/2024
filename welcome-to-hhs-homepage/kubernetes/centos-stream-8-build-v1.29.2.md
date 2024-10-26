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

# CentOS Stream 8 build v1.29.2



### 1 环境规划

| 主机名     | IP              | 网关 / DNS      | CPU / 内存 | 磁盘   | 角色     | 备注 |
| ------- | --------------- | ------------- | -------- | ---- | ------ | -- |
| kmaster | 192.168.100.138 | 192.168.100.2 | 2c / 8G  | 100g | 控制节点   |    |
| knode1  | 192.168.100.139 | 192.168.100.2 | 2c / 8G  | 100g | 工作节点 1 |    |
| knode2  | 192.168.100.140 | 192.168.100.2 | 2c / 8G  | 100g | 工作节点 2 |    |

### 2 系统环境配置

使用脚本安装以下环境

````
#!/bin/bash
# CentOS stream 8 install kubenetes 1.27.0
# the number of available CPUs 1 is less than the required 2
# k8s 环境要求虚拟cpu数量至少2个
# 使用方法：在所有节点上执行该脚本，所有节点配置完成后，复制第11步语句，单独在master节点上进行集群初始化。
#1 rpm
echo '###00 Checking RPM###'
yum install -y yum-utils vim bash-completion net-tools wget
echo "00 configuration successful ^_^"
#Basic Information
echo '###01 Basic Information###'
hostname=`hostname`
hostip=$(ifconfig ens160 |grep -w "inet" |awk '{print $2}')
echo 'The Hostname is:'$hostname
echo 'The IPAddress is:'$hostip

#2 /etc/hosts
echo '###02 Checking File:/etc/hosts###'
hosts=$(cat /etc/hosts)
result01=$(echo $hosts |grep -w "${hostname}")
if [[ "$result01" != "" ]]
then
	echo "Configuration passed ^_^"
else
	echo "hostname and ip not set,configuring......"
	echo "$hostip $hostname" >> /etc/hosts
	echo "configuration successful ^_^"
fi
echo "02 configuration successful ^_^"

#3 firewall & selinux
echo '###03 Checking Firewall and SELinux###'
systemctl stop firewalld
systemctl disable firewalld
se01="SELINUX=disabled"
se02=$(cat /etc/selinux/config |grep -w "^SELINUX")
if [[ "$se01" == "$se02" ]]
then
	echo "Configuration passed ^_^"
else
	echo "SELinux Not Closed,configuring......"
	sed -i 's/^SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
	echo "configuration successful ^_^"
fi
echo "03 configuration successful ^_^"

#4 swap
echo '###04 Checking swap###'
swapoff -a
sed -i "s/^.*swap/#&/g" /etc/fstab
echo "04 configuration successful ^_^"

#5 docker-ce
echo '###05 Checking docker###'
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
echo 'list docker-ce versions'
yum list docker-ce --showduplicates | sort -r
yum install -y docker-ce
systemctl start docker 
systemctl enable docker
cat <<EOF > /etc/docker/daemon.json
{
  "registry-mirrors": ["https://cc2d8woc.mirror.aliyuncs.com"]
}
EOF
systemctl restart docker
echo "05 configuration successful ^_^"

#6 iptables
echo '###06 Checking iptables###'
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl -p /etc/sysctl.d/k8s.conf
echo "06 configuration successful ^_^"

#7 cgroup(systemd/cgroupfs)
echo '###07 Checking cgroup###'
containerd config default > /etc/containerd/config.toml
sed -i "s#registry.k8s.io/pause#registry.aliyuncs.com/google_containers/pause#g" /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
systemctl restart containerd
echo "07 configuration successful ^_^"

#8 kubenetes.repo
echo '###08 Checking repo###'
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.29/rpm/repodata/repomd.xml.key
EOF
echo "08 configuration successful ^_^"

#9 crictl
echo "Checking crictl"
cat <<EOF > /etc/crictl.yaml 
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 5
debug: false
EOF
echo "09 configuration successful ^_^"

#10 kube1.29.2
echo "Checking kube"
yum install -y kubelet-1.29.2 kubeadm-1.29.2 kubectl-1.29.2 --disableexcludes=kubernetes
systemctl enable --now kubelet
echo "10 configuration successful ^_^"
echo "Congratulations ! The basic configuration has been completed"

#11 Initialize the cluster
# 仅在master主机上做集群初始化
# kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version=v1.29.2 --pod-network-cidr=10.244.0.0/16
```
````

> 三台节点配置好主机名及 IP 地址即可，系统环境配置分别通过脚本完成，脚本及配置文件[点击此处获取](https://pan.baidu.com/s/1y2GPoOOdoM0QucZzyyxS1A?pwd=rmk8)

<pre><code><strong>[root@kmaster ~]# sh Stream8-k8s-v1.29.2.sh
</strong>[root@knode1 ~]# sh Stream8-k8s-v1.29.2.sh
[root@knode2 ~]# sh Stream8-k8s-v1.29.2.sh

***kmaster输出记录***
==============================================================================================================================
 Package                               Architecture          Version                          Repository                 Size
==============================================================================================================================
Installing:
 kubeadm                               x86_64                1.29.2-150500.1.1                kubernetes                9.7 M
 kubectl                               x86_64                1.29.2-150500.1.1                kubernetes                 10 M
 kubelet                               x86_64                1.29.2-150500.1.1                kubernetes                 19 M

......略......

Verifying        : kubeadm-1.29.2-150500.1.1.x86_64                                                                    7/10 
Verifying        : kubectl-1.29.2-150500.1.1.x86_64                                                                    8/10 
Verifying        : kubelet-1.29.2-150500.1.1.x86_64                                                                    9/10 
Verifying        : kubernetes-cni-1.3.0-150500.1.1.x86_64                                                             10/10

Installed:
  conntrack-tools-1.4.4-11.el8.x86_64       cri-tools-1.29.0-150500.1.1.x86_64         kubeadm-1.29.2-150500.1.1.x86_64      
  kubectl-1.29.2-150500.1.1.x86_64          kubelet-1.29.2-150500.1.1.x86_64           kubernetes-cni-1.3.0-150500.1.1.x86_64
  libnetfilter_cthelper-1.0.0-15.el8.x86_64 libnetfilter_cttimeout-1.0.0-11.el8.x86_64 libnetfilter_queue-1.0.4-3.el8.x86_64 
  socat-1.7.4.1-1.el8.x86_64               

Complete!
Created symlink /etc/systemd/system/multi-user.target.wants/kubelet.service → /usr/lib/systemd/system/kubelet.service.
10 configuration successful ^_^
Congratulations ! The basic configuration has been completed

***knode1和knode2输出记录与kmaster一致***


</code></pre>

### 3 集群搭建

#### 4.1 初始化集群（仅 master 节点）

> 脚本中最后一条`第 11 条命令`，单独拷贝在 `kmaster` 节点上运行

```
[root@kmaster ~]# kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version=v1.29.2 --pod-network-cidr=10.244.0.0/16
[init] Using Kubernetes version: v1.29.2
[preflight] Running pre-flight checks
	[WARNING FileExisting-tc]: tc not found in system path
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W0312 12:13:25.451708   13656 checks.go:835] detected that the sandbox image "registry.aliyuncs.com/google_containers/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.aliyuncs.com/google_containers/pause:3.9" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kmaster kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.100.138]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kmaster localhost] and IPs [192.168.100.138 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kmaster localhost] and IPs [192.168.100.138 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 10.002081 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node kmaster as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node kmaster as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: 9wo4e7.663zzjqsxfoo8rol
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.100.138:6443 --token 9wo4e7.663zzjqsxfoo8rol \
	--discovery-token-ca-cert-hash sha256:3b2314552f1c32e08d2f6947c00e295ad308175b17a04a8cd9a496a0419dc275 


```

#### 4.2 配置环境变量（仅 master 节点）

```
[root@kmaster ~]# mkdir -p $HOME/.kube
[root@kmaster ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@kmaster ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config
[root@kmaster ~]# echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> /etc/profile
[root@kmaster ~]# source /etc/profile

[root@kmaster ~]# kubectl get node
NAME      STATUS     ROLES           AGE     VERSION
kmaster   NotReady   control-plane   5m37s   v1.29.2


```

#### 4.3 工作节点加入集群（knode1 节点及 knode2 节点）

> 将 `4.1 小节` 最后生成的 `kubeadm join` 语句，分别拷贝至两个节点执行

```
[root@knode1 ~]# kubeadm join 192.168.100.138:6443 --token 9wo4e7.663zzjqsxfoo8rol \
> --discovery-token-ca-cert-hash sha256:3b2314552f1c32e08d2f6947c00e295ad308175b17a04a8cd9a496a0419dc275
[preflight] Running pre-flight checks
	[WARNING FileExisting-tc]: tc not found in system path
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

[root@knode2 ~]# kubeadm join 192.168.100.138:6443 --token 9wo4e7.663zzjqsxfoo8rol \
> --discovery-token-ca-cert-hash sha256:3b2314552f1c32e08d2f6947c00e295ad308175b17a04a8cd9a496a0419dc275
[preflight] Running pre-flight checks
	[WARNING FileExisting-tc]: tc not found in system path
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

[root@kmaster ~]# kubectl get node
NAME      STATUS     ROLES           AGE     VERSION
kmaster   NotReady   control-plane   6m59s   v1.29.2
knode1    NotReady   <none>          50s     v1.29.2
knode2    NotReady   <none>          46s     v1.29.2


```

#### 4.4 安装 calico 网络（仅 master 节点）

> 安装网络组件前，集群状态为 `NotReady`，安装后，稍等片刻，集群状态将变为 `Ready`。

* 查看集群状态

```
[root@kmaster ~]# kubectl get node
NAME      STATUS     ROLES           AGE     VERSION
kmaster   NotReady   control-plane   6m59s   v1.29.2
knode1    NotReady   <none>          50s     v1.29.2
knode2    NotReady   <none>          46s     v1.29.2


```

* 安装 Tigera Calico operator

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/tigera-operator.yaml
***如果由于网络问题无法创建，可以提前把文件内容上传到本地 Calico ***

[root@kmaster ~]# kubectl create -f tigera-operator-3-27-2.yaml 
namespace/tigera-operator created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpfilters.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/apiservers.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/imagesets.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/installations.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/tigerastatuses.operator.tigera.io created
serviceaccount/tigera-operator created
clusterrole.rbac.authorization.k8s.io/tigera-operator created
clusterrolebinding.rbac.authorization.k8s.io/tigera-operator created
deployment.apps/tigera-operator created


```

* 配置 custom-resources.yaml

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/custom-resources.yaml
***如果由于网络问题无法创建，可以提前把文件内容上传到本地（自行网盘获取，没找到？认真找一下）***

[root@kmaster ~]# vim custom-resources-3-27-2.yaml 
***更改IP地址池中的 CIDR，和 kubeadm 初始化集群中的 --pod-network-cidr 参数保持一致（配置文件已做更改）***
cidr: 10.244.0.0/16

[root@kmaster ~]# kubectl create -f custom-resources-3-27-2.yaml 
installation.operator.tigera.io/default created
apiserver.operator.tigera.io/default created

***动态查看calico容器状态，待全部running后，集群状态变为正常***
[root@kmaster ~]# watch kubectl get pods -n calico-system

NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-f88f77676-crjd9   1/1     Running   0          27m
calico-node-2w85m                         1/1     Running   0          27m
calico-node-f7wqs                         1/1     Running   0          27m
calico-node-j4mpk                         1/1     Running   0          27m
calico-typha-c5b7dff9d-9glpc              1/1     Running   0          27m
calico-typha-c5b7dff9d-n5r4c              1/1     Running   0          27m
csi-node-driver-bbf2r                     2/2     Running   0          27m
csi-node-driver-q4ctf                     2/2     Running   0          27m
csi-node-driver-zwz5p                     2/2     Running   0          27m


```

* 再次查看集群状态

```
[root@kmaster ~]# kubectl get node
NAME      STATUS   ROLES           AGE   VERSION
kmaster   Ready    control-plane   38m   v1.29.2
knode1    Ready    <none>          32m   v1.29.2
knode2    Ready    <none>          32m   v1.29.2

[root@kmaster ~]# kubectl get pod -A
NAMESPACE          NAME                                      READY   STATUS    RESTARTS   AGE
calico-apiserver   calico-apiserver-7f468f98b5-ncns4         1/1     Running   0          24m
calico-apiserver   calico-apiserver-7f468f98b5-xx6dd         1/1     Running   0          24m
calico-system      calico-kube-controllers-f88f77676-crjd9   1/1     Running   0          28m
calico-system      calico-node-2w85m                         1/1     Running   0          28m
calico-system      calico-node-f7wqs                         1/1     Running   0          28m
calico-system      calico-node-j4mpk                         1/1     Running   0          28m
calico-system      calico-typha-c5b7dff9d-9glpc              1/1     Running   0          28m
calico-system      calico-typha-c5b7dff9d-n5r4c              1/1     Running   0          28m
calico-system      csi-node-driver-bbf2r                     2/2     Running   0          28m
calico-system      csi-node-driver-q4ctf                     2/2     Running   0          28m
calico-system      csi-node-driver-zwz5p                     2/2     Running   0          28m
kube-system        coredns-857d9ff4c9-h8qbw                  1/1     Running   0          39m
kube-system        coredns-857d9ff4c9-x9hx2                  1/1     Running   0          39m
kube-system        etcd-kmaster                              1/1     Running   0          39m
kube-system        kube-apiserver-kmaster                    1/1     Running   0          39m
kube-system        kube-controller-manager-kmaster           1/1     Running   0          39m
kube-system        kube-proxy-5ddsn                          1/1     Running   0          39m
kube-system        kube-proxy-rgbhv                          1/1     Running   0          33m
kube-system        kube-proxy-rs29b                          1/1     Running   0          33m
kube-system        kube-scheduler-kmaster                    1/1     Running   0          39m
tigera-operator    tigera-operator-748c69cf45-48bpc          1/1     Running   0          29m

[root@kmaster ~]# crictl images
IMAGE                                                             TAG                 IMAGE ID            SIZE
docker.io/calico/cni                                              v3.27.2             bbf4b051c5078       88.2MB
docker.io/calico/csi                                              v3.27.2             b2c0fe47b0708       8.74MB
docker.io/calico/kube-controllers                                 v3.27.2             849ce09815546       33.4MB
docker.io/calico/node-driver-registrar                            v3.27.2             73ddb59b21918       11.2MB
docker.io/calico/node                                             v3.27.2             50df0b2eb8ffe       117MB
docker.io/calico/pod2daemon-flexvol                               v3.27.2             ea79f2d96a361       7.6MB
registry.aliyuncs.com/google_containers/coredns                   v1.11.1             cbb01a7bd410d       18.2MB
registry.aliyuncs.com/google_containers/etcd                      3.5.10-0            a0eed15eed449       56.6MB
registry.aliyuncs.com/google_containers/kube-apiserver            v1.29.2             8a9000f98a528       35.1MB
registry.aliyuncs.com/google_containers/kube-controller-manager   v1.29.2             138fb5a3a2e34       33.4MB
registry.aliyuncs.com/google_containers/kube-proxy                v1.29.2             9344fce2372f8       28.4MB
registry.aliyuncs.com/google_containers/kube-scheduler            v1.29.2             6fc5e6b7218c7       18.5MB
registry.aliyuncs.com/google_containers/pause                     3.6                 6270bb605e12e       302kB
registry.aliyuncs.com/google_containers/pause                     3.9                 e6f1816883972       322kB


```

***

### 4 部署istio

```
1、下载istio安装包并且解压
2、进入istio目录中bin目录
cd istio-1.17.2/bin/ 
3、将istioctl复制到/uar/local/bin目录下，让其可以执行
cp istioctl /usr/local/bin/
4、安装部署istio
istioctl install --set profile=demo
5、查看istio服务是否部署成功
kubectl get crd
```

### 5 部署vm

```
kubectl create namespace kubevirt
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/latest/download/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/latest/download/kubevirt-cr.yaml
```
