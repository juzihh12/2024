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

# Ceph

## 一、Ceph的简介

_**ceph是一种开源的分布式的存储系统**_

### 1、ceph存储类型

#### 1.块存储（rbd）：

块是一个字节序列（例如，512字节的数据块）。 基于块的存储接口是使用旋转介质（如硬盘，CD，软盘甚至传统的9轨磁带）存储数据的最常用方法;Ceph块设备是精简配置，可调整大小并存储在Ceph集群中多个OSD条带化的数据。 Ceph块设备利用RADOS功能，如快照，复制和一致性。 Ceph的RADOS块设备（RBD）使用内核模块或librbd库与OSD进行交互;Ceph的块设备为内核模块或QVM等KVM以及依赖libvirt和QEMU与Ceph块设备集成的OpenStack和CloudStack等基于云的计算系统提供高性能和无限可扩展性。 可以使用同一个集群同时运行Ceph RADOS Gateway，CephFS文件系统和Ceph块设备。

linux系统中，ls /dev/下有很多块设备文件，这些文件就是我们添加硬盘时识别出来的；

_rbd_就是由Ceph集群提供出来的块设备。可以这样理解，sda是通过数据线连接到了真实的硬盘，而rbd是通过网络连接到了Ceph集群中的一块存储区域，往rbd设备文件写入数据，最终会被存储到Ceph集群的这块区域中；

总结：块设备可理解成一块硬盘，用户可以直接使用不含文件系统的块设备，也可以将其格式化成特定的文件系统，由文件系统来组织管理存储空间，从而为用户提供丰富而友好的数据操作支持。

#### 2.文件系统cephfs

Ceph文件系统（CephFS）是一个符合POSIX标准的文件系统，它使用Ceph存储集群来存储其数据。 Ceph文件系统使用与Ceph块设备相同的Ceph存储集群系统。

用户可以在块设备上创建xfs文件系统，也可以创建ext4等其他文件系统，Ceph集群实现了自己的文件系统来组织管理集群的存储空间，用户可以直接将Ceph集群的文件系统挂载到用户机上使用，

_Ceph有了块设备接口，在块设备上完全可以构建一个文件系统，那么Ceph为什么还需要文件系统接口呢？_

主要是因为应用场景的不同，Ceph的块设备具有优异的读写性能，但不能多处挂载同时读写，目前主要用在OpenStack上作为虚拟磁盘，而Ceph的文件系统接口读写性能较块设备接口差，但具有优异的共享性。

#### 3.对象存储

Ceph对象存储使用Ceph对象网关守护进程（radosgw），它是一个用于与Ceph存储集群交互的HTTP服务器。由于它提供与OpenStack Swift和Amazon S3兼容的接口，因此Ceph对象网关具有自己的用户管理。 Ceph对象网关可以将数据存储在用于存储来自Ceph文件系统客户端或Ceph块设备客户端的数据的相同Ceph存储集群中

使用方式就是通过http协议上传下载删除对象（文件即对象）。

_有了块设备接口存储和文件系统接口存储，为什么还整个对象存储呢?_

Ceph的块设备存储具有优异的存储性能但不具有共享性，而Ceph的文件系统具有共享性然而性能较块设备存储差，为什么不权衡一下存储性能和共享性，整个具有共享性而存储性能好于文件系统存储的存储呢，对象存储就这样出现了。

### 2、分布式存储的优点

高可靠：既满足存储读取不丢失，还要保证数据长期存储。 在保证部分硬件损坏后依然可以保证数据安全

高性能：读写速度快

可扩展：分布式存储的优势就是“分布式”，所谓的“分布式”就是能够将多个物理节点整合在一起形成共享的存储池，节点可以线性扩充，这样可以源源不断的通过扩充节点提升性能和扩大容量，这是传统存储阵列无法做到的

## 二、Ceph核心组件介绍

在ceph集群中，不管你是想要提供对象存储，块设备存储，还是文件系统存储，所有Ceph存储集群部署都是从设置每个Ceph节点，网络和Ceph存储开始 的。 Ceph存储集群至少需要一个Ceph Monitor，Ceph Manager和Ceph OSD（对象存储守护进程）。 运行Ceph Filesystem客户端时也需要Ceph元数据服务器。

### 1、Monitors

Ceph监视器（ceph-mon）维护集群状态的映射，包括监视器映射，管理器映射，OSD映射和CRUSH映射。这些映射是Ceph守护进程相互协调所需的关键集群状态。监视器还负责管理守护进程和客户端之间的身份验证。冗余和高可用性通常至少需要三个监视器。

### 2、Managers

Ceph Manager守护程序（ceph-mgr）负责跟踪运行时指标和Ceph集群的当前状态，包括存储利用率，当前性能指标和系统负载。 Ceph Manager守护进程还托管基于python的模块来管理和公开Ceph集群信息，包括基于Web的Ceph Dashboard和REST API。高可用性通常至少需要两名Managers。

### 3、Ceph OSD

Ceph OSD（对象存储守护进程，ceph-osd）存储数据，处理数据复制，恢复，重新平衡，并通过检查其他Ceph OSD守护进程来获取心跳，为Ceph监视器和管理器提供一些监视信息。冗余和高可用性通常至少需要3个Ceph OSD。

### 4、MDS

Ceph元数据服务器（MDS，ceph-mds）代表Ceph文件系统存储元数据（即，Ceph块设备和Ceph对象存储不使用MDS）。 Ceph元数据服务器允许POSIX文件系统用户执行基本命令（如ls，find等），而不会给Ceph存储集群带来巨大负担。

## 三、安装ceph集群

| 管理节点           | 工作节点           | 工作节点           |
| -------------- | -------------- | -------------- |
| ceph-master    | ceph-node1     | ceph-node1     |
| 192.168.100.40 | 192.168.100.50 | 192.168.100.60 |

### 1、配置静态ip

```
1、配置ceph-master节点
vim /etc/sysconfig/network-scripts/ifcfg-ens33
    TYPE=Ethernet
    BOOTPROTO=static
    NAME=ens33
    UUID=7fac90d8-db6f-4c6f-a255-d932a1e6f5b0
    DEVICE=ens33
    ONBOOT=yes
    IPADDR=192.168.100.40
    GATEWAY=192.168.100.2
    NETMASK=255.255.255.0
    DNS1=8.8.8.8
    DNS2=114.114.114.114
systemctl restart network
2、配置ceph-node1节点
vim /etc/sysconfig/network-scripts/ifcfg-ens33
    TYPE=Ethernet
    BOOTPROTO=static
    NAME=ens33
    UUID=7fac90d8-db6f-4c6f-a255-d932a1e6f5b0
    DEVICE=ens33
    ONBOOT=yes
    IPADDR=192.168.100.50
    GATEWAY=192.168.100.2
    NETMASK=255.255.255.0
    DNS1=8.8.8.8
    DNS2=114.114.114.114
systemctl restart network
3、配置ceph-node2节点
vim /etc/sysconfig/network-scripts/ifcfg-ens33
    TYPE=Ethernet
    BOOTPROTO=static
    NAME=ens33
    UUID=7fac90d8-db6f-4c6f-a255-d932a1e6f5b0
    DEVICE=ens33
    ONBOOT=yes
    IPADDR=192.168.100.60
    GATEWAY=192.168.100.2
    NETMASK=255.255.255.0
    DNS1=8.8.8.8
    DNS2=114.114.114.114
systemctl restart network
```

### 2、配置主机名

```
1、配置ceph-master节点
hostnamectl set-hostname ceph-master && bash
2、配置ceph-node1节点
hostnamectl set-hostname ceph-node1 && bash
3、配置ceph-node2节点
hostnamectl set-hostname ceph-node2 && bash
```

### 3、配置hosts文件（域名解析）

```
vim /etc/hosts
    192.168.100.40 ceph-master
    192.168.100.50 ceph-node1
    192.168.100.60 ceph-node2
# 复制到其他节点
scp /etc/hosts ceph-node1:/etc/hosts
scp /etc/hosts ceph-node2:/etc/hosts
```

### 4、配置互信（免密）

```
ssh-keygen 
ssh-copy-id ceph-node1
ssh-copy-id ceph-node2
ssh-copy-id ceph-master
```

### 5、关闭防火墙（三个节点都要操作）

```
systemctl stop firewalld  && systemctl disable firewalld  &&  systemctl status firewalld
```

### 6、关闭selinux（三个节点都要操作）

```
vi /etc/selinux/config    # 设置永久关闭
    SELINUX=permissive
setenforce permissive     # 临时关闭
```

### 7、配置Ceph安装源（三个节点都要操作）

<pre><code>yum install -y yum-utils &#x26;&#x26; sudo yum-config-manager --add-repo https://archives.fedoraproject.org/pub/archive/epel/7/x86_64/ &#x26;&#x26; sudo yum install --nogpgcheck -y epel-release &#x26;&#x26; sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 &#x26;&#x26; sudo rm /etc/yum.repos.d/dl.fedoraproject.org*
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
vim /etc/yum.repos.d/ceph.repo
<strong>    [Ceph]
</strong>    name=Ceph packages for $basearch
    baseurl=https://mirrors.tuna.tsinghua.edu.cn/ceph/rpm-mimic/el7/$basearch
    enabled=1
    gpgcheck=1
    type=rpm-md
    gpgkey=https://mirrors.tuna.tsinghua.edu.cn/ceph/keys/release.asc
    priority=1
    
    [Ceph-noarch]
    name=Ceph noarch packages
    baseurl=https://mirrors.tuna.tsinghua.edu.cn/ceph/rpm-mimic/el7/noarch
    enabled=1
    gpgcheck=1
    type=rpm-md
    gpgkey=https://mirrors.tuna.tsinghua.edu.cn/ceph/keys/release.asc
    priority=1
    
    [ceph-source]
    name=Ceph source packages
    baseurl=https://mirrors.tuna.tsinghua.edu.cn/ceph/rpm-mimic/el7/SRPMS
    enabled=1
    gpgcheck=1
    type=rpm-md
    gpgkey=https://mirrors.tuna.tsinghua.edu.cn/ceph/keys/release.asc
    priority=1
</code></pre>

### 8、配置时间同步

```
1、配置ceph-master节点
    1.1 安装chrony时间同步服务
    yum install -y chrony
    1.2 修改配置文件
        # 删除自带的服务地址，添加阿里和腾讯的地址
        server ntp1.aliyun.com iburst
        server time.cloud.tencent.com iburst
        allow 192.168.100.0/24     # 允许某个网段的chrony客户端使用本机NTP服务
    systemctl restart chronyd && systemctl enable chronyd
2、配置ceph-node1节点
    2.1 安装chrony时间同步服务
    yum install -y chrony
    2.2 修改配置文件
        # 删除自带的服务地址，添加chep-master主机地址
        server chep-master iburst
    systemctl restart chronyd && systemctl enable chronyd
3、配置ceph-node2节点
    3.1 安装chrony时间同步服务
    yum install -y chrony
    3.2 修改配置文件
        # 删除自带的服务地址，添加chep-master主机地址
        server chep-master iburst
    systemctl restart chronyd && systemctl enable chronyd
```

### 9、安装基础软件包（三个节点都要操作）

```
yum install -y yum-utils device-mapper-persistent-data lvm2 wget net-tools nfs-utils lrzsz gcc gcc-c++ make cmake libxml2-devel openssl-devel curl curl-devel unzip sudo ntp libaio-devel wget vim ncurses-devel autoconf automake zlib-devel  python-devel epel-release openssh-server socat  ipvsadm conntrack ntpdate telnet deltarpm
```

### 10、安装ceph-deploy

<pre><code><strong>1、配置ceph-master节点
</strong>yum install python-setuptools  ceph-deploy -y
yum install ceph ceph-radosgw  -y
2、配置ceph-node1节点
yum install ceph ceph-radosgw  -y
3、配置ceph-node2节点
yum install ceph ceph-radosgw  -y
</code></pre>

### 11、[创建node1节点](chuang-jian-node1-jie-dian-shi-chu-xian-yi-xia-wen-ti.md)

```
# 创建一个目录，用于保存 ceph-deploy 生成的配置文件信息的
ceph-deploy new ceph-master ceph-node1 ceph-node2
# 创建完成后，会生成Ceph配置文件、一个monitor密钥环和一个日志文件
-rw-r--r-- 1 root root   257 Oct 23 16:23 ceph.conf
-rw-r--r-- 1 root root 10981 Oct 23 16:23 ceph-deploy-ceph.log
-rw-r--r-- 1 root root  6595 Oct 23 16:18 ceph.log
-rw------- 1 root root    73 Oct 23 16:23 ceph.mon.keyring
-rw-r--r-- 1 root root    92 Jul 10  2018 rbdmap
```

### 12、安装ceph-monitor

```
1、修改ceph配置文件
#把ceph.conf配置文件里的默认副本数从3改成1 。把osd_pool_default_size = 2
加入[global]段，这样只有2个osd也能达到active+clean状态
[global]
fsid = 64405767-4b66-43e9-8346-6ff0c7ed07bd
mon_initial_members = ceph-master, ceph-node1, ceph-node2
mon_host = 192.168.100.40,192.168.100.50,192.168.100.60
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
filestore_xattr_use_omap = true
osd_pool_default_size = 2
mon clock drift allowed = 0.500
mon clock drift warn backoff = 10
# mon clock drift allowed #监视器间允许的时钟漂移量默认值0.05
# mon clock drift warn backoff #时钟偏移警告的退避指数。默认值5
# ceph对每个mon之间的时间同步延时默认要求在0.05s之间，这个时间有的时候太短了。所以如果ceph集群如果出现clock问题就检查ntp时间同步或者适当放宽这个误差时间。
# cephx是认证机制是整个 Ceph 系统的用户名/密码
2、配置初始monitor、收集所有的密钥
cd /etc/ceph && ceph-deploy mon create-initial
-rw------- 1 root root   113 Oct 23 19:09 ceph.bootstrap-mds.keyring
-rw------- 1 root root   113 Oct 23 19:09 ceph.bootstrap-mgr.keyring
-rw------- 1 root root   113 Oct 23 19:09 ceph.bootstrap-osd.keyring
-rw------- 1 root root   113 Oct 23 19:09 ceph.bootstrap-rgw.keyring
-rw------- 1 root root   151 Oct 23 19:09 ceph.client.admin.keyring
```

### 13、部署osd服务

```
ceph-deploy osd create --zap-disk --data /dev/sdb ceph-master
ceph-deploy osd create --zap-disk --data /dev/sdb ceph-node1
ceph-deploy osd create --zap-disk --data /dev/sdb ceph-node2
[root@ceph-master ceph]# ceph osd tree
ID CLASS WEIGHT  TYPE NAME            STATUS REWEIGHT PRI-AFF 
-1       0.29306 root default                                 
-3       0.09769     host ceph-master                         
 0   hdd 0.09769         osd.0            up  1.00000 1.00000 
-5       0.09769     host ceph-node1                          
 1   hdd 0.09769         osd.1            up  1.00000 1.00000 
-7       0.09769     host ceph-node2                          
 2   hdd 0.09769         osd.2            up  1.00000 1.00000 
 [root@ceph-master ceph]# ceph-deploy osd list ceph-master ceph-node1 ceph-node2
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/local/bin/ceph-deploy osd list ceph-master ceph-node1 ceph-node2
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  subcommand                    : list
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf object at 0x7efccf463e10>
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  func                          : <function osd at 0x7efccfd1c048>
[ceph_deploy.cli][INFO  ]  host                          : ['ceph-master', 'ceph-node1', 'ceph-node2']
[ceph_deploy.cli][INFO  ]  debug                         : False
[ceph-master][DEBUG ] connected to host: ceph-master 
[ceph_deploy.osd][INFO  ] Distro info: CentOS Linux 7.9.2009 Core
[ceph_deploy.osd][DEBUG ] Listing disks on ceph-master...
[ceph-master][INFO  ] Running command: /usr/sbin/ceph-volume lvm list
[ceph-master][DEBUG ] 
[ceph-master][DEBUG ] 
[ceph-master][DEBUG ] ====== osd.0 =======
[ceph-master][DEBUG ] 
[ceph-master][DEBUG ]   [block]       /dev/ceph-f1e46278-e665-46a7-9144-29460e039ac1/osd-block-2a1f37dc-e916-40e2-a1ca-bd27ce23c3e1
[ceph-master][DEBUG ] 
[ceph-master][DEBUG ]       block device              /dev/ceph-f1e46278-e665-46a7-9144-29460e039ac1/osd-block-2a1f37dc-e916-40e2-a1ca-bd27ce23c3e1
[ceph-master][DEBUG ]       block uuid                RABeWL-WDlD-NTb8-oKa0-JK5V-8wUH-VuIpDb
[ceph-master][DEBUG ]       cephx lockbox secret      
[ceph-master][DEBUG ]       cluster fsid              bc2adb69-d1e0-4579-b053-869b1c17a2c2
[ceph-master][DEBUG ]       cluster name              ceph
[ceph-master][DEBUG ]       crush device class        None
[ceph-master][DEBUG ]       encrypted                 0
[ceph-master][DEBUG ]       osd fsid                  2a1f37dc-e916-40e2-a1ca-bd27ce23c3e1
[ceph-master][DEBUG ]       osd id                    0
[ceph-master][DEBUG ]       type                      block
[ceph-master][DEBUG ]       vdo                       0
[ceph-master][DEBUG ]       devices                   /dev/sdb
[ceph-node1][DEBUG ] connected to host: ceph-node1 
[ceph_deploy.osd][INFO  ] Distro info: CentOS Linux 7.9.2009 Core
[ceph_deploy.osd][DEBUG ] Listing disks on ceph-node1...
[ceph-node1][INFO  ] Running command: /usr/sbin/ceph-volume lvm list
[ceph-node1][DEBUG ] 
[ceph-node1][DEBUG ] 
[ceph-node1][DEBUG ] ====== osd.1 =======
[ceph-node1][DEBUG ] 
[ceph-node1][DEBUG ]   [block]       /dev/ceph-efbe7edb-000d-48cb-90bc-ba31d0a95f04/osd-block-e7a9feb4-6854-438e-a2db-0af230a08509
[ceph-node1][DEBUG ] 
[ceph-node1][DEBUG ]       block device              /dev/ceph-efbe7edb-000d-48cb-90bc-ba31d0a95f04/osd-block-e7a9feb4-6854-438e-a2db-0af230a08509
[ceph-node1][DEBUG ]       block uuid                ViQC06-Gfxi-fGhu-7j92-iJw7-mqIR-FF4Kln
[ceph-node1][DEBUG ]       cephx lockbox secret      
[ceph-node1][DEBUG ]       cluster fsid              bc2adb69-d1e0-4579-b053-869b1c17a2c2
[ceph-node1][DEBUG ]       cluster name              ceph
[ceph-node1][DEBUG ]       crush device class        None
[ceph-node1][DEBUG ]       encrypted                 0
[ceph-node1][DEBUG ]       osd fsid                  e7a9feb4-6854-438e-a2db-0af230a08509
[ceph-node1][DEBUG ]       osd id                    1
[ceph-node1][DEBUG ]       type                      block
[ceph-node1][DEBUG ]       vdo                       0
[ceph-node1][DEBUG ]       devices                   /dev/sdb
[ceph-node2][DEBUG ] connected to host: ceph-node2 
[ceph_deploy.osd][INFO  ] Distro info: CentOS Linux 7.9.2009 Core
[ceph_deploy.osd][DEBUG ] Listing disks on ceph-node2...
[ceph-node2][INFO  ] Running command: /usr/sbin/ceph-volume lvm list
[ceph-node2][DEBUG ] 
[ceph-node2][DEBUG ] 
[ceph-node2][DEBUG ] ====== osd.2 =======
[ceph-node2][DEBUG ] 
[ceph-node2][DEBUG ]   [block]       /dev/ceph-407b219a-ed76-48dc-838c-eefc5ec69fc2/osd-block-524e6aa7-e552-4e4a-a121-dbf9fd2e69e0
[ceph-node2][DEBUG ] 
[ceph-node2][DEBUG ]       block device              /dev/ceph-407b219a-ed76-48dc-838c-eefc5ec69fc2/osd-block-524e6aa7-e552-4e4a-a121-dbf9fd2e69e0
[ceph-node2][DEBUG ]       block uuid                AtIDsS-Jriq-mbql-bDuZ-BJcJ-2KN1-Vl1wXy
[ceph-node2][DEBUG ]       cephx lockbox secret      
[ceph-node2][DEBUG ]       cluster fsid              bc2adb69-d1e0-4579-b053-869b1c17a2c2
[ceph-node2][DEBUG ]       cluster name              ceph
[ceph-node2][DEBUG ]       crush device class        None
[ceph-node2][DEBUG ]       encrypted                 0
[ceph-node2][DEBUG ]       osd fsid                  524e6aa7-e552-4e4a-a121-dbf9fd2e69e0
[ceph-node2][DEBUG ]       osd id                    2
[ceph-node2][DEBUG ]       type                      block
[ceph-node2][DEBUG ]       vdo                       0
[ceph-node2][DEBUG ]       devices                   /dev/sdb
```

### 14、创建Ceph文件系统

```
1、创建mds
ceph-deploy mds create master1-admin node1-monitor node2-osd
2、查看ceph当前文件系统
ceph fs ls
    No filesystems enabled
3、创建存储池
ceph osd pool create cephfs_data 128
ceph osd pool create cephfs_metadata 128
# 一个cephfs至少要求两个librados存储池，一个为data，一个为metadata。当配置这两个存储池时，注意：
# 1. 为metadata pool设置较高级别的副本级别，因为metadata的损坏可能导致整个文件系统不用
# 2. 建议，metadata pool使用低延时存储，比如SSD，因为metadata会直接影响客户端的响应速度。
4、创建文件系统
ceph fs new ceph cephfs_metadata cephfs_data
5、查看创建后的cephfs
ceph fs ls
    name: ceph, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
6、创建 MGR（管理守护进程）
ceph-deploy mgr create ceph-node1
7、查看mds节点状态
ceph mds stat
    ceph-1/1/1 up  {0=ceph-node1=up:active}, 2 up:standby
# active是活跃的，另1个是处于热备份的状态
8、查看集群状态 ceph -s
  cluster:
    id:     bc2adb69-d1e0-4579-b053-869b1c17a2c2
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-master,ceph-node1,ceph-node2
    mgr: ceph-node1(active)
    mds: ceph-1/1/1 up  {0=ceph-node1=up:active}, 2 up:standby
    osd: 3 osds: 3 up, 3 in
 
  data:
    pools:   2 pools, 256 pgs
    objects: 22  objects, 2.2 KiB
    usage:   3.0 GiB used, 297 GiB / 300 GiB avail
    pgs:     256 active+clean
```

`关于创建存储池`

`确定 pg_num 取值是强制性的，因为不能自动计算。下面是几个常用的值：`

`*少于 5 个 OSD 时可把 pg_num 设置为 128`

`*OSD 数量在 5 到 10 个时，可把 pg_num 设置为 512`

`*OSD 数量在 10 到 50 个时，可把 pg_num 设置为 4096`

`*OSD 数量大于 50 时，你得理解权衡方法、以及如何自己计算 pg_num 取值`

`*自己计算 pg_num 取值时可借助 pgcalc 工具`

`随着 OSD 数量的增加，正确的 pg_num 取值变得更加重要，因为它显著地影响着集群的行为、以及出错时的数据持久性（即灾难性事件导致数据丢失的概率）。`

