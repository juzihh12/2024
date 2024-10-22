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

# 2024 BRICS Provincial Championships

### A模块：私有云平台部署与运维（30分）

项目需求：某企业根据自身业务需求，实施数字化转型，规划和建设数字化平台建设，平台聚焦“DevOps建运一体”和“数据驱动产品开发”，拟采用开源OpenStack搭建企业内部私有云平台。拟将该任务交给工程师A与B，分工协助完成云平台服务部署、云应用开发、云系统运维等任务。

表 1 IP 地址规划

| 设备名称   | 主机名                    | 接 口  | IP 地址                                            | 说明 |
| ------ | ---------------------- | ---- | ------------------------------------------------ | -- |
| 云服务器 1 | controller             | eth0 | <p>公网IP:********</p><p>私网IP:192.168.100.*/24</p> |    |
| eth1   | 私网IP:192.168.200.\*/24 |      |                                                  |    |
| 云服务器 2 | compute                | eth0 | <p>公网IP:********</p><p>私网IP:192.168.100.*/24</p> |    |
| eth1   | 私网IP:192.168.200.\*/24 |      |                                                  |    |

**说明：**

1.选手自行检查工位pc机硬件及网络是否正常；

2.竞赛使用集群模式进行，给每个参赛队提供华为云账号和密码及竞赛系统的账号和密码。选手通过用户名与密码分别登录华为云和考试系统；

3.竞赛用到的软件包都在云主机/root下。

4.表1中的公网IP和私网IP以自己云主机显示为准，每个人的公网IP和私网IP不同。使用第三方软件远程连接云主机，使用公网IP连接。

### 任务1 OpenStack私有云平台搭建(11分)

#### 基础环境配置(1分)

把controller节点主机名设置为controller, compute节点主机名设置为compute，修改hosts文件将IP地址映射为主机名；配置SSH免密通信；在compute节点把数据盘的80G空间分为3个空白分区，大小自定义，供后续组件使用。

在controller节点将hostnamectl && cat /etc/hosts && cat /root/.ssh/known\_hosts命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# hostnamectl set-hostname controller

\[root@compute \~]# hostnamectl set-hostname compute

\[root@controller \~]# vim /etc/hosts

10.0.3.175 controller

10.0.3.158 compute

\[root@compute \~]# vim /etc/hosts

10.0.3.175 controller

10.0.3.158 compute

\[root@controller \~]# ssh-copy-id controller

\[root@compute \~]# ssh-copy-id compute

**#答案**

\[root@controller \~]# hostnamectl && cat /etc/hosts && cat /root/.ssh/known\_hosts

Static hostname: controller

Icon name: computer-vm

Chassis: vm

Machine ID: cc2c86fe566741e6a2ff6d399c5d5daa

Boot ID: f1df36131d8247c2a6595a0823e0fb95

Virtualization: kvm

Operating System: CentOS Linux 7 (Core)

CPE OS Name: cpe:/o:centos:centos:7

Kernel: Linux 3.10.0-1160.el7.x86\_64

Architecture: x86-64

127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4

::1 localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.100.4 controller

192.168.100.227 compute

controller,192.168.100.4 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBKSPQjT1znw9npuosr7STU4luO8xXupbvL2SAnzoeWjZpbsXrP1xti1O5rM4ojYIToYf7Wk+CXrDxNbVsOsHtTQ=

compute,192.168.100.227 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBDCvv127V/+59IGWruGybKSEreW5fHFbeGinpCmyUUBKKjGVAIFGZOhxWzhD2M4ZrM7FQcAlLboqdZAm6PCrHts=

#### yum源配置(1分)

在controller节点使用提供的CentOS-7-x86\_64-DVD-2009.iso和openstack-train.tar.gz配置本地yum源local.repo，在compute节点创建FTP源ftp.repo，使用controller节点为FTP服务器。

在controller节点将yum repolist && rpm -qa | grep ftp && ssh compute "cat /etc/yum.repos.d/ftp.repo"命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# mount CentOS-7-x86\_64-DVD-2009.iso /mnt/

\[root@controller \~]# mkdir -p /opt/centos

\[root@controller \~]# cp -rvf /mnt/\* /opt/centos/

\[root@controller \~]# umount /mnt

\[root@controller \~]# tar -zxvf openstack-train.tar.gz -C /opt/

\#controller节点yum源文件http.repo

\[root@controller \~]# vim /etc/yum.repos.d/ftp.repo

\[centos]

name=centos

baseurl=file:///opt/centos

gpgcheck=0

enabled=1

\[openstack]

name=openstack

baseurl=file:///opt/openstack

gpgcheck=0

enabled=1

\[root@controller \~]# yum clean all

\[root@controller \~]# yum repolist

\[root@controller \~]# yum install vsftpd -y

\[root@controller \~]# echo anon\_root=/opt >> /etc/vsftpd/vsftpd.conf

\[root@controller \~]# systemctl restart vsftpd

\[root@controller \~]# systemctl enable vsftpd

\#compute节点yum源文件ftp.repo

\[root@compute \~]# vim /etc/yum.repos.d/ftp.repo

\[centos]

name=centos

baseurl=ftp://controller/centos

gpgcheck=0

enabled=1

\[openstack]

name=openstack

baseurl=ftp://controller/openstack

gpgcheck=0

enabled=1

\[root@compute \~]# yum clean all

\[root@compute \~]# yum repolist

**答案**

\[root@controller \~]# yum repolist && rpm -qa | grep ftp && ssh compute "cat /etc/yum.repos.d/ftp.repo"

Loaded plugins: fastestmirror

Loading mirror speeds from cached hostfile

repo id repo name status

centos centos 4,070

openstack openstack 1,008

repolist: 5,078

vsftpd-3.0.2-29.el7\_9.x86\_64

\[centos]

name=centos

baseurl=ftp://controller/centos

gpgcheck=0

enabled=1

\[openstack]

name=openstack

baseurl=ftp://controller/openstack

gpgcheck=0

enabled=1

#### 基础安装(1分)

在控制节点和计算节点分别安装openstack-shell软件包，根据表2配置两个节点脚本文件中的基本变量（配置脚本文件为/root/variable.sh），其它变量根据实际情况配置。配置完成后执行openstack-completion.sh 脚本完成基础安装。

表2 云平台配置信息

| 服务名称             | 变量           | 参数/密码  |   |
| ---------------- | ------------ | ------ | - |
| Mysql            | root         | 000000 |   |
| Keystone         | 000000       |        |   |
| Glance           | 000000       |        |   |
| Nova             | 000000       |        |   |
| Neutron          | 000000       |        |   |
| Heat             | 000000       |        |   |
| Cinder           | 000000       |        |   |
| Swift            | 000000       |        |   |
| Ceilometer       | 000000       |        |   |
| Manila           | 000000       |        |   |
| Cloudkitty       | 000000       |        |   |
| barbican         | 000000       |        |   |
| Keystone         | DOMAIN\_NAME | demo   |   |
| Admin            | 000000       |        |   |
| Rabbit           | 000000       |        |   |
| Glance           | 000000       |        |   |
| Nova             | 000000       |        |   |
| Neutron          | 000000       |        |   |
| Heat             | 000000       |        |   |
| Cinder           | 000000       |        |   |
| Swift            | 000000       |        |   |
| Ceilometer       | 000000       |        |   |
| Manila           | 000000       |        |   |
| Cloudkitty       | 000000       |        |   |
| barbican         | 000000       |        |   |
| Neutron          | Metadata     | 000000 |   |
| External Network | eth1（以实际为准）  |        |   |

在controller节点将cat /root/variable.sh | grep -Ev '^$|#|000000' && systemctl status chronyd命令的返回结果提交到答题框。

**答案**

\[root@controller \~]# cat /root/variable.sh | grep -Ev '^$|#|000000' && systemctl status chronyd

HOST\_IP=192.168.100.4

HOST\_NAME=controller

HOST\_IP\_NODE=192.168.100.227

HOST\_NAME\_NODE=compute

network\_segment\_IP=192.168.100.0/24

RABBIT\_USER=openstack

DOMAIN\_NAME=demo

INTERFACE\_IP\_HOST=192.168.100.4

INTERFACE\_IP\_NODE=192.168.100.227

INTERFACE\_NAME\_HOST=eth1

INTERFACE\_NAME\_NODE=eth1

Physical\_NAME=provider

minvlan=100

maxvlan=200

BLOCK\_DISK=sdb1

OBJECT\_DISK=sdb2

STORAGE\_LOCAL\_NET\_IP=192.168.100.227

SHARE\_DISK=sdb3

● chronyd.service - NTP client/server

Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)

Active: active (running) since Tue 2024-07-02 12:59:19 UTC; 11s ago

Docs: man:chronyd(8)

man:chrony.conf(5)

Main PID: 27410 (chronyd)

CGroup: /system.slice/chronyd.service

└─27410 /usr/sbin/chronyd

Jul 02 12:59:19 controller systemd\[1]: Starting NTP client/server...

Jul 02 12:59:19 controller chronyd\[27410]: chronyd version 3.4 starting (+CMDMON +NTP +REFCLOCK +RTC +PRIVDROP +SCF...DEBUG)

Jul 02 12:59:19 controller chronyd\[27410]: Frequency -1.278 +/- 0.487 ppm read from /var/lib/chrony/drift

Jul 02 12:59:19 controller systemd\[1]: Started NTP client/server.

Jul 02 12:59:24 controller chronyd\[27410]: Selected source 192.168.100.4

Hint: Some lines were ellipsized, use -l to show in full.

#### 数据库安装与调优(1分)

在controller节点上使用openstack-controller-mysql.sh脚本安装Mariadb、Memcached、RabbitMQ等服务。安装服务完毕后，修改/etc/my.cnf文件，完成下列要求：

(1)设置数据库支持大小写；

(2)设置数据库缓存innodb表的索引，数据，插入数据时的缓冲为4G；

(3)设置数据库的log buffer为64MB；

(4)设置数据库的redo log大小为256MB；

(5)设置数据库的redo log文件组为2。

在controller节点将cat /etc/my.cnf | grep -Ev ^'(#|$)' && source /root/variable.sh && mysql -uroot -p$DB\_PASS -e "show variables like 'innodb\_log%';"命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# openstack-controller-mysql.sh

\[root@controller \~]# vim /etc/my.cnf

\#添加

lower\_case\_table\_names = 1

innodb\_buffer\_pool\_size = 4G

innodb\_log\_buffer\_size = 64M

innodb\_log\_file\_size = 256M

innodb\_log\_files\_in\_group = 2

\[root@controller \~]# systemctl restart mariadb

**答案**

\[root@controller \~]# cat /etc/my.cnf | grep -Ev ^'(#|$)' && source /root/variable.sh && mysql -uroot -p$DB\_PASS -e "show variables like 'innodb\_log%';"

\[client-server]

\[mysqld]

symbolic-links=0

!includedir /etc/my.cnf.d

default-storage-engine = innodb

innodb\_file\_per\_table

collation-server = utf8\_general\_ci

init-connect = 'SET NAMES utf8'

character-set-server = utf8

max\_connections = 10000

lower\_case\_table\_names = 1

innodb\_buffer\_pool\_size = 4G

innodb\_log\_buffer\_size = 64M

innodb\_log\_file\_size = 256M

innodb\_log\_files\_in\_group = 2

\+-----------------------------+-----------+

\| Variable\_name | Value |

\+-----------------------------+-----------+

\| innodb\_log\_buffer\_size | 67108864 |

\| innodb\_log\_checksums | ON |

\| innodb\_log\_compressed\_pages | ON |

\| innodb\_log\_file\_size | 268435456 |

\| innodb\_log\_files\_in\_group | 2 |

\| innodb\_log\_group\_home\_dir | ./ |

\| innodb\_log\_optimize\_ddl | ON |

\| innodb\_log\_write\_ahead\_size | 8192 |

\+-----------------------------+-----------+

#### Keystone服务安装与使用(1分)

在controller节点上使用openstack-controller-keystone.sh脚本安装Keystone服务。安装完成后，使用相关命令，创建用户competition，密码为000000。

在controller节点将source /root/admin-openrc && openstack service list && openstack user list命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# openstack-controller-keystone.sh

\[root@controller \~]# source /root/admin-openrc

\[root@controller \~]# openstack user create --password 000000 competition

\+---------------------+----------------------------------+

\| Field | Value |

\+---------------------+----------------------------------+

\| domain\_id | default |

\| enabled | True |

\| id | 8911fe08fad04d75a0a7dc567f895f80 |

\| name | competition |

\| options | {} |

\| password\_expires\_at | None |

\+---------------------+----------------------------------+

**答案**

\[root@controller \~]# source /root/admin-openrc && openstack service list && openstack user list

\+----------------------------------+----------+----------+

\| ID | Name | Type |

\+----------------------------------+----------+----------+

\| c1028e54fcab4604a31a1b062c208d2b | keystone | identity |

\+----------------------------------+----------+----------+

\+----------------------------------+-------------+

\| ID | Name |

\+----------------------------------+-------------+

\| a6d86c811dbe41ac8f9bde49d950e860 | admin |

\| 216a58918927476a96ec2cc4af7bbc72 | demo |

\| 5971a102636d46ab8aef8ac3e6fd2029 | competition |

\+----------------------------------+-------------+

#### Glance安装与使用(1分)

在controller节点上使用openstack-controller-glance.sh脚本安装glance 服务。使用命令将提供的cirros-0.3.4-x86\_64-disk.img镜像上传至平台，命名为cirros，并设置最小启动需要的硬盘为10G，最小启动需要的内存为1G。

在controller节点将source /root/admin-openrc && openstack-service status glance && openstack image show cirros | sed s/\[\[:space:]]//g命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# openstack-controller-glance.sh

\[root@controller \~]# source /root/admin-openrc

\[root@controller \~]# openstack image create --container-format bare --disk-format qcow2 --public --file cirros-0.3.4-x86\_64-disk.img --min-disk 10 --min-ram 1024 cirros

**答案**

\[root@controller \~]# source /root/admin-openrc && openstack-service status glance && openstack image show cirros | sed s/\[\[:space:]]//g

MainPID=30644 Id=openstack-glance-api.service ActiveState=active

MainPID=30645 Id=openstack-glance-registry.service ActiveState=active

\+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

|Field|Value|

\+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

|checksum|ee1eca47dc88f4879d8a229cc70a07c6|

|container\_format|bare|

|created\_at|2024-07-02T13:03:40Z|

|disk\_format|qcow2|

|file|/v2/images/d3cb0019-6bc0-4340-8660-ada2d6991e9f/file|

|id|d3cb0019-6bc0-4340-8660-ada2d6991e9f|

|min\_disk|10|

|min\_ram|1024|

|name|cirros|

|owner|27e2f4b37b084ad9b853c1013de66f61|

|properties|os\_hash\_algo='sha512',os\_hash\_value='1b03ca1bc3fafe448b90583c12f367949f8b0e665685979d95b004e48574b953316799e23240f4f739d1b5eb4c4ca24d38fdc6f4f9d8247a2bc64db25d6bbdb2',os\_hidden='False'|

|protected|False|

|schema|/v2/schemas/image|

|size|13287936|

|status|active|

|tags||

|updated\_at|2024-07-02T13:03:40Z|

|virtual\_size|None|

|visibility|public|

\+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

#### Nova安装(1分)

在controller节点和compute节点上分别使用openstack-controller-nova.sh脚本、openstack-compute-nova.sh脚本安装Nova服务。安装完成后，请修改nova相关配置文件，解决因等待时间过长而导致虚拟机启动超时从而获取不到IP地址而报错失败的问题。

在controller节点将source admin-openrc && openstack-service status nova && cat /etc/nova/nova.conf | grep -Ev ^'(#|$|\\\[)'命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# openstack-controller-nova.sh

\[root@compute \~]# openstack-compute-nova.sh

\[root@controller \~]# vim /etc/nova/nova.conf

\[DEFAULT]#下添加

vif\_plugging\_is\_fatal=false

\#重启

\[root@controller \~]# systemctl restart openstack-nova-\*

**答案**

\[root@controller \~]# source admin-openrc && openstack-service status nova && cat /etc/nova/nova.conf | grep -Ev ^'(#|$|\\\[)'

MainPID=32069 Id=openstack-nova-api.service ActiveState=active

MainPID=32094 Id=openstack-nova-conductor.service ActiveState=active

MainPID=32064 Id=openstack-nova-novncproxy.service ActiveState=active

MainPID=32065 Id=openstack-nova-scheduler.service ActiveState=active

enabled\_apis = osapi\_compute,metadata

transport\_url = rabbit://openstack:000000@controller

my\_ip = 192.168.100.4

use\_neutron = True

firewall\_driver = nova.virt.firewall.NoopFirewallDriver

vif\_plugging\_is\_fatal=false

auth\_strategy = keystone

connection = mysql+pymysql://nova:000000@controller/nova\_api

connection = mysql+pymysql://nova:000000@controller/nova

api\_servers = http://controller:9292

www\_authenticate\_uri = http://controller:5000

auth\_url = http://controller:5000

memcached\_servers = controller:11211

auth\_type = password

project\_domain\_name = demo

user\_domain\_name = demo

project\_name = service

username = nova

password = 000000

lock\_path = /var/lib/nova/tmp

region\_name = RegionOne

project\_domain\_name = demo

project\_name = service

auth\_type = password

user\_domain\_name = demo

auth\_url = http://controller:5000/v3

username = placement

password = 000000

discover\_hosts\_in\_cells\_interval = 300

enabled = true

server\_listen = 192.168.100.4

server\_proxyclient\_address = 192.168.100.4

#### Neutron安装(1分)

使用提供的脚本openstack-controller-neutron.sh和openstack-compute-neutron.sh，分别在controller和compute节点上安装neutron服务。

在controller节点将source /root/admin-openrc && openstack-service status neutron && openstack network agent list命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# openstack-controller-neutron.sh

\[root@compute \~]# openstack-controller-neutron.sh

**答案**

\[root@controller \~]# source /root/admin-openrc && openstack-service status neutron && openstack network agent list

MainPID=1126 Id=neutron-dhcp-agent.service ActiveState=active

MainPID=1128 Id=neutron-l3-agent.service ActiveState=active

MainPID=1133 Id=neutron-linuxbridge-agent.service ActiveState=active

MainPID=1127 Id=neutron-metadata-agent.service ActiveState=active

MainPID=1124 Id=neutron-server.service ActiveState=active

\+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+

\| ID | Agent Type | Host | Availability Zone | Alive | State | Binary |

\+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+

\| 2065502a-aef9-4c02-ad05-6a8d59a6efc6 | Linux bridge agent | controller | None | :-) | UP | neutron-linuxbridge-agent |

\| 73ad8900-d9e2-4fac-ac96-23fd2812039c | L3 agent | controller | nova | :-) | UP | neutron-l3-agent |

\| 9e3d851b-15e5-4c91-bce1-be7b37474c06 | Linux bridge agent | compute | None | :-) | UP | neutron-linuxbridge-agent |

\+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+

#### Dashboard安装(1分)

在controller节点上使用openstack-controller-dashboard.sh脚本安装dashboad服务。安装完成后，将Dashboard中的Django数据修改为存储在文件中。

在controller节点将cat /etc/openstack-dashboard/local\_settings | grep -Ev '(#|^$)' | grep django命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# openstack-controller-dashboard.sh

\[root@controller \~]# vim /etc/openstack-dashboard/local\_settings

将SESSION\_ENGINE = 'django.contrib.sessions.backends.cache'修改为

django.contrib.sessions.backends.file

\#重启

\[root@controller \~]# systemctl restart httpd

**答案**

\[root@controller \~]# cat /etc/openstack-dashboard/local\_settings | grep -Ev '(#|^$)' | grep django

from django.utils.translation import ugettext\_lazy as \_

EMAIL\_BACKEND = 'django.core.mail.backends.console.EmailBackend'

'django': {

'django.db.backends': {

SESSION\_ENGINE = 'django.contrib.sessions.backends.file'

'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',

#### Cinder创建硬盘(1分)

在控制节点和计算节点分别使用openstack-controller-cinder.sh、openstack-compute-cinder.sh脚本安装Cinder服务，请在compute节点，对块存储进行扩容操作，即在计算节点再分出一个5G的分区，加入到cinder块存储的后端存储中去。

在compute节点将openstack-service status cinder && vgdisplay命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# openstack-controller-cinder.sh

\[root@controller \~]# openstack-compute-cinder.sh

\[root@compute \~]# lsblk

NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT

vda 253:0 0 200G 0 disk

└─vda1 253:1 0 200G 0 part /

vdb 253:16 0 200G 0 disk

├─vdb1 253:17 0 50G 0 part

│ ├─cinder--volumes-cinder--volumes--pool\_tmeta 252:0 0 48M 0 lvm

│ │ └─cinder--volumes-cinder--volumes--pool 252:2 0 47.5G 0 lvm

│ └─cinder--volumes-cinder--volumes--pool\_tdata 252:1 0 47.5G 0 lvm

│ └─cinder--volumes-cinder--volumes--pool 252:2 0 47.5G 0 lvm

├─vdb2 253:18 0 50G 0 part

├─vdb3 253:19 0 50G 0 part

├─vdb4 253:20 0 1K 0 part

└─vdb5 253:21 0 5G 0 part

\[root@compute \~]# pvcreate -f /dev/vdb5

Physical volume "/dev/vdb5" successfully created.

\[root@compute \~]# vgextend cinder-volumes /dev/vdb5

Volume group "cinder-volumes" successfully extended

\[root@compute \~]# lvextend /dev/cinder-volumes/cinder-volumes-pool /dev/vdb5

Size of logical volume cinder-volumes/cinder-volumes-pool\_tdata changed from 47.50 GiB (12160 extents) to <52.50 GiB (13439 extents).

Logical volume cinder-volumes/cinder-volumes-pool\_tdata successfully resized.

**答案**

\[root@compute \~]# openstack-service status cinder && vgdisplay

MainPID=30265 Id=openstack-cinder-volume.service ActiveState=active

\--- Volume group ---

VG Name cinder-volumes

System ID

Format lvm2

Metadata Areas 2

Metadata Sequence No 6

VG Access read/write

VG Status resizable

MAX LV 0

Cur LV 1

Open LV 0

Max PV 0

Cur PV 2

Act PV 2

VG Size 24.99 GiB

PE Size 4.00 MiB

Total PE 6398

Alloc PE / Size 6398 / 24.99 GiB

Free PE / Size 0 / 0

VG UUID EqpUz4-36e4-QsxK-iBQ1-UIde-itHv-csMmZK

1. **Swift安装(2分)**

在控制节点和计算节点上分别使用openstack-controller-swift.sh和openstack-compute-swift.sh脚本安装Swift服务。安装完成后，使用命令创建一个名叫container的容器，将cirros-0.3.4-x86\_64-disk.img镜像上传到container容器中，并设置分段存放，每一段大小为10M。

在controller节点将source /root/admin-openrc && openstack-service status swift && swift list container\_segments命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# openstack-controller-swift.sh

\[root@controller \~]# openstack-compute-swift.sh

\[root@controller \~]# swift post container

\[root@controller \~]# swift upload -S 10000000 container cirros-0.3.4-x86\_64-disk.img

cirros-0.3.4-x86\_64-disk.img segment 1

cirros-0.3.4-x86\_64-disk.img segment 0

cirros-0.3.4-x86\_64-disk.img

\[root@controller \~]# swift stat container\_segments

Account: AUTH\_55c418b9592a41d3a6402c92263315b7

Container: container\_segments

Objects: 2

Bytes: 13287936

Read ACL:

Write ACL:

Sync To:

Sync Key:

Accept-Ranges: bytes

X-Storage-Policy: Policy-0

Last-Modified: Tue, 05 Sep 2023 06:56:20 GMT

X-Timestamp: 1693896979.72677

X-Trans-Id: tx4f54354ad5f04a75b64fc-0064f6d11d

Content-Type: application/json; charset=utf-8

X-Openstack-Request-Id: tx4f54354ad5f04a75b64fc-0064f6d11d

**答案**

\[root@controller \~]# source /root/admin-openrc && openstack-service status swift && swift list container\_segments

MainPID=23848 Id=openstack-swift-proxy.service ActiveState=active

cirros-0.3.4-x86\_64-disk.img/1657759800.000000/13287936/10000000/00000000

cirros-0.3.4-x86\_64-disk.img/1657759800.000000/13287936/10000000/00000001

#### Manila服务安装与使用(1分)

在控制和计算节点上分别使用openstack-controller-manila.sh和openstack-compute-manila.sh脚本安装manila服务。安装服务后创建default\_share\_type共享类型（不使用驱动程序支持），接着创建一个大小为2G的共享存储名为share01并开放share01目录对OpenStack管理网段使用权限。

在controller节点将source /root/admin-openrc && openstack-service status manila && manila access-list share01命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# openstack-controller-manila.sh

\[root@compute \~]# openstack-compute-manila.sh

\#创建共享类型

\[root@controller \~]# manila type-create default\_share\_type false

\+----------------------+--------------------------------------+

\| Property | Value |

\+----------------------+--------------------------------------+

\| required\_extra\_specs | driver\_handles\_share\_servers : False |

\| Name | default\_share\_type |

\| Visibility | public |

\| is\_default | YES |

\| ID | 1660d2c2-7e25-4fa4-abdc-9025bd5cee95 |

\| optional\_extra\_specs | |

\| Description | None |

\+----------------------+--------------------------------------+

\#创建共享存储

\[root@controller \~]# manila create NFS 2 --name share01

\+---------------------------------------+--------------------------------------+

\| Property | Value |

\+---------------------------------------+--------------------------------------+

\| status | creating |

\| share\_type\_name | default\_share\_type |

\| description | None |

\| availability\_zone | None |

\| share\_network\_id | None |

\| share\_server\_id | None |

\| share\_group\_id | None |

\| host | |

\| revert\_to\_snapshot\_support | False |

\| access\_rules\_status | active |

\| snapshot\_id | None |

\| create\_share\_from\_snapshot\_support | False |

\| is\_public | False |

\| task\_state | None |

\| snapshot\_support | False |

\| id | 59b6d2bc-334f-42a5-8b97-c9bb84070fb1 |

\| size | 2 |

\| source\_share\_group\_snapshot\_member\_id | None |

\| user\_id | 9c75a25a9d644ed0a6a0f1c2d82550b8 |

\| name | share01 |

\| share\_type | 1660d2c2-7e25-4fa4-abdc-9025bd5cee95 |

\| has\_replicas | False |

\| replication\_type | None |

\| created\_at | 2023-09-05T07:55:55.000000 |

\| share\_proto | NFS |

\| mount\_snapshot\_support | False |

\| project\_id | 55c418b9592a41d3a6402c92263315b7 |

\| metadata | {} |

\+---------------------------------------+--------------------------------------+

\[root@controller \~]# manila access-allow share01 ip 10.0.3.0/24 --access-level rw

\+--------------+--------------------------------------+

\| Property | Value |

\+--------------+--------------------------------------+

\| access\_key | None |

\| share\_id | 59b6d2bc-334f-42a5-8b97-c9bb84070fb1 |

\| created\_at | 2023-09-05T07:56:06.000000 |

\| updated\_at | None |

\| access\_type | ip |

\| access\_to | 10.0.3.0/24 |

\| access\_level | rw |

\| state | queued\_to\_apply |

\| id | 7f496fe3-53d7-48cf-a613-833e367647f4 |

\| metadata | {} |

\+--------------+--------------------------------------+

**答案**

\[root@controller \~]# source /root/admin-openrc && openstack-service status manila && manila access-list share01

MainPID=3928 Id=openstack-manila-api.service ActiveState=active

MainPID=3931 Id=openstack-manila-scheduler.service ActiveState=active

\+--------------------------------------+-------------+------------------+--------------+--------+------------+----------------------------+------------+

\| id | access\_type | access\_to | access\_level | state | access\_key | created\_at | updated\_at |

\+--------------------------------------+-------------+------------------+--------------+--------+------------+----------------------------+------------+

\| c38b78fa-cc81-42a7-a070-761a392af43f | ip | 192.168.200.0/24 | rw | active | None | 2024-07-02T13:31:36.000000 | None |

\+--------------------------------------+-------------+------------------+--------------+--------+------------+----------------------------+------------+

### 任务2 OpenStack私有云服务运维(12分)

#### Raid磁盘阵列管理(1分)

在controller节点分出4个大小为10G的分区，使用这4个分区，创建名为/dev/md5、raid级别为5的磁盘阵列加一个热备盘。

创建完成后将mdadm -D /dev/md5命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# mdadm -C /dev/md5 -l5 -n3 /dev/sdb{1,2,3} -x1 /dev/sdb5

mdadm: Defaulting to version 1.2 metadata

mdadm: array /dev/md5 started.

**答案**

\[root@controller \~]# mdadm -D /dev/md5

/dev/md5:

Version : 1.2

Creation Time : Tue Jul 2 13:33:36 2024

Raid Level : raid5

Array Size : 20953088 (19.98 GiB 21.46 GB)

Used Dev Size : 10476544 (9.99 GiB 10.73 GB)

Raid Devices : 3

Total Devices : 4

Persistence : Superblock is persistent

Update Time : Tue Jul 2 13:37:02 2024

State : clean

Active Devices : 3

Working Devices : 4

Failed Devices : 0

Spare Devices : 1

Layout : left-symmetric

Chunk Size : 512K

Consistency Policy : resync

Name : controller:5 (local to host controller)

UUID : defd7616:d2639b68:fe9493f6:6dd71f19

Events : 18

Number Major Minor RaidDevice State

0 8 17 0 active sync /dev/sdb1

1 8 18 1 active sync /dev/sdb2

4 8 19 2 active sync /dev/sdb3

3 8 21 - spare /dev/sdb5

#### Keystone优化-优化token失效时间(1分)

openstack api server在处理请求前会校验token是否合法，除了校验token是否过期，同时还校验token是否在token失效列表里面；这个token失效列表会在本地缓存，如果过期，则会去keystone重新获取，在并发的时候，keystone会成为瓶颈点。请修改相关配置，将Keystone的失效列表缓存时间增加到原来的两倍。

在controller节点将cat /etc/keystone/keystone.conf | grep -Ev ^'(#|$|\\\[)'命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# vim /etc/keystone/keystone.conf

\[revoke]

expiration\_buffer = 3600

cache\_time = 7200

**答案**

\[root@controller \~]# cat /etc/keystone/keystone.conf | grep -Ev ^'(#|$|\\\[)'

connection = mysql+pymysql://keystone:000000@controller/keystone

expiration\_buffer = 3600

cache\_time = 7200

provider = fernet

#### Keystone权限控制(1分)

使用自行搭建的OpenStack私有云平台，修改普通用户权限，使普通用户不能对镜像进行创建和删除操作。

在controller节点将source /root/demo-openrc && openstack image create --container-format bare --disk-format qcow2 test\_CentOS7.9命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# vim /etc/glance/policy.json

\#修改

"add\_image": "role:admin",

"delete\_image": "role:admin",

**答案**

\[root@controller \~]# source /root/demo-openrc && openstack image create --container-format bare --disk-format qcow2 test\_CentOS7.9

HTTP 403 Forbidden: You are not authorized to complete add\_image action.

#### 修改文件句柄数(1分)

Linux服务器大并发时，往往需要预先调优Linux参数。默认情况下，Linux最大文件句柄数为1024个。当你的服务器在大并发达到极限时，就会报出“too many open files”。修改相关配置文件，将控制节点的最大文件句柄数永久修改为65535。

在controller节点将ulimit -n && cat /etc/security/limits.conf | grep -Ev ^'(#|$)'命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# ulimit -n 655355

\[root@controller \~]# echo "\* soft nofile 65535 " >> /etc/security/limits.conf

\[root@controller \~]# echo "\* hard nofile 65535 " >> /etc/security/limits.conf

**答案**

\[root@controller \~]# ulimit -n && cat /etc/security/limits.conf | grep -Ev ^'(#|$)'

655355

rabbitmq soft nofile 10240

rabbitmq hard nofile 10240

\* soft nofile 65535

\* hard nofile 65535

#### 镜像转换(1分)

使用自行搭建的OpenStack平台。上传CentOS\_7.9\_x86\_64\_GJ.qcow2镜像，请使用qemu相关命令，将镜像转换为raw格式镜像，转换后的镜像命名为CentOS\_7.9.raw并存放在/root 目录下。

在controller节点将qemu-img info /root/CentOS7.9.raw命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# qemu-img convert -f qcow2 -O raw CentOS\_7.9\_x86\_64\_GJ.qcow2 /root/CentOS7.9.raw

**答案**

\[root@controller \~]# qemu-img info /root/CentOS7.9.raw

image: /root/CentOS7.9.raw

file format: raw

virtual size: 8.0G (8589934592 bytes)

disk size: 953M

#### OpenStack镜像压缩(1分)

使用自行搭建的OpenStack平台。上传提供的CentOS-7-x86\_64-GenericCloud-2009.qcow2镜像，请使用qemu相关命令，对该镜像进行压缩，压缩后的镜像命名为CentOS-GenericCloud-2009.qcow2并存放在/root目录下。

将qemu-img info /root/CentOS-GenericCloud-2009.qcow2命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# qemu-img convert -c -O qcow2 CentOS-7-x86\_64-GenericCloud-2009.qcow2 /root/CentOS-GenericCloud-2009.qcow2

\[root@controller \~]# systemctl restart openstack-glance-api.service

**答案**

\[root@controller \~]# qemu-img info /root/CentOS-GenericCloud-2009.qcow2

image: CentOS-GenericCloud-2009.qcow2

file format: qcow2

virtual size: 8.0G (8589934592 bytes)

disk size: 378M

cluster\_size: 65536

Format specific information:

compat: 1.1

lazy refcounts: false

refcount bits: 16

corrupt: false

#### Nova优化-优化数据库连接(1分)

当并发业务处理需要连接数据库，并发度高的时候，提示数据库连接超过了上限 。解决思路：调整各组件的数据库连接数配置，下面通过修改nova相关配置文件，修改连接池大小和最大允许超出的连接数为10。

在controller节点将cat /etc/nova/nova.conf | grep -Ev ^'(#|$|\\\[)' | grep 10命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# vim /etc/nova/nova.conf

\[database]

max\_pool\_size=10

max\_overflow=10

\#重启服务

\[root@controller \~]# systemctl restart openstack-nova-\*

**答案**

\[root@controller \~]# cat /etc/nova/nova.conf | grep -Ev ^'(#|$|\\\[)' | grep 10

my\_ip = 192.168.100.4

max\_pool\_size=10

max\_overflow=10

server\_listen = 192.168.100.4

server\_proxyclient\_address = 192.168.100.4

#### Nova保持云主机状态(1分)

OpenStack平台若意外断电，在电力系统恢复后，OpenStack平台可以自启动，但是运行的云主机需要管理员手动开启，在OpenStack平台中配置虚拟机自启动，当宿主机启动后，把虚拟机恢复到之前的状态，如果虚拟机之前是关机，则宿主机启动后，虚拟机也是关机状态；如果虚拟机之前是开机状态，则宿主机启动后，虚拟机还是开机状态中运行的虚拟机。

在controller节点将cat /etc/nova/nova.conf | grep -Ev ^'(#|$|\\\[)' | grep true命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# vim /etc/nova/nova.conf

\[DEFAULT]

resume\_guests\_state\_on\_host\_boot=true

\[root@controller \~]# openstack-service restart nova

**答案**

\[root@controller \~]# cat /etc/nova/nova.conf | grep -Ev ^'(#|$|\\\[)' | grep true

resume\_guests\_state\_on\_host\_boot=true

service\_metadata\_proxy = true

enabled = true

1. **Nova自动清理镜像缓存(1分)**

当在openstack平台创建虚拟机时，若是第一次在计算节点创建虚拟机，会先将镜像文件复制到该计算节点目录/var/lib/nova/instances/\_base，长期下来，该目录会占用比较大的磁盘空间而需要清理，可以通过修改nova的配置文件来自动清理该缓存目录，即在该节点没有使用其镜像启动的云主机时，那么这个镜像在过一定的时间后就会被自动删除。

在修改节点将cat /etc/nova/nova.conf | grep -Ev ^'(#|$|\\\[)'命令的返回结果提交到答题框。

**操作步骤**

\[root@compute \~]# vim /etc/nova/nova.conf

\#修改

remove\_unused\_base\_images=True

remove\_unused\_original\_minimum\_age\_seconds=3

image\_cache\_manager\_interval=5

\#remove\_unused\_base\_images用于开启自动清理镜像缓存。

\#remove\_unused\_original\_minimum\_age\_seconds也是为了缩短测试时间，默认是超过247个消失的无用base镜像才会被删除。

\#image\_cache\_manager\_interval是为了缩短测试时间才需要设置，默认40分钟才检查一次

\[root@compute \~]# systemctl restart openstack-nova-compute

**答案**

\[root@compute \~]# cat /etc/nova/nova.conf | grep -Ev ^'(#|$|\\\[)'

enabled\_apis = osapi\_compute,metadata

transport\_url = rabbit://openstack:000000@controller

my\_ip = 192.168.100.10

use\_neutron = True

firewall\_driver = nova.virt.firewall.NoopFirewallDriver

remove\_unused\_base\_images=true

remove\_unused\_original\_minimum\_age\_seconds=3

reserved\_host\_memory\_mb=4096

image\_cache\_manager\_interval=5

auth\_strategy = keystone

api\_servers = http://controller:9292

www\_authenticate\_uri = http://controller:5000

auth\_url = http://controller:5000

memcached\_servers = controller:11211

auth\_type = password

project\_domain\_name = demo

user\_domain\_name = demo

project\_name = service

username = nova

password = 000000

auth\_url = http://controller:5000

auth\_type = password

project\_domain\_name = demo

user\_domain\_name = demo

region\_name = RegionOne

project\_name = service

username = neutron

password = 000000

lock\_path = /var/lib/nova/tmp

region\_name = RegionOne

project\_domain\_name = demo

project\_name = service

auth\_type = password

user\_domain\_name = demo

auth\_url = http://controller:5000/v3

username = placement

password = 000000

discover\_hosts\_in\_cells\_interval = 300

enabled = True

server\_listen = 0.0.0.0

server\_proxyclient\_address = 192.168.100.20

novncproxy\_base\_url = http://192.168.100.10:6080/vnc\_auto.html

#### Cinder限速(1分)

在使用Cinder服务的时候，为了减缓来自实例的数据访问速度的减慢，OpenStack Block Storage 支持对卷数据复制带宽的速率限制。请修改cinder后端配置文件将卷复制带宽限制为最高100 MiB/s。

在修改节点将cat /etc/cinder/cinder.conf | grep -Ev ^'(#|$|\\\[)'命令的返回结果提交到答题框。

**操作步骤**

\[root@compute \~]# vim /etc/cinder/cinder.conf

\#修改

volume\_copy\_bps\_limit = 100000000

\[root@compute \~]# systemctl restart openstack-cinder-volume\*

**答案**

\[root@compute \~]# cat /etc/cinder/cinder.conf | grep -Ev ^'(#|$|\\\[)'

transport\_url = rabbit://openstack:000000@controller

auth\_strategy = keystone

my\_ip = 192.168.100.20

enabled\_backends = lvm

glance\_api\_servers = http://controller:9292

volume\_copy\_bps\_limit = 100000000

connection = mysql+pymysql://cinder:000000@controller/cinder

www\_authenticate\_uri = http://controller:5000

auth\_url = http://controller:5000

memcached\_servers = controller:11211

auth\_type = password

project\_domain\_name = demo

user\_domain\_name = demo

project\_name = service

username = cinder

password = 000000

lock\_path = /var/lib/cinder/tmp

volume\_driver = cinder.volume.drivers.lvm.LVMVolumeDriver

volume\_group = cinder-volumes

target\_protocol = iscsi

target\_helper = lioadm

#### 使用Heat模板创建用户(2分)

在controller节点执行openstack-controller-heat.sh会自行安装heat服务并完成配置，执行完成后在/root目录下编写Heat模板create\_user.yaml，创建名为heat-user的用户，属于admin项目包，并赋予heat-user用户admin的权限，配置用户密码为123456，编写完成之后不要创建堆栈。

在controller节点将source admin-openrc && openstack-service status heat && openstack stack create -t /root/create\_network.yaml heat-network && cat /root/create\_network.yaml命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# vim create\_user.yaml

heat\_template\_version: 2014-10-16

resources:

user:

type: OS::Keystone::User

properties:

name: heat-user

password: "123456"

domain: demo

default\_project: admin

roles: \[{"role": admin, "project": admin}]

**答案**

\[root@controller \~]# source admin-openrc && openstack-service status heat && openstack stack create -t /root/create\_user.yaml test-user && cat /root/create\_user.yaml

MainPID=964 Id=openstack-heat-api-cfn.service ActiveState=active

MainPID=972 Id=openstack-heat-api.service ActiveState=active

MainPID=982 Id=openstack-heat-engine.service ActiveState=active

\+---------------------+--------------------------------------+

\| Field | Value |

\+---------------------+--------------------------------------+

\| id | ed1c7f33-ac2b-4c86-ac9c-d6b4d6a07c11 |

\| stack\_name | test-user |

\| description | No description |

\| creation\_time | 2024-06-24T08:37:24Z |

\| updated\_time | None |

\| stack\_status | CREATE\_IN\_PROGRESS |

\| stack\_status\_reason | Stack CREATE started |

\+---------------------+--------------------------------------+

heat\_template\_version: 2014-10-16

resources:

user:

type: OS::Keystone::User

properties:

name: heat-user

password: "123456"

domain: demo

default\_project: admin

roles: \[{"role": admin, "project": admin}]

### 任务3 OpenStack私有云运维开发(7分)

#### 安装python3环境(1分)

在controller节点安装python3环境。安装完之后查看python3版本，使用提供的whl文件安装依赖。

在controller节点将python3 --version && pip3 --version && pip3 list命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# yum install python3 -y

\[root@controller python-depend]# pip3 install certifi-2019.11.28-py2.py3-none-any.whl

\[root@controller python-depend]# pip3 install urllib3-1.25.11-py3-none-any.whl

\[root@controller python-depend]# pip3 install idna-2.8-py2.py3-none-any.whl

\[root@controller python-depend]# pip3 install chardet-3.0.4-py2.py3-none-any.whl

\[root@controller python-depend]# pip3 install requests-2.24.0-py2.py3-none-any.whl

**答案**

\[root@controller \~]# python3 --version && pip3 --version && pip3 list

Python 3.6.8

pip 9.0.3 from /usr/lib/python3.6/site-packages (python 3.6)

DEPRECATION: The default format will switch to columns in the future. You can use --format=(legacy|columns) (or define a format=(legacy|columns) in your pip.conf under the \[list] section) to disable this warning.

certifi (2019.11.28)

chardet (3.0.4)

idna (2.8)

pip (9.0.3)

requests (2.24.0)

setuptools (39.2.0)

urllib3 (1.25.11)

#### Python运维开发：基于OpenStack API实现镜像上传(3分)

编写python代码对接OpenStack API，完成镜像的上传。在controller节点的/root目录下创建create\_image.py文件，在该文件中编写python代码对接openstack api（需在py文件中获取token），要求在openstack私有云平台中上传镜像cirros-0.3.4-x86\_64-disk.img，命名为cirros\_python，disk\_format为qcow2，container\_format为bare。执行完代码要求输出“创建镜像成功，id为：xxxxxx”。

在controller节点将cat /root/create\_image.py && source /root/admin-openrc && openstack image list命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# python3 create\_image.py

请输入访问openstack平台控制节点IP地址：(xx.xx.xx.xx)

10.0.3.40

创建镜像成功，id为：57913340-fbce-4198-9210-697d44bf88bd

镜像文件上传成功

**答案**

\[root@controller \~]# cat /root/create\_image.py && source /root/admin-openrc && openstack image list

import requests,json,time

\# \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*全局变量IP\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\#执行代码前，请修改controller\_ip的IP地址，与指定router，IP可以input，也可以写成静态

controller\_ip = input("请输入访问openstack平台控制节点IP地址：(xx.xx.xx.xx)\n")

image\_name = "cirros\_python"

file\_path = "/root/cirros-0.3.4-x86\_64-disk.img"

try:

url = f"http://{controller\_ip}:5000/v3/auth/tokens"

body = {

"auth": {

"identity": {

"methods":\["password"],

"password": {

"user": {

"domain":{

"name": "demo"

},

"name": "admin",

"password": "000000"

}

}

},

"scope": {

"project": {

"domain": {

"name": "demo"

},

"name": "admin"

}

}

}

}

headers = {"Content-Type": "application/json"}

Token = requests.post(url, data=json.dumps(body), headers=headers).headers\['X-Subject-Token']

headers = {"X-Auth-Token": Token}

except Exception as e:

print(f"获取Token值失败，请检查访问云主机控制节点IP是否正确？输出错误信息如下：{str(e)}")

exit(0)

class glance\_api:

def \_\_init\_\_(self, headers: dict, resUrl: str):

self.headers = headers

self.resUrl = resUrl

\#创建glance镜像

def create\_glance(self, container\_format="bare", disk\_format="qcow2"):

body = {

"container\_format": container\_format,

"disk\_format": disk\_format,

"name": image\_name,

}

status\_code = requests.post(self.resUrl, data=json.dumps(body), headers=self.headers).status\_code

if(status\_code == 201):

return f"创建镜像成功，id为：{glance\_api.get\_glance\_id()}"

else:

return "创建镜像失败"

\#获取glance镜像id

def get\_glance\_id(self):

result = json.loads(requests.get(self.resUrl,headers=self.headers).text)

for item in result\['images']:

if(item\['name'] == image\_name):

return item\['id']

\#上传glance镜像

def update\_glance(self):

self.resUrl=self.resUrl+"/"+self.get\_glance\_id()+"/file"

self.headers\['Content-Type'] = "application/octet-stream"

status\_code = requests.put(self.resUrl,data=open(file\_path,'rb').read(),headers=self.headers).status\_code

if(status\_code == 204):

return "镜像文件上传成功"

else:

return "镜像文件上传失败"

glance\_api = glance\_api(headers,f"http://{controller\_ip}:9292/v2/images")

print(glance\_api.create\_glance()) #调用glance-api中创建镜像方法

print(glance\_api.update\_glance())

\+--------------------------------------+---------------+--------+

\| ID | Name | Status |

\+--------------------------------------+---------------+--------+

\| 17e77a03-6fb9-41df-835e-f20cc380ed77 | cirros | active |

\| 2b5bc39d-1ada-47b5-8ea4-6f010802c86f | cirros-swift | active |

\| 57913340-fbce-4198-9210-697d44bf88bd | cirros\_python | active |

\| 8e27da65-6f4e-4a62-85d7-c6b163d1ad98 | cirrossave | active |

\+--------------------------------------+---------------+--------+

#### Python运维开发：基于Openstack API创建用户(3分)

编写python代码对接OpenStack API，完成用户的创建。在controller节点的/root目录下创建create\_user.py文件，在该文件中编写python代码对接openstack api（需在py文件中获取token），要求在openstack私有云平台中创建用户gjbs。

在controller节点将cat /root/create\_user.py && source admin-openrc && openstack user list命令的返回结果提交到答题框。

**操作步骤**

\[root@controller \~]# python3 create\_user.py

请输入访问openstack平台控制节点IP地址：(xx.xx.xx.xx)

10.0.3.40

用户 gjbs 创建成功,ID为674a75c3dee642fbae395d58b8a14562

该平台的用户为：

gjbs

admin

demo

glance

nova

competition

placement

neutron

cinder

swift

heat

heat\_domain\_admin

**答案**

\[root@controller \~]# cat /root/create\_user.py && source admin-openrc && openstack user list

import requests,json,time

\# \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*全局变量IP\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\#执行代码前，请修改controller\_ip的IP地址，与指定router，IP可以input，也可以写成静态

controller\_ip = input("请输入访问openstack平台控制节点IP地址：(xx.xx.xx.xx)\n")

try:

url = f"http://{controller\_ip}:5000/v3/auth/tokens"

body = {

"auth": {

"identity": {

"methods":\["password"],

"password": {

"user": {

"domain":{

"name": "demo"

},

"name": "admin",

"password": "000000"

}

}

},

"scope": {

"project": {

"domain": {

"name": "demo"

},

"name": "admin"

}

}

}

}

headers = {"Content-Type": "application/json"}

Token = requests.post(url, data=json.dumps(body), headers=headers).headers\['X-Subject-Token']

headers = {"X-Auth-Token": Token}

except Exception as e:

print(f"获取Token值失败，请检查访问云主机控制节点IP是否正确？输出错误信息如下：{str(e)}")

exit(0)

class openstack\_user\_api:

def \_\_init\_\_(self, handers: dict, resUrl: str):

self.headers = handers

self.resUrl = resUrl

def create\_users(self, user\_name):

body = {

"user": {

"description": "API create user!",

"domain\_id": "default",

"name": user\_name

}

}

status\_code = requests.post(self.resUrl, data=json.dumps(body), headers=self.headers).text

result = json.loads(requests.get(self.resUrl, headers=self.headers).text)

user\_name = user\_name

for i in result\['users']:

if i\['name'] == user\_name:

return f"用户 {user\_name} 创建成功,ID为{i\['id']}"

def list\_users(self):

result = json.loads(requests.get(self.resUrl, headers=self.headers).text)

roles = \[]

for i in result\['users']:

if i\['name'] not in roles:

roles.append(i\['name'])

return "该平台的用户为：\n"+'\n'.join(roles)

def get\_user\_id(self, user\_name):

result = json.loads(requests.get(self.resUrl, headers=self.headers).text)

user\_name = user\_name

for i in result\['users']:

if i\['name'] == user\_name:

return (f"用户 {user\_name} 的ID为{i\['id']}")

def delete\_user(self, user\_name):

result = json.loads(requests.get(self.resUrl, headers=self.headers).text)

for i in result\['users']:

if i\['name'] == user\_name:

i = i\['id']

status\_code = requests.delete(f'http://{controller\_ip}:5000/v3/users/{i}', headers=self.headers)

return f"用户 {user\_name} 已删除！"

openstack\_user\_api = openstack\_user\_api(headers, f"http://{controller\_ip}:5000/v3/users")

print(openstack\_user\_api.create\_users("gjbs"))

print(openstack\_user\_api.list\_users())

\#print(openstack\_user\_api.delete\_user("gjbs"))

\+----------------------------------+-------------------+

\| ID | Name |

\+----------------------------------+-------------------+

\| fc2723632cb542bc8feebcad9e39b694 | gjbs |

\| b014a12c9f9e4c64bf5b05c6c559de04 | admin |

\| c3e56fba030549d0a2d678a84c9a672d | demo |

\| 8d51a119470a4c7899997baff50ffee9 | competition |

\| 232278a6da6549ed9171a35332af9e60 | glance |

\| f00bfb6493a4479b9165f4f11052e3da | nova |

\| 40249277d2a446649f7833abcb829cad | placement |

\| 809ea9c5fa714787b86dcf5469e31490 | neutron |

\| 2f77e75cf908412a9906579fd0aaea59 | cinder |

\| 2610a2821b1249978cf0d248c512cd0d | swift |

\| da7155a091b940e8876e4ddcfffd7c1f | manila |

\| 386ca446983940d5ba7cc369732b7fbc | heat |

\| 8716f4f490594d02bd242b888b5469bd | heat\_domain\_admin |

\| 5f3428cd4ebf4996ac7e6d5b3323e841 | cloudkitty |

\+----------------------------------+-------------------+

### B模块：容器云平台部署与运维（30分）

企业构建Kubernetes容器云集群，引入KubeVirt实现OpenStack到Kubernetes的全面转型，用Kubernetes来管一切虚拟化运行时，包含裸金属、VM、容器。同时研发团队决定搭建基于Kubernetes 的CI/CD环境，基于这个平台来实现DevOps流程。引入服务网格Istio，实现业务系统的灰度发布，治理和优化公司各种微服务，并开发自动化运维程序。

表2 IP地址规划

| 设备名称  | 主机名    | 接口   | IP地址                                             | 说明              |
| ----- | ------ | ---- | ------------------------------------------------ | --------------- |
| 云服务器1 | master | eth0 | <p>公网IP:********</p><p>私网IP:192.168.100.*/24</p> | Harbor也是使用该云服务器 |
| 云服务器2 | node   | eth0 | <p>公网IP:********</p><p>私网IP:192.168.100.*/24</p> |                 |

**说明：**

1.表1中的公网IP和私网IP以自己云主机显示为准，每个人的公网IP和私网IP不同。使用第三方软件远程连接云主机，使用公网IP连接。

2.华为云中云主机名字已命好，直接使用对应名字的云主机即可。

3.竞赛用到的软件包都在云主机/root下。

### 任务1 容器云服务搭建（6分）

#### 部署容器云平台（6分）

在master节点和node节点将root密码设为000000，完成Kubernetes集群的部署，并完成Istio服务网格、KubeVirt虚拟化和Harbor镜像仓库的部署（master节点依次执行k8s\_harbor\_install.sh、k8s\_image\_push.sh、k8s\_master\_install.sh、k8s\_project \_install.sh，node节点执行k8s\_node\_install.sh）。

请将kubectl cluster-info&\&kubectl -n istio-system get all&\&kubectl -n kubevirt get deployment命令的返回结果提交到答题框。

**操作步骤：**

Master节点

\[root@master \~]# hostnamectl set-hostname master

\[root@master \~]# vi /etc/hosts

127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4

::1 localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.200.10 master

192.168.200.20 node

\[root@master \~]# mkdir /opt/centos

\[root@master \~]# mount CentOS-7-x86\_64-DVD-2009.iso /opt/centos/

\[root@master \~]# mount kubernetes\_v1.6.iso /mnt/

\[root@master \~]# cp -rvf /mnt/\* /opt/

\[root@master \~]# rm -rf /etc/yum.repos.d/\*

\[root@master \~]# vi /etc/yum.repos.d/local.repo

\[local]

name=local

gpgcheck=0

enabled=1

baseurl=file:///opt/centos

\[k8s]

name=k8s

gpgcheck=0

enabled=1

baseurl=file:///opt/kubernetes-repo

\[root@master \~]# cd /opt/

\[root@master opt]# ./k8s\_harbor\_install.sh

\[root@master opt]# ./k8s\_image\_push.sh

输入镜像仓库地址(不加http/https): 192.168.200.10

输入镜像仓库用户名: admin

输入镜像仓库用户密码: Harbor12345

您设置的仓库地址为: 192.168.200.10,用户名: admin,密码: xxx

是否确认(Y/N): Y

\[root@master opt]# ./k8s\_master\_install.sh

\[root@master \~]# scp /opt/k8s\_node\_install.sh node:/root/

\[root@master \~]# yum install -y vsftpd

\[root@master \~]# echo "anon\_root=/opt" >> /etc/vsftpd/vsftpd.conf

\[root@master \~]# systemctl restart vsftpd

Node节点

\[root@node \~]# hostnamectl set-hostname node

\[root@node \~]## rm -rf /etc/yum.repos.d/\*

\[root@node \~]# vi /etc/hosts

127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4

::1 localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.200.10 master

192.168.200.20 node

\[root@node \~]# vi /etc/yum.repos.d/local.repo

\[centos]

name=centos

gpgcheck=0

enabled=1

baseurl=ftp://192.168.200.10/centos

\[k8s]

name=k8s

gpgcheck=0

enabled=1

baseurl=ftp://192.168.200.10/kubernetes-repo

\[root@node \~]# ./k8s\_node\_install.sh

\# 安装部署istio和kubevirt

\[root@master opt]# ./k8s\_project \_install.sh

**答案**

\[root@master \~]# kubectl cluster-info&\&kubectl -n istio-system get all&\&kubectl -n kubevirt get deployment

Kubernetes control plane is running at https://192.168.100.105:6443

CoreDNS is running at https://192.168.100.105:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

Warning: kubevirt.io/v1 VirtualMachineInstancePresets is now deprecated and will be removed in v2.

NAME READY STATUS RESTARTS AGE

pod/grafana-56bdf8bf85-cjtnw 1/1 Running 0 2m46s

pod/istio-egressgateway-85649899f8-svrkj 1/1 Running 0 2m52s

pod/istio-ingressgateway-f56888458-gpk6h 1/1 Running 0 2m52s

pod/istiod-64848b6c78-ng8vf 1/1 Running 0 2m56s

pod/jaeger-76cd7c7566-hflpz 1/1 Running 0 2m46s

pod/kiali-646db7568f-5k8vs 1/1 Running 0 2m46s

pod/prometheus-85949fddb-z7c7v 2/2 Running 0 2m46s

NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE

service/grafana ClusterIP 10.108.78.249 \<none> 3000/TCP 2m46s

service/istio-egressgateway ClusterIP 10.98.41.83 \<none> 80/TCP,443/TCP 2m52s

service/istio-ingressgateway LoadBalancer 10.96.228.3 \<pending> 15021:31902/TCP,80:30125/TCP,443:32162/TCP,31400:31827/TCP,15443:31436/TCP 2m52s

service/istiod ClusterIP 10.100.163.50 \<none> 15010/TCP,15012/TCP,443/TCP,15014/TCP 2m56s

service/jaeger-collector ClusterIP 10.107.163.161 \<none> 14268/TCP,14250/TCP,9411/TCP 2m46s

service/kiali ClusterIP 10.106.85.34 \<none> 20001/TCP,9090/TCP 2m46s

service/prometheus ClusterIP 10.108.246.214 \<none> 9090/TCP 2m46s

service/tracing ClusterIP 10.98.9.225 \<none> 80/TCP,16685/TCP 2m46s

service/zipkin ClusterIP 10.110.212.28 \<none> 9411/TCP 2m46s

NAME READY UP-TO-DATE AVAILABLE AGE

deployment.apps/grafana 1/1 1 1 2m46s

deployment.apps/istio-egressgateway 1/1 1 1 2m52s

deployment.apps/istio-ingressgateway 1/1 1 1 2m52s

deployment.apps/istiod 1/1 1 1 2m56s

deployment.apps/jaeger 1/1 1 1 2m46s

deployment.apps/kiali 1/1 1 1 2m46s

deployment.apps/prometheus 1/1 1 1 2m46s

NAME DESIRED CURRENT READY AGE

replicaset.apps/grafana-56bdf8bf85 1 1 1 2m46s

replicaset.apps/istio-egressgateway-85649899f8 1 1 1 2m52s

replicaset.apps/istio-ingressgateway-f56888458 1 1 1 2m52s

replicaset.apps/istiod-64848b6c78 1 1 1 2m56s

replicaset.apps/jaeger-76cd7c7566 1 1 1 2m46s

replicaset.apps/kiali-646db7568f 1 1 1 2m46s

replicaset.apps/prometheus-85949fddb 1 1 1 2m46s

NAME READY UP-TO-DATE AVAILABLE AGE

virt-api 2/2 2 2 94s

virt-controller 2/2 2 2 69s

virt-operator 2/2 2 2 2m

### 任务2 容器云服务运维（18分）

#### 容器化部署Node-Exporter（1分）

编写Dockerfile文件构建exporter镜像，要求基于centos完成Node-Exporter服务的安装与配置，并设置服务开机自启。（需要的包在Technology\_packageV1.0.iso中Monitor.tar.gz）

(1)基础镜像：centos:centos7.9.2009；

(2)使用二进制包node\_exporter-0.18.1.linux-amd64.tar.gz安装node-exporter服务；

(3)声明端口：9100；

(4)设置服务开机自启。

请使用docker build命令进行构建镜像monitor-exporter:v1.0并使用 docker run 命令运行该容器。

将docker run -d --name exporter-test monitor-exporter:v1.0 && sleep 5 && docker exec exporter-test ps -aux && docker rm -f exporter-test 命令的返回结果提交到答题框。

**答案**

\[root@master Monitor]# docker run -d --name exporter-test monitor-exporter:v1.0 && sleep 5 && docker exec exporter-test ps -aux && docker rm -f exporter-test

b1bc70566251b1b9d6feb8b8b088ac149ea38e855ebebfd82c164659b952effd

USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND

root 1 0.4 0.0 113932 5176 ? Ssl 14:33 0:00 node\_exporter

root 11 0.0 0.0 51732 1700 ? Rs 14:33 0:00 ps -aux

exporter-test

**操作步骤**

\[root@master Monitor]# cat Dockerfile-exporter

FROM centos:centos7.9.2009

ADD node\_exporter-0.18.1.linux-amd64.tar.gz /root/

RUN mv /root/node\_exporter-0.18.1.linux-amd64/node\_exporter /usr/local/bin

EXPOSE 9100

ENTRYPOINT \["node\_exporter"]

\[root@master Monitor]# cd /root/Monitor && docker build -t monitor-exporter:v1.0 -f Dockerfile-exporter .

Sending build context to Docker daemon 741MB

Step 1/5 : FROM centos:centos7.9.2009

\---> eeb6ee3f44bd

Step 2/5 : ADD node\_exporter-0.18.1.linux-amd64.tar.gz /root/

\---> Using cache

\---> d15d0198c74c

Step 3/5 : RUN mv /root/node\_exporter-0.18.1.linux-amd64/node\_exporter /usr/local/bin

\---> Using cache

\---> 1341329663e0

Step 4/5 : EXPOSE 9100

\---> Using cache

\---> 3f07edb7a5bc

Step 5/5 : ENTRYPOINT \["node\_exporter"]

\---> Using cache

\---> 444b36432c64

Successfully built 444b36432c64

Successfully tagged monitor-exporter:v1.0

#### 容器化部署Alertmanager（1分）

编写Dockerfile文件构建alert镜像，要求基于centos完成Alertmanager服务的安装与配置，并设置服务开机自启。（需要的包在Technology\_packageV1.0.iso中Monitor.tar.gz）

(1)基础镜像：centos:centos7.9.2009

(2)使用提供的二进制包alertmanager-0.19.0.linux-amd64.tar.gz安装Alertmanager服务；

(3)声明端口：9093、9094；

(4)设置服务开机自启。

请使用docker build命令进行构建镜像monitor-alert:v1.0并使用 docker run 命令运行该容器。

将docker run -d --name alert-test monitor-alert:v1.0 && sleep 5 && docker exec alert-test ps -aux && docker rm -f alert-test命令的返回结果提交到答题框。

**答案**

\[root@master Monitor]# docker run -d --name alert-test monitor-alert:v1.0 && sleep 5 && docker exec alert-test ps -aux && docker rm -f alert-test

4c47c6e364ae32f9132e331d0ff5f792f20172aa562358682e5baae3005ff8fa

USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND

root 1 1.8 0.1 123664 16172 ? Ssl 14:34 0:00 ./alertmanager

root 19 0.0 0.0 51732 1700 ? Rs 14:34 0:00 ps -aux

alert-test

**操作步骤**

\[root@master Monitor]# cat Dockerfile-alert

FROM centos:centos7.9.2009

ADD alertmanager-0.19.0.linux-amd64.tar.gz /usr/local/bin

WORKDIR /usr/local/bin/alertmanager-0.19.0.linux-amd64

EXPOSE 9093 9094

ENTRYPOINT \["./alertmanager"]

\[root@master Monitor]# cd /root/Monitor && docker build -t monitor-alert:v1.0 -f Dockerfile-alert .

Sending build context to Docker daemon 360.5MB

Step 1/5 : FROM centos:centos7.9.2009

\---> eeb6ee3f44bd

Step 2/5 : ADD alertmanager-0.19.0.linux-amd64.tar.gz /usr/local/bin

\---> 963a3838e418

Step 3/5 : WORKDIR /usr/local/bin/alertmanager-0.19.0.linux-amd64

\---> Running in ff84e607dbc1

Removing intermediate container ff84e607dbc1

\---> c5f37060220a

Step 4/5 : EXPOSE 9093 9094

\---> Running in 19b6aca86b79

Removing intermediate container 19b6aca86b79

\---> 57eec22f50d9

Step 5/5 : ENTRYPOINT \["./alertmanager"]

\---> Running in ca8d983ded0a

Removing intermediate container ca8d983ded0a

\---> b036d073b0d3

Successfully built b036d073b0d3

Successfully tagged monitor-alert:v1.0

\[root@master Monitor]# docker run -d --name alert-test monitor-alert:v1.0 && sleep 5 && docker exec alert-test ps -aux && docker rm -f alert-test

6cec1f0cb4618fc934169c033c439e5826e7575c316c0932b8b2faea9f30e342

USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND

root 1 2.8 0.1 123664 16424 ? Ssl 15:44 0:00 ./alertmanager

root 20 0.0 0.0 51732 1700 ? Rs 15:44 0:00 ps -aux

alert-test

#### 容器化部署Grafana（1分）

编写Dockerfile文件构建grafana镜像，要求基于centos完成Grafana服务的安装与配置，并设置服务开机自启。（需要的包在Technology\_packageV1.0.iso中Monitor.tar.gz）

(1)基础镜像：centos:centos7.9.2009；

(2)使用提供的二进制包grafana-6.4.1.linux-amd64.tar.gz安装grafana服务；

(3)声明端口：3000；

(4)设置nacos服务开机自启。

请使用docker build命令进行构建镜像monitor-grafana:v1.0并使用 docker run 命令运行该容器。

将docker run -d --name grafana-test monitor-grafana:v1.0 && sleep 5 && docker exec grafana-test ps -aux && docker rm -f grafana-test 命令的返回结果提交到答题框。

**答案**

\[root@master Monitor]# docker run -d --name grafana-test monitor-grafana:v1.0 && sleep 5 && docker exec grafana-test ps -aux && docker rm -f grafana-test

73288553358ae0d3f62466ff78576637e499c06f3e68faf0035cce0491921e10

USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND

root 1 5.2 0.1 644488 28876 ? Ssl 14:35 0:00 ./grafana-server

root 18 0.0 0.0 51732 1696 ? Rs 14:35 0:00 ps -aux

grafana-test

**操作步骤**

\[root@master Monitor]# cat Dockerfile-grafana

FROM centos:centos7.9.2009

ADD grafana-6.4.1.linux-amd64.tar.gz /usr/local/bin

EXPOSE 3000

WORKDIR /usr/local/bin/grafana-6.4.1/bin

ENTRYPOINT \["./grafana-server"]

\[root@master Monitor]# cd /root/Monitor/ && docker build -t monitor-grafana:v1.0 -f Dockerfile-grafana .

Sending build context to Docker daemon 741MB

Step 1/5 : FROM centos:centos7.9.2009

\---> eeb6ee3f44bd

Step 2/5 : ADD grafana-6.4.1.linux-amd64.tar.gz /usr/local/bin

\---> Using cache

\---> 13fa73252116

Step 3/5 : EXPOSE 3000

\---> Using cache

\---> ee9b8fab46be

Step 4/5 : WORKDIR /usr/local/bin/grafana-6.4.1/bin

\---> Using cache

\---> d7a55b3e7067

Step 5/5 : ENTRYPOINT \["./grafana-server"]

\---> Using cache

\---> cb4ad90c5e57

Successfully built cb4ad90c5e57

Successfully tagged monitor-grafana:v1.0

\[root@master Monitor]# docker run -d --name grafana-test monitor-grafana:v1.0 && sleep 5 && docker exec grafana-test ps -aux && docker rm -f grafana-test

802f08eddc73903d003c0b83a3c7134c2c78ae75e1892cfcc31dc2248d33dd62

USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND

root 1 7.6 0.1 423368 17472 ? Ssl 15:54 0:00 ./grafana-server

root 17 0.0 0.0 51732 1696 ? Rs 15:54 0:00 ps -aux

grafana-test

#### 容器化部署Prometheus（2分）

编写Dockerfile文件构建prometheus镜像，要求基于centos完成Promethues服务的安装与配置，并设置服务开机自启。（需要的包在Technology\_packageV1.0.iso中Monitor.tar.gz）

（1）基础镜像：centos:centos7.9.2009；

（2）使用提供的二进制包prometheus-2.13.0.linux-amd64.tar.gz安装promethues服务；

（3）编写prometheus.yml文件，创建3个任务模板：prometheus、node和alertmanager，并将该文件拷贝到/data/prometheus/目录下；

（4）声明端口：9090；

（5）设置服务开机自启。

请使用docker build命令进行构建镜像monitor-prometheus:v1.0并使用 docker run 命令运行该容器。

将docker run -d --name prometheus-test monitor-prometheus:v1.0 && sleep 5 && docker exec prometheus-test ps -aux && docker rm -f prometheus-test命令的返回结果提交到答题框。

**答案**

\[root@master Monitor]# docker run -d --name prometheus-test monitor-prometheus:v1.0 && sleep 5 && docker exec prometheus-test ps -aux && docker rm -f prometheus-test

4deb3de5abf883531ab77522e61e89af8fb996768e4b5e3d9d17084b0ffedea8

USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND

root 1 1.6 0.1 161028 27332 ? Ssl 14:41 0:00 ./prometheus --config.file=/data/prometheus/prometheus.yml

root 19 0.0 0.0 51732 1696 ? Rs 14:41 0:00 ps -aux

prometheus-test

**操作步骤**

\[root@master Monitor]# cat Dockerfile-prometheus

FROM centos:centos7.9.2009

ADD prometheus-2.13.0.linux-amd64.tar.gz /usr/local/bin

WORKDIR /usr/local/bin/prometheus-2.13.0.linux-amd64

RUN mkdir -p /data/prometheus/

COPY prometheus.yml /usr/local/bin/prometheus-2.13.0.linux-amd64

COPY prometheus.yml /data/prometheus/

EXPOSE 9090

CMD \["./prometheus","--config.file=/data/prometheus/prometheus.yml"]

\[root@master Monitor]# cat prometheus.yml

\# prometheus.yml配置文件示例

\# my global config

global:

scrape\_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.

evaluation\_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

\# scrape\_timeout is set to the global default (10s).

\# Alertmanager configuration

alerting:

alertmanagers:

\- static\_configs:

\- targets:

\# - alertmanager:9093

\# Load rules once and periodically evaluate them according to the global 'evaluation\_interval'.

rule\_files:

\# - "first\_rules.yml"

\# - "second\_rules.yml"

\# A scrape configuration containing exactly one endpoint to scrape:

\# Here it's Prometheus itself.

scrape\_configs:

\- job\_name: 'prometheus'

static\_configs:

\- targets: \['172.16.20.228:9090']

\- job\_name: 'node-exporter'

static\_configs:

\- targets: \['172.16.20.228:9100']

\- job\_name: 'alertmanager'

static\_configs:

\- targets: \['172.16.20.228:9093']

\[root@master Monitor]# cd /root/Monitor/ && docker build -t monitor-prometheus:v1.0 -f Dockerfile-prometheus .

Sending build context to Docker daemon 741MB

Step 1/8 : FROM centos:centos7.9.2009

\---> eeb6ee3f44bd

Step 2/8 : ADD prometheus-2.13.0.linux-amd64.tar.gz /usr/local/bin

\---> Using cache

\---> 8454034faa17

Step 3/8 : WORKDIR /usr/local/bin/prometheus-2.13.0.linux-amd64

\---> Using cache

\---> 945562bd51c1

Step 4/8 : RUN mkdir -p /data/prometheus/

\---> Using cache

\---> b029764fd391

Step 5/8 : COPY prometheus.yml /usr/local/bin/prometheus-2.13.0.linux-amd64

\---> Using cache

\---> fbe3acb68679

Step 6/8 : COPY prometheus.yml /data/prometheus/

\---> Using cache

\---> fb9bc2298ed3

Step 7/8 : EXPOSE 9090

\---> Using cache

\---> 145073db7c6a

Step 8/8 : CMD \["./prometheus","--config.file=/data/prometheus/prometheus.yml"]

\---> Using cache

\---> 7977109c2467

Successfully built 7977109c2467

Successfully tagged monitor-prometheus:v1.0

\[root@master Monitor]# docker run -d --name prometheus-test monitor-prometheus:v1.0 && sleep 5 && docker exec prometheus-test ps -aux && docker rm -f prometheus-test

440ededb7d23cb777e63888992225d4fd0ae28bb90da7509289f0beae077a42b

USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND

root 1 4.0 0.2 161028 27408 ? Ssl 15:57 0:00 ./prometheus --config.file=/data/prometheus/prometheus.yml

root 19 3.0 0.0 51732 1700 ? Rs 15:57 0:00 ps -aux

prometheus-test

#### 编排部署监控系统（2分）

编写docker-compose.yaml文件，使用镜像exporter、alert、grafana和prometheus完成监控系统的编排部署。（需要的包在Technology\_packageV1.0.iso中Monitor.tar.gz）

(1)容器monitor-node；镜像：monitor-exporter:v1.0；端口映射：9100:9100；

(2)容器monitor- alertmanager；镜像：monitor-alert:v1.0；端口映射：9093:9093、9094:9094；

(3)容器monitor-grafana；镜像：monitor-grafana:v1.0；端口映射：3000:3000；

(4)容器monitor-prometheus；镜像：monitor-prometheus:v1.0；端口映射：9090:9090。

1.使用docker-compose ps命令进行查看，将返回结果提交至答题框。

2.将curl -L http://$(hostname -i):9090/targets | grep up 命令的返回结果提交到答题框。

**答案**

\[root@master Monitor]# docker-compose ps

NAME COMMAND SERVICE STATUS PORTS

monitor-alertmanager "./alertmanager" monitor-alertmanager running 0.0.0.0:9093->9093/tcp, :::9093->9093/tcp

monitor-grafana "./grafana-server" monitor-grafana running 0.0.0.0:3000->3000/tcp, :::3000->3000/tcp

monitor-node "node\_exporter" monitor-node running 0.0.0.0:9100->9100/tcp, :::9100->9100/tcp

monitor-prometheus "./prometheus --conf…" monitor-prometheus running 0.0.0.0:9090->9090/tcp, :::9090->9090/tcp

\[root@master Monitor]# curl -L http://$(hostname -i):9090/targets | grep up

% Total % Received % Xferd Average Speed Time Time Time Current

Dload Upload Total Spent Left Speed

100 9112 0 9112 0 0 2596k 0 --:--:-- --:--:-- --:--:-- 2966k

\<a href="#" class="nav-link dropdown-toggle" data-toggle="dropdown" role="button" aria-haspopup="true" aria-expanded="false">Status \<span class="caret">\</span>\</a>

\<div id="showTargets" class="btn-group btn-group-toggle" data-toggle="buttons">

\<a id="job-alertmanager" href="#job-alertmanager">alertmanager (1/1 up)\</a>

\<span class="alert alert-success state\_indicator text-uppercase">up\</span>

\<a id="job-node" href="#job-node">node (1/1 up)\</a>

\<span class="alert alert-success state\_indicator text-uppercase">up\</span>

\<a id="job-prometheus" href="#job-prometheus">prometheus (1/1 up)\</a>

\<span class="alert alert-success state\_indicator text-uppercase">up\</span>

**操作步骤**

\[root@master Monitor]# cat docker-compose.yaml

version: '3'

services:

monitor-node:

image: monitor-exporter:v1.0

container\_name: monitor-node

ports:

\- "9100:9100"

monitor-alertmanager:

image: monitor-alert:v1.0

container\_name: monitor-alertmanager

ports:

\- "9093:9093"

monitor-grafana:

image: monitor-grafana:v1.0

container\_name: monitor-grafana

ports:

\- "3000:3000"

depends\_on:

\- monitor-prometheus

monitor-prometheus:

image: monitor-prometheus:v1.0

container\_name: monitor-prometheus

ports:

\- "9090:9090"

\[root@master Monitor]# docker-compose up -d

\[+] Running 4/4

⠿ Container monitor-prometheus Started 1.7s

⠿ Container monitor-grafana Started 3.6s

⠿ Container monitor-node Started 1.9s

⠿ Container monitor-alertmanager Started

\[root@master Monitor]# docker-compose -f /root/Monitor/docker-compose.yaml ps

NAME COMMAND SERVICE STATUS PORTS

monitor-alertmanager "./alertmanager" monitor-alertmanager running 0.0.0.0:9093->9093/tcp, :::9093->9093/tcp

monitor-grafana "./grafana-server" monitor-grafana running 0.0.0.0:3000->3000/tcp, :::3000->3000/tcp

monitor-node "node\_exporter" monitor-node running 0.0.0.0:9100->9100/tcp, :::9100->9100/tcp

monitor-prometheus "./prometheus --conf…" monitor-prometheus running 0.0.0.0:9090->9090/tcp, :::9090->9090/tcp

\[root@master Monitor]# curl -L http://$(hostname -i):9090/targets | grep up

% Total % Received % Xferd Average Speed Time Time Time Current

Dload Upload Total Spent Left Speed

100 9112 0 9112 0 0 2596k 0 --:--:-- --:--:-- --:--:-- 2966k

\<a href="#" class="nav-link dropdown-toggle" data-toggle="dropdown" role="button" aria-haspopup="true" aria-expanded="false">Status \<span class="caret">\</span>\</a>

\<div id="showTargets" class="btn-group btn-group-toggle" data-toggle="buttons">

\<a id="job-alertmanager" href="#job-alertmanager">alertmanager (1/1 up)\</a>

\<span class="alert alert-success state\_indicator text-uppercase">up\</span>

\<a id="job-node" href="#job-node">node (1/1 up)\</a>

\<span class="alert alert-success state\_indicator text-uppercase">up\</span>

\<a id="job-prometheus" href="#job-prometheus">prometheus (1/1 up)\</a>

\<span class="alert alert-success state\_indicator text-uppercase">up\</span>

1. **导入jenkins镜像（1分）**

基于Kubernetes构建持续集成,master节点、harbor节点和cicd节点对应的IP都为master节点的IP, CICD\_OFF.TAR（需要的包在Technology\_packageV1.0.iso中CICD\_CICD\_Offline.TAR）。把CICD\_CICD\_Offline.TAR移动到/opt目录下然后解压。导入jenkins.tar文件中的镜像。

将docker images | grep jenkins命令的返回结果提交到答题框。

**答案**

\[root@master Monitor]# docker images | grep jenkins

jenkins/jenkins 2.262-centos f04839b3e211 2 years ago 638MB

**操作步骤**

\[root@master opt]#cd /opt

\[root@master opt]# mv /root/CICD\_Offline.TAR /opt/

\[root@master opt]#tar -zxvf CICD\_Offline.TAR

\[root@master opt]# docker load -i jenkins.tar

\[root@master opt]# docker images | grep jenkins

jenkins/jenkins 2.262-centos f04839b3e211 2 years ago 638MB

1. **安装Jenkins（1分）**

编写Jenkins编排文件，启动并配置Jenkins。

(1)编写docker-compose.yaml启动Jenkins。

(2)新建用户springcloud，密码000000

(3)修改系统配置Resource root URL。

将docker-compose ps命令的返回结果提交到答题框。

**答案**

\[root@master jenkins]# docker-compose ps

NAME COMMAND SERVICE STATUS PORTS

jenkins "/sbin/tini -- /usr/…" jenkins running 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 50000/tcp

**操作步骤**

\[root@master \~]# mkdir jenkins

\[root@master \~]# cd jenkins

\[root@master jenkins]# vi docker-compose.yaml

version: '3.1'

services:

jenkins:

image: 'jenkins/jenkins:2.262-centos'

volumes:

\- /home/jenkins\_home:/var/jenkins\_home

\- /var/run/docker.sock:/var/run/docker.sock

\- /usr/bin/docker:/usr/bin/docker

\- /usr/bin/kubectl:/usr/local/bin/kubectl

\- /root/.kube:/root/.kube

ports:

\- "8080:8080"

expose:

\- "8080"

\- "50000"

privileged: true

user: root

restart: always

container\_name: jenkins

启动Jenkins：

\[root@master jenkins]# docker-compose -f docker-compose.yaml up -d

Creating network "root\_default" with the default driver

Creating jenkins ... done

\[root@master jenkins]# docker-compose up -d

Creating network "jenkins\_default" with the default driver

Creating jenkins ... done

\[root@master jenkins]# docker-compose ps

Name Command State Ports

\----------------------------------------------------------------------------------------------------------

jenkins /sbin/tini -- /usr/local/b ... Up 50000/tcp, 0.0.0.0:8080->8080/tcp

安装插件：

\[root@master jenkins]# cp -rfv /opt/plugins/\* /home/jenkins\_home/plugins/

\[root@master jenkins]# docker restart jenkins

Jenkins

在web端通过http://IP:8080访问Jenkins，如图所示：

![](<../../.gitbook/assets/0 (2).png>)

查看密码：

\[root@master \~]# docker exec jenkins cat /var/jenkins\_home/secrets/initialAdminPassword

54670064056e42a2a47485a5e3356e57

输入密码并点击“继续”，如图所示：

![](<../../.gitbook/assets/1 (2).png>)

依次点击“系统管理”、“管理用户”按钮进入用户列表界面，如图所示：

![](<../../.gitbook/assets/2 (1).png>)

点击“新建用户”，输入用户信息，如图所示：

![](<../../.gitbook/assets/3 (1).png>)

点击“创建用户”如图所示：

![](<../../.gitbook/assets/4 (1).png>)

退出admin用户登录，使用新创建的用户登录Jenkins。

依次点击 “系统管理”🡪“系统配置”按钮进入系统配置界面，在“Resource root URL”处配置Jenkins URL，如图所示：

![](<../../.gitbook/assets/5 (1).png>)

完成后点击“保存”。

1. **部署Gitlab（2分）**

编写Gitlab编排文件并启动Gitlab。

(1)编写docker-compose.yaml启动Gitlab。

(2)使用root用户登录Gitlab。

(3)在harbor仓库创建公开项目springcloud。

将docker-compose ps命令的返回结果提交到答题框。

**答案**

\[root@master gitlab]# docker-compose ps

NAME COMMAND SERVICE STATUS PORTS

gitlab "/assets/wrapper" gitlab running (healthy) 0.0.0.0:443->443/tcp, :::443->443/tcp, 0.0.0.0:1022->22/tcp, :::1022->22/tcp, 0.0.0.0:81->80/tcp, :::81->80/tcp

**操作步骤**

\[root@master \~]# mkdir gitlab

\[root@master \~]# cd gitlab/

\[root@master gitlab]# vi docker-compose.yaml

version: '3'

services:

gitlab:

image: 'gitlab/gitlab-ce:12.9.2-ce.0'

container\_name: gitlab

restart: always

hostname: '192.168.100.10'

privileged: true

environment:

TZ: 'Asia/Shanghai'

ports:

\- '81:80'

\- '443:443'

\- '1022:22'

volumes:

\- /srv/gitlab/config:/etc/gitlab

\- /srv/gitlab/gitlab/logs:/var/log/gitlab

\- /srv/gitlab/gitlab/data:/var/opt/gitlab

启动Gitlab：

\[root@master gitlab]# docker-compose up -d

\[+] Running 2/2

⠿ Network gitlab\_default Created 0.0s

⠿ Container gitlab Started 0.3s

\[root@master gitlab]# docker-compose ps

NAME COMMAND SERVICE STATUS PORTS

gitlab "/assets/wrapper" gitlab running (healthy) 0.0.0.0:443->443/tcp, :::443->443/tcp, 0.0.0.0:1022->22/tcp, :::1022->22/tcp, 0.0.0.0:81->80/tcp, :::81->80/tcp

Gitlab启动较慢，启动完成后，在web端访问Gitlab（http://IP:81），如图所示：

![](../../.gitbook/assets/6.png)

设置root用户信息并使用root用户登录Gitlab，如图所示：

![](../../.gitbook/assets/7.png)（2）创建项目

点击“Create a project”，创建项目springcloud，可见等级选择“Public”，如图所示：

![](../../.gitbook/assets/8.png)

点击“创建项目”，进入项目，如图所示：

![](../../.gitbook/assets/9.png)

1. **push源代码（2分）**

push源代码到gitlab的springcloud项目，并完成相关配置。

将git push -u origin master命令成功push的返回结果提交到答题框。

**答案**

\[root@master springcloud]# git push -u origin master

Username for 'http://192.168.200.115:81': root

Password for 'http://root@192.168.200.115:81':

Counting objects: 3192, done.

Delta compression using up to 4 threads.

Compressing objects: 100% (1428/1428), done.

Writing objects: 100% (3192/3192), 1.40 MiB | 1.59 MiB/s, done.

Total 3192 (delta 1233), reused 3010 (delta 1207)

remote: Resolving deltas: 100% (1233/1233), done.

To http://192.168.200.115:81/root/springcloud.git

\* \[new branch] master -> master

Branch master set up to track remote branch master from origin.

**操作步骤**

\[root@master gitlab]# yum install -y git

\[root@master gitlab]# cd /opt/springcloud/

\[root@master springcloud]# git config --global user.name "administrator"

\[root@master springcloud]# git config --global user.email "admin@example.com"

\[root@master springcloud]# git remote remove origin

\[root@master springcloud]# git remote add origin http://192.168.100.10:81/root/springcloud.git

\[root@master springcloud]# git add .

\[root@master springcloud]# git commit -m "initial commit"

\[root@master springcloud]# git push -u origin master

Username for 'http://192.168.100.10:81': root

Password for 'http://root@192.168.100.10:81':

Counting objects: 3192, done.

Delta compression using up to 4 threads.

Compressing objects: 100% (1428/1428), done.

Writing objects: 100% (3192/3192), 1.40 MiB | 0 bytes/s, done.

Total 3192 (delta 1233), reused 3010 (delta 1207)

remote: Resolving deltas: 100% (1233/1233), done.

To http://192.168.100.10:81/root/springcloud.git

\* （new branch） master -> master

Branch master set up to track remote branch master from origin.

刷新网页，springcloud项目中文件已经更新了，如图所示：

![](../../.gitbook/assets/10.png)

1. **Jenkins连接maven （1分）**

配置Jenkins连接Gitlab，安装maven并完成相关配置。

将docker exec jenkins bash -c "source /etc/profile && mvn -v"命令的返回结果提交到答题框。

**答案**

Apache Maven 3.6.3 (cecedd343002696d0abb50b32b541b8a6ba2883f)

Maven home: /usr/local/maven

Java version: 1.8.0\_265, vendor: Oracle Corporation, runtime: /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.265.b01-0.el8\_2.x86\_64/jre

Default locale: en\_US, platform encoding: ANSI\_X3.4-1968

OS name: "linux", version: "3.10.0-862.2.3.el7.x86\_64", arch: "amd64", family: "unix"

**操作步骤**

3.配置Jenkins连接Gitlab

（1）设置Outbound requests

登录Gitlab首页，如图所示：

![](../../.gitbook/assets/11.png)

点击管理区域的扳手图标，如图所示：

![](../../.gitbook/assets/12.png)

点击左侧导航栏的“Settings”→“Network”，设置“Outbound requests”，勾选“Allow requests to the local network from web hooks and services”，如图所示：

![](../../.gitbook/assets/13.png)

配置完成后保存。

（2）创建Gitlab API Token

点击Gitlab用户头像图标，如图所示：

![](../../.gitbook/assets/14.png)

点击“Settings”，如图所示：

![](../../.gitbook/assets/15.png)

点击左侧导航栏的“Access Tokens”添加token，如图所示：

![](../../.gitbook/assets/16.png)

点击“Create personal access token”生成Token，如图所示：

6Tftkc2C\_5P8pS74ZDuF

![](../../.gitbook/assets/17.png)

复制Token（fNJF37GcttyG18v83tcy），后面配置Jenkins时会用到。

（3）设置Jenkins

登录Jenkins首页，点击“系统管理”→“系统配置”，配置Gitlab信息，取消勾选“Enable authentication for '/project' end-point”，如图所示：

![](../../.gitbook/assets/18.png)

点击“添加”→“Jenkins”添加认证信息，将Gitlab API Token填入，如图所示：

![](../../.gitbook/assets/19.png)

点击“Test Connection”，如图所示：

![](../../.gitbook/assets/20.png)

\[root@master \~]# cp -rf /opt/apache-maven-3.6.3-bin.tar.gz /home/jenkins\_home/

\[root@master \~]# docker exec -it jenkins bash

\[root@1a3190c3a8b2 /]# tar -zxvf /var/jenkins\_home/apache-maven-3.6.3-bin.tar.gz -C .

\[root@1a3190c3a8b2 /]# mv apache-maven-3.6.3/ /usr/local/maven

\[root@1a3190c3a8b2 /]# vi /etc/profile

export M2\_HOME=/usr/local/maven # 行末添加两行

export PATH=$PATH:$M2\_HOME/bin

\[root@1a3190c3a8b2 /]# vi /root/.bashrc

\# .bashrc

\# User specific aliases and functions

alias rm='rm -i'

alias cp='cp -i'

alias mv='mv -i'

\# Source global definitions

if （ -f /etc/bashrc ）; then

. /etc/bashrc

source /etc/profile # 添加本行

fi

退出容器重新进入：

\[root@1a3190c3a8b2 /]# mvn -v

Apache Maven 3.6.3 (cecedd343002696d0abb50b32b541b8a6ba2883f)

Maven home: /usr/local/maven

Java version: 1.8.0\_265, vendor: Oracle Corporation, runtime: /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.265.b01-0.el8\_2.x86\_64/jre

Default locale: en\_US, platform encoding: ANSI\_X3.4-1968

OS name: "linux", version: "3.10.0-1160.el7.x86\_64", arch: "amd64", family: "unix"

登录Jenkins首页，点击“系统管理”→“全局工具配置”，如图所示：

点击“新增Maven”，如图所示。取消勾选“自动安装”，填入maven名称和安装路径，配置完成后点击“应用”。

1. **配置并触发CI/CD（3分）**

编写流水线脚本配置CI/CD，habor仓库创建springcloud项目，上传代码触发自动构建。

构建成功后将curl \`kubectl get endpoints -n springcloud gateway |grep -v AGE| awk '{print $2}'\` 命令的返回结果提交到答题框。

**答案**

\[root@master \~]# curl \`kubectl get endpoints -n springcloud gateway |grep -v AGE| awk '{print $2}'\` | grep Pig

% Total % Received % Xferd Average Speed Time Time Time Current

Dload Upload Total Spent Left Speed

0 0 0 0 0 0 0 0 --:--:-- --:--:-- --:--:-- 0 \<title>Piggy Metrics\</title>

100 32164 100 \<span class="frominfo" id="infosubtitle">Piggy Metrics is the new simple way to deal with personal finances\</span>

32164 0 0 1847k 0 --:--:-- --:--:-- --:--:-- 1963k

\<a id="infofooter" href="https://github.com/sqshq/PiggyMetrics">\&copy; 2016 sqshq.com\</a>

**操作步骤**

配置CI/CD

（1）新建任务

登录Jenkins首页，点击左侧导航栏“新建任务”，如图所示，选择构建一个流水线。

点击“确定”，配置构建触发器，如图所示：

记录下GitLab webhook URL的地址（http://192.168.100.10:8080/project/springcloud），后期配置webhook需要使用。

配置流水线，如图所示：

点击“流水线语法”，如图所示，示例步骤选择“git：Git”，将springcloud项目地址填入仓库URL。

点击“添加”→“jenkins”添加凭据，如图所示。类型选择“Username with password”，用户名和密码为Gitlab仓库的用户名和密码。

添加凭据后选择凭据，如图所示：

点击“生成流水线脚本”，如图所示：

git credentialsId: '7a2472ee-2a20-494f-a6bc-42a5630afc9e', url: 'http://192.168.100.10:81/juzihh12/springcloud.git'

记录生成的值，并将其写入流水线脚本中，完整的流水线脚本如下：

node{

stage('git clone'){

//check CODE

git credentialsId: 'f30e738e-7795-4348-b3ee-eb73cd274add', url: 'http://192.168.100.10:81/root/springcloud.git'

}

stage('maven build'){

sh '''/usr/local/maven/bin/mvn package -DskipTests -f /var/jenkins\_home/workspace/springcloud'''

}

stage('image build'){

sh '''

echo $BUILD\_ID

docker build -t 192.168.100.10/springcloud/gateway:$BUILD\_ID -f /var/jenkins\_home/workspace/springcloud/gateway/Dockerfile /var/jenkins\_home/workspace/springcloud/gateway

docker build -t 192.168.100.10/springcloud/config:$BUILD\_ID -f /var/jenkins\_home/workspace/springcloud/config/Dockerfile /var/jenkins\_home/workspace/springcloud/config'''

}

stage('test'){

sh '''docker run -itd --name gateway 192.168.100.10/springcloud/gateway:$BUILD\_ID

docker ps -a|grep springcloud|grep Up

if （ $? -eq 0 ）;then

echo "Success!"

docker rm -f gateway

else

docker rm -f gateway

exit 1

fi

'''

}

stage('upload registry'){

sh '''docker login 192.168.100.10 -u=admin -p=Harbor12345

docker push 192.168.100.10/springcloud/gateway:$BUILD\_ID

docker push 192.168.100.10/springcloud/config:$BUILD\_ID'''

}

stage('deploy Rancher'){

//执行部署脚本

sh 'sed -i "s/sqshq\\\\/piggymetrics-gateway/192.168.100.10\\\\/springcloud\\\\/gateway:$BUILD\_ID/g" /var/jenkins\_home/workspace/springcloud/yaml/deployment/gateway-deployment.yaml'

sh 'sed -i "s/sqshq\\\\/piggymetrics-config/192.168.100.10\\\\/springcloud\\\\/config:$BUILD\_ID/g" /var/jenkins\_home/workspace/springcloud/yaml/deployment/config-deployment.yaml'

sh 'kubectl create ns springcloud'

sh 'kubectl apply -f /var/jenkins\_home/workspace/springcloud/yaml/deployment/gateway-deployment.yaml --kubeconfig=/root/.kube/config'

sh 'kubectl apply -f /var/jenkins\_home/workspace/springcloud/yaml/deployment/config-deployment.yaml --kubeconfig=/root/.kube/config'

sh 'kubectl apply -f /var/jenkins\_home/workspace/springcloud/yaml/svc/gateway-svc.yaml --kubeconfig=/root/.kube/config'

sh 'kubectl apply -f /var/jenkins\_home/workspace/springcloud/yaml/svc/config-svc.yaml --kubeconfig=/root/.kube/config'

}

}

脚本中所有IP均为Harbor仓库的地址。

在网页写入完整的流水线脚本，如图所示，完成后点击“应用”。

（2）开启Jenkins匿名访问

登录Jenkins首页，点击“系统管理”→“全局安全配置”，配置授权策略允许匿名用户访问，如图所示。

（3）配置Webhook

登录Gitlab，进入springcloud项目，点击左侧导航栏“Settings”→“Webhooks”，将前面记录的GitLab webhook URL地址填入URL处，禁用SSL认证，如图所示。

点击“Add webhook”添加webhook，完成后如图所示：

点击“Test”→“Push events”进行测试，如图所示：

结果返回HTTP 200则表明Webhook配置成功。

（4）创建仓库项目

登录Harbor，新建项目springcloud，访问级别设置为公开，创建完成后如图所示：

进入项目查看镜像列表，如图所示，此时为空，无任何镜像：

（1）触发构建

上传代码触发自动构建：

\[root@master \~]# docker cp /opt/repository/ jenkins:/root/.m2/

\[root@master \~]# docker restart jenkins

jenkins

\[root@master \~]# cd /opt/springcloud/

\[root@master springcloud]# git add .

\[root@master springcloud]# git commit -m "Initial commit"

\# On branch master

nothing to commit, working directory clean

\[root@master springcloud]# git push -u origin master

Username for 'http://192.168.100.10:81': root

Password for 'http://root@192.168.100.10:81':

Branch master set up to track remote branch master from origin.

Everything up-to-date

（2）Jenkins查

登录Jenkins，可以看到springcloud项目已经开始构建，如图所示：

点击项目名称查看流水线阶段视图，如图所示：

点击右侧“#1”可查看控制台输出，此处会显示构建的详细进程，如图所示：

构建完成后控制台输出如图所示：

返回项目查看流水线阶段视图，如图所示：

（3）Harbor查看

进入Harbor仓库springcloud项目查看镜像列表，可以看到已自动上传了一个gateway镜像，如图所示：

（4）Kubernetes查看

Pod的启动较慢，需等待3--5分钟。在命令行查看Pod：

\[root@master \~]# kubectl -n springcloud get pods

NAME READY STATUS RESTARTS AGE

config-6c988c4dc5-2522c 1/1 Running 0 21m

gateway-6545fc58c5-d6rgn 1/1 Running 0 21m

查看service：

\[root@master \~]# kubectl -n springcloud get service

NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE

config NodePort 10.101.42.47 \<none> 8888:30015/TCP 22m

gateway NodePort 10.100.62.39 \<none> 4000:30010/TCP 22m

通过端口30010访问服务，如图所示：

至此，完整的CI/CD流程就完成了。

#### 服务网格：创建Ingress Gateway（2分）

在提供的kubernetes镜像中，使用 project/istio/istio-1.17.2/services/bookinfo.yaml部署bookinfo应用，将Bookinfo应用部署到default命名空间下，使用Istio Gateway可以实现应用程序从外部访问，请为Bookinfo应用创建一个名为bookinfo-gateway的网关，指定所有HTTP流量通过80端口流入网格，然后将网关绑定到虚拟服务bookinfo上。

使用curl -L http://$(hostname -i):$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports\[?(@.name=="http2")].nodePort}')/productpage命令的返回结果提交到答题框。

**答案**

\[root@master \~]# curl -L http://$(hostname -i):$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports\[?(@.name=="http2")].nodePort}')/productpage | grep brand

% Total % Received % Xferd Average Speed Time Time Time Current

Dload Upload Total Spent Left Speed

100 5289 100 5289 0 0 9769 0 --:--:-- --:--:-- --:--:-- 9758 \<a class="navbar-brand" href="#">BookInfo Sample\</a>

\<a class="navbar-brand" href="#">Microservices Fabric BookInfo Demo\</a>

100 5289 100 5289 0 0 9768 0 --:--:-- --:--:-- --:--:-- 9758

**操作步骤**

\[root@master \~]# kubectl apply -f /opt/project/istio/istio-1.17.2/services/bookinfo.yaml

\[root@master \~]# vi bookinfo-gateway.yaml

apiVersion: networking.istio.io/v1alpha3

kind: Gateway

metadata:

name: bookinfo-gateway

spec:

selector:

istio: ingressgateway # use istio default controller

servers:

\- port:

number: 80

name: http

protocol: HTTP

hosts:

\- "\*"

\---

apiVersion: networking.istio.io/v1alpha3

kind: VirtualService

metadata:

name: bookinfo

spec:

hosts:

\- "\*"

gateways:

\- bookinfo-gateway

http:

\- match:

\- uri:

exact: /productpage

\- uri:

prefix: /static

\- uri:

exact: /login

\- uri:

exact: /logout

\- uri:

prefix: /api/v1/products

route:

\- destination:

host: productpage

port:

number: 9080

\[root@master \~]# kubectl apply -f bookinfo-gateway.yaml

\[root@master \~]# curl -L http://$(hostname -i):$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports\[?(@.name=="http2")].nodePort}')/productpage

#### 服务网格：创建VirtualService（2分）

在我们部署好的Bookinfo服务中，访问Bookinfo应用发现，其中一个微服务 reviews 的三个不同版本已经部署并同时运行 ，在浏览器中访问 Bookinfo 应用程序的 /productpage 并刷新几次，您会注意到，有时书评的输出包含星级评分，有时则不包含。这是因为没有明确的默认服务版本可路由，Istio将以循环方式将请求路由到所有可用版本。

（1）将default命名空间下的pod全部删除，并添加自动注入Sidecar并进行重新调度管理。

（2）请为Bookinfo应用创建DestinationRule规则，然后创建VirtualService服务，将所有流量路由到微服务的 v1 版本。

完成这些内容，我们再次访问Bookinfo网页就会发现，流量只会访问到我们的v1版本中。

1、将kubectl get namespace default --show-labels命令的返回结果提交至答题框

2、将kubectl describe vs reviews命令的返回结果提交至答题框

**答案**

\[root@master \~]# kubectl get namespace default --show-labels

NAME STATUS AGE LABELS

default Active 4h5m istio-injection=enabled,kubernetes.io/metadata.name=default

\[root@master \~]# kubectl describe vs reviews

Name: reviews

Namespace: default

Labels: \<none>

Annotations: \<none>

API Version: networking.istio.io/v1beta1

Kind: VirtualService

Metadata:

Creation Timestamp: 2023-09-07T14:28:18Z

Generation: 1

Managed Fields:

API Version: networking.istio.io/v1alpha3

Fields Type: FieldsV1

fieldsV1:

f:metadata:

f:annotations:

.:

f:kubectl.kubernetes.io/last-applied-configuration:

f:spec:

.:

f:hosts:

f:http:

Manager: kubectl-client-side-apply

Operation: Update

Time: 2023-09-07T14:28:18Z

Resource Version: 41317

UID: 295b72bf-2704-442d-8e8f-e9b8d9c20ab3

Spec:

Hosts:

reviews

Http:

Route:

Destination:

Host: reviews

Subset: v1

Events: \<none>

**操作步骤**

\[root@master \~]# kubectl label namespace default istio-injection=enabled

namespace/default labeled

\[root@master \~]# kubectl delete pod --all

\[root@master \~]# vi dr.yaml

apiVersion: networking.istio.io/v1alpha3

kind: DestinationRule

metadata:

name: productpage

spec:

host: productpage

subsets:

\- name: v1

labels:

version: v1

\---

apiVersion: networking.istio.io/v1alpha3

kind: DestinationRule

metadata:

name: reviews

spec:

host: reviews

subsets:

\- name: v1

labels:

version: v1

\- name: v2

labels:

version: v2

\- name: v3

labels:

version: v3

\---

apiVersion: networking.istio.io/v1alpha3

kind: DestinationRule

metadata:

name: ratings

spec:

host: ratings

subsets:

\- name: v1

labels:

version: v1

\- name: v2

labels:

version: v2

\- name: v2-mysql

labels:

version: v2-mysql

\- name: v2-mysql-vm

labels:

version: v2-mysql-vm

\---

apiVersion: networking.istio.io/v1alpha3

kind: DestinationRule

metadata:

name: details

spec:

host: details

subsets:

\- name: v1

labels:

version: v1

\- name: v2

labels:

version: v2

\[root@master \~]# vi vs.yaml

apiVersion: networking.istio.io/v1alpha3

kind: VirtualService

metadata:

name: productpage

spec:

hosts:

\- productpage

http:

\- route:

\- destination:

host: productpage

subset: v1

\---

apiVersion: networking.istio.io/v1alpha3

kind: VirtualService

metadata:

name: reviews

spec:

hosts:

\- reviews

http:

\- route:

\- destination:

host: reviews

subset: v1

\---

apiVersion: networking.istio.io/v1alpha3

kind: VirtualService

metadata:

name: ratings

spec:

hosts:

\- ratings

http:

\- route:

\- destination:

host: ratings

subset: v1

\---

apiVersion: networking.istio.io/v1alpha3

kind: VirtualService

metadata:

name: details

spec:

hosts:

\- details

http:

\- route:

\- destination:

host: details

subset: v1

\[root@master \~]# kubectl apply -f dr.yaml

destinationrule.networking.istio.io/productpage created

destinationrule.networking.istio.io/reviews created

destinationrule.networking.istio.io/ratings created

destinationrule.networking.istio.io/details created

\[root@master \~]# kubectl apply -f vs.yaml

virtualservice.networking.istio.io/productpage created

virtualservice.networking.istio.io/reviews created

virtualservice.networking.istio.io/ratings created

virtualservice.networking.istio.io/details created

#### KubeVirt运维：创建VM（2分）

使用提供的镜像（images/fedora-virt\_v1.0.tar）在default命名空间下创建一台VM，名称为test-vm，内存大小为1Gi，磁盘驱动：virtio，运行策略：Manual。

(1)如果VM出现调度失败的情况，请修改kubevirt配置，让其支持硬件仿真。将kubectl edit kubevirts.kubevirt.io -n kubevirt命令的返回结果提交至答题框。

(2)将kubectl describe vm test-vm命令的返回结果提交到答题框。

**答案**

\[root@master \~]# kubectl edit kubevirts.kubevirt.io -n kubevirt kubevirt

configuration:

developerConfiguration:

useEmulation: true

\[root@master \~]# kubectl describe vm test-vm

Name: test-vm

Namespace: default

Labels: \<none>

Annotations: kubevirt.io/latest-observed-api-version: v1

kubevirt.io/storage-observed-api-version: v1alpha3

API Version: kubevirt.io/v1

Kind: VirtualMachine

Metadata:

Creation Timestamp: 2023-05-21T20:03:15Z

Finalizers:

kubevirt.io/virtualMachineControllerFinalize

Generation: 1

Managed Fields:

API Version: kubevirt.io/v1alpha3

Fields Type: FieldsV1

fieldsV1:

f:metadata:

f:annotations:

f:kubevirt.io/latest-observed-api-version:

f:kubevirt.io/storage-observed-api-version:

f:finalizers:

.:

v:"kubevirt.io/virtualMachineControllerFinalize":

Manager: Go-http-client

Operation: Update

Time: 2023-05-21T20:03:15Z

API Version: kubevirt.io/v1

Fields Type: FieldsV1

fieldsV1:

f:metadata:

f:annotations:

.:

f:kubectl.kubernetes.io/last-applied-configuration:

f:spec:

.:

f:runStrategy:

f:template:

.:

f:spec:

.:

f:domain:

.:

f:devices:

.:

f:disks:

f:resources:

.:

f:requests:

.:

f:memory:

f:volumes:

Manager: kubectl-client-side-apply

Operation: Update

Time: 2023-05-21T20:03:15Z

API Version: kubevirt.io/v1alpha3

Fields Type: FieldsV1

fieldsV1:

f:status:

.:

f:conditions:

f:created:

f:desiredGeneration:

f:observedGeneration:

f:printableStatus:

f:ready:

f:volumeSnapshotStatuses:

Manager: Go-http-client

Operation: Update

Subresource: status

Time: 2023-05-21T20:42:03Z

Resource Version: 58020

UID: e9cfd677-9ca2-47df-9d17-313978636ebc

Spec:

Run Strategy: Manual

Template:

Metadata:

Creation Timestamp: \<nil>

Spec:

Domain:

Devices:

Disks:

Disk:

Bus: virtio

Name: containerdisk

Machine:

Type: q35

Resources:

Requests:

Memory: 1Gi

Volumes:

Container Disk:

Image: fedora-virt:v1.0

Image Pull Policy: IfNotPresent

Name: containerdisk

Status:

Conditions:

Last Probe Time: \<nil>

Last Transition Time: 2023-05-21T20:42:02Z

Status: True

Type: Ready

Last Probe Time: \<nil>

Last Transition Time: \<nil>

Message: cannot migrate VMI which does not use masquerade to connect to the pod network or bridge with kubevirt.io/allow-pod-bridge-network-live-migration VM annotation

Reason: InterfaceNotLiveMigratable

Status: False

Type: LiveMigratable

Created: true

Desired Generation: 1

Observed Generation: 1

Printable Status: Running

Ready: true

Volume Snapshot Statuses:

Enabled: false

Name: containerdisk

Reason: Snapshot is not supported for this volumeSource type \[containerdisk]

Events:

Type Reason Age From Message

\---- ------ ---- ---- -------

Normal SuccessfulDelete 37m virtualmachine-controller Stopped the virtual machine by deleting the virtual machine instance dbcdbd0c-b49b-40fa-b779-f9c2a4be7390

**操作步骤**

\[root@master \~]# kubectl edit kubevirts.kubevirt.io -n kubevirt kubevirt

configuration:

developerConfiguration:

useEmulation: true

\[root@master \~]# cat vm.yaml

apiVersion: kubevirt.io/v1

kind: VirtualMachine

metadata:

name: test-vm

spec:

runStrategy: Manual

template:

spec:

domain:

resources:

requests:

memory: 1Gi

devices:

disks:

\- name: containerdisk

disk:

bus: virtio

volumes:

\- name: containerdisk

containerDisk:

image: fedora-virt:v1.0

imagePullPolicy: IfNotPresent

\[root@master \~]# kubectl apply -f vm.yaml

virtualmachine.kubevirt.io/test-vm created

\[root@master \~]# virtctl start test-vm

VM test-vm was scheduled to start

\[root@master \~]# kubectl describe vm test-vm

Name: test-vm

Namespace: default

Labels: \<none>

Annotations: kubevirt.io/latest-observed-api-version: v1

kubevirt.io/storage-observed-api-version: v1alpha3

API Version: kubevirt.io/v1

Kind: VirtualMachine

Metadata:

Creation Timestamp: 2023-05-21T20:03:15Z

Finalizers:

kubevirt.io/virtualMachineControllerFinalize

Generation: 1

Managed Fields:

API Version: kubevirt.io/v1alpha3

Fields Type: FieldsV1

fieldsV1:

f:metadata:

f:annotations:

f:kubevirt.io/latest-observed-api-version:

f:kubevirt.io/storage-observed-api-version:

f:finalizers:

.:

v:"kubevirt.io/virtualMachineControllerFinalize":

Manager: Go-http-client

Operation: Update

Time: 2023-05-21T20:03:15Z

API Version: kubevirt.io/v1

Fields Type: FieldsV1

fieldsV1:

f:metadata:

f:annotations:

.:

f:kubectl.kubernetes.io/last-applied-configuration:

f:spec:

.:

f:runStrategy:

f:template:

.:

f:spec:

.:

f:domain:

.:

f:devices:

.:

f:disks:

f:resources:

.:

f:requests:

.:

f:memory:

f:volumes:

Manager: kubectl-client-side-apply

Operation: Update

Time: 2023-05-21T20:03:15Z

API Version: kubevirt.io/v1alpha3

Fields Type: FieldsV1

fieldsV1:

f:status:

.:

f:conditions:

f:created:

f:desiredGeneration:

f:observedGeneration:

f:printableStatus:

f:ready:

f:volumeSnapshotStatuses:

Manager: Go-http-client

Operation: Update

Subresource: status

Time: 2023-05-21T20:42:03Z

Resource Version: 58020

UID: e9cfd677-9ca2-47df-9d17-313978636ebc

Spec:

Run Strategy: Manual

Template:

Metadata:

Creation Timestamp: \<nil>

Spec:

Domain:

Devices:

Disks:

Disk:

Bus: virtio

Name: containerdisk

Machine:

Type: q35

Resources:

Requests:

Memory: 1Gi

Volumes:

Container Disk:

Image: fedora-virt:v1.0

Image Pull Policy: IfNotPresent

Name: containerdisk

Status:

Conditions:

Last Probe Time: \<nil>

Last Transition Time: 2023-05-21T20:42:02Z

Status: True

Type: Ready

Last Probe Time: \<nil>

Last Transition Time: \<nil>

Message: cannot migrate VMI which does not use masquerade to connect to the pod network or bridge with kubevirt.io/allow-pod-bridge-network-live-migration VM annotation

Reason: InterfaceNotLiveMigratable

Status: False

Type: LiveMigratable

Created: true

Desired Generation: 1

Observed Generation: 1

Printable Status: Running

Ready: true

Volume Snapshot Statuses:

Enabled: false

Name: containerdisk

Reason: Snapshot is not supported for this volumeSource type \[containerdisk]

Events:

Type Reason Age From Message

\---- ------ ---- ---- -------

Normal SuccessfulDelete 37m virtualmachine-controller Stopped the virtual machine by deleting the virtual machine instance dbcdbd0c-b49b-40fa-b779-f9c2a4be7390

#### KubeVirt运维：开启功能优化（1分）

在KubeVirt中有很多功能，为了更好的使用，更改kubevirt配置开启以下功能

（1）启用快照/恢复支持功能

（2）启动热插拔卷功能

（3）启动实时迁移功能

（4）启动边车功能

（5）启动主机磁盘功能

更改完成后，将kubectl describe kubevirt kubevirt -n kubevirt命令的返回结果提交到答题框。

**答案**

\[root@master \~]# kubectl describe kubevirt kubevirt -n kubevirt

Name: kubevirt

Namespace: kubevirt

Labels: \<none>

Annotations: kubevirt.io/latest-observed-api-version: v1

kubevirt.io/storage-observed-api-version: v1alpha3

API Version: kubevirt.io/v1

Kind: KubeVirt

Metadata:

Creation Timestamp: 2023-09-07T10:29:47Z

Finalizers:

foregroundDeleteKubeVirt

Generation: 3

Managed Fields:

API Version: kubevirt.io/v1

Fields Type: FieldsV1

fieldsV1:

f:metadata:

f:annotations:

.:

f:kubectl.kubernetes.io/last-applied-configuration:

f:spec:

.:

f:certificateRotateStrategy:

f:configuration:

.:

f:developerConfiguration:

f:customizeComponents:

f:imagePullPolicy:

f:workloadUpdateStrategy:

Manager: kubectl-client-side-apply

Operation: Update

Time: 2023-09-07T10:29:47Z

API Version: kubevirt.io/v1alpha3

Fields Type: FieldsV1

fieldsV1:

f:metadata:

f:annotations:

f:kubevirt.io/latest-observed-api-version:

f:kubevirt.io/storage-observed-api-version:

f:finalizers:

.:

v:"foregroundDeleteKubeVirt":

Manager: Go-http-client

Operation: Update

Time: 2023-09-07T10:30:13Z

API Version: kubevirt.io/v1

Fields Type: FieldsV1

fieldsV1:

f:spec:

f:configuration:

f:developerConfiguration:

f:featureGates:

Manager: kubectl-edit

Operation: Update

Time: 2023-09-07T15:08:48Z

API Version: kubevirt.io/v1alpha3

Fields Type: FieldsV1

fieldsV1:

f:status:

.:

f:conditions:

f:generations:

f:observedDeploymentConfig:

f:observedDeploymentID:

f:observedGeneration:

f:observedKubeVirtRegistry:

f:observedKubeVirtVersion:

f:operatorVersion:

f:outdatedVirtualMachineInstanceWorkloads:

f:phase:

f:targetDeploymentConfig:

f:targetDeploymentID:

f:targetKubeVirtRegistry:

f:targetKubeVirtVersion:

Manager: Go-http-client

Operation: Update

Subresource: status

Time: 2023-09-07T15:09:02Z

Resource Version: 48407

UID: 927cbaa5-0e26-4dbc-9584-7353cf5e5420

Spec:

Certificate Rotate Strategy:

Configuration:

Developer Configuration:

Feature Gates:

Snapshot

HotplugVolumes

LiveMigration

Sidecar

HostDisk

Customize Components:

Image Pull Policy: IfNotPresent

**操作步骤**

\[root@master \~]# kubectl edit kubevirt kubevirt -n kubevirt

\# Please edit the object below. Lines beginning with a '#' will be ignored,

\# and an empty file will abort the edit. If an error occurs while saving this file will be

\# reopened with the relevant failures.

\#

apiVersion: kubevirt.io/v1

kind: KubeVirt

metadata:

annotations:

kubectl.kubernetes.io/last-applied-configuration: |

{"apiVersion":"kubevirt.io/v1","kind":"KubeVirt","metadata":{"annotations":{},"name":"kubevirt","namespace":"kubevirt"},

"spec":{"certificateRotateStrategy":{},"configuration":{"developerConfiguration":{"featureGates":\[]\}},"customizeComponents":{}

,"imagePullPolicy":"IfNotPresent","workloadUpdateStrategy":{\}}}

kubevirt.io/latest-observed-api-version: v1

kubevirt.io/storage-observed-api-version: v1alpha3

creationTimestamp: "2023-09-07T10:29:47Z"

finalizers:

\- foregroundDeleteKubeVirt

generation: 3

name: kubevirt

namespace: kubevirt

resourceVersion: "48407"

uid: 927cbaa5-0e26-4dbc-9584-7353cf5e5420

spec:

certificateRotateStrategy: {}

configuration:

developerConfiguration:

featureGates:

\- Snapshot

\- HotplugVolumes

\- LiveMigration

\- Sidecar

\- HostDisk

customizeComponents: {}

#### Deployment管理：创建deployment（2分）

在master节点打上标签“tty=master”，然后编写deployment.yaml文件创建deployment，具体的要求如下。

(1)创建的deployment的名字为test-deployment

(2)使用nginx镜像

(3)Pod只能调度到标签为“tty=master”的节点上

创建deployment后将cat deployment.yaml &\&kubectl describe deployment test-deployment命令的返回结果提交至答题框。

**答案**

\[root@master \~]# cat deployment.yaml &\&kubectl describe deployment test-deployment

apiVersion: apps/v1

kind: Deployment

metadata:

name: test-deployment

namespace: default

spec:

selector:

matchLabels:

app: nginx

template:

metadata:

labels:

app: nginx

spec:

containers:

\- name: nginx

image: 192.168.200.10/library/nginx:latest

imagePullPolicy: IfNotPresent

affinity:

nodeAffinity:

requiredDuringSchedulingIgnoredDuringExecution:

nodeSelectorTerms:

\- matchExpressions:

\- key: tty

operator: In

values:

\- master

Name: test-deployment

Namespace: default

CreationTimestamp: Thu, 07 Sep 2023 12:29:43 -0400

Labels: \<none>

Annotations: deployment.kubernetes.io/revision: 3

Selector: app=nginx

Replicas: 1 desired | 1 updated | 1 total | 1 available | 0 unavailable

StrategyType: RollingUpdate

MinReadySeconds: 0

RollingUpdateStrategy: 25% max unavailable, 25% max surge

Pod Template:

Labels: app=nginx

Containers:

nginx:

Image: 192.168.200.10/library/nginx:latest

Port: \<none>

Host Port: \<none>

Environment: \<none>

Mounts: \<none>

Volumes: \<none>

Conditions:

Type Status Reason

\---- ------ ------

Available True MinimumReplicasAvailable

Progressing True NewReplicaSetAvailable

OldReplicaSets: \<none>

NewReplicaSet: test-deployment-6b49cc498f (1/1 replicas created)

Events:

Type Reason Age From Message

\---- ------ ---- ---- -------

Normal ScalingReplicaSet 14m deployment-controller Scaled up replica set test-deployment-575667d677 to 1

**操作步骤**

\[root@master \~]# kubectl label nodes master tty=master

\[root@master \~]# cat deployment.yaml

apiVersion: apps/v1

kind: Deployment

metadata:

name: test-deployment

namespace: default

spec:

selector:

matchLabels:

app: nginx

template:

metadata:

labels:

app: nginx

spec:

containers:

\- name: nginx

image: 192.168.200.10/library/nginx:latest

imagePullPolicy: IfNotPresent

affinity:

nodeAffinity:

requiredDuringSchedulingIgnoredDuringExecution:

nodeSelectorTerms:

\- matchExpressions:

\- key: tty

operator: In

values:

\- master

\[root@master \~]# kubectl apply -f deployment.yaml

deployment.apps/test-deployment created

#### PV卷管理：创建PV卷（1分）

创建一个 pv，名字为 app-config，大小为 2Gi， 访问权限为 ReadWriteMany。Volume 的类型为 hostPath，路径为 /srv/app-config。

创建完成后，将kubectl describe pv命令的返回结果提交至答题框。

**答案**

\[root@master \~]# kubectl describe pv

Name: app-config

Labels: \<none>

Annotations: \<none>

Finalizers: \[kubernetes.io/pv-protection]

StorageClass:

Status: Available

Claim:

Reclaim Policy: Retain

Access Modes: RWX

VolumeMode: Filesystem

Capacity: 2Gi

Node Affinity: \<none>

Message:

Source:

Type: HostPath (bare host directory volume)

Path: /srv/app-config

HostPathType:

Events: \<none>

**操作步骤**

\[root@master exam-test1]# mkdir -pv /srv/app-config

\[root@master exam-test1]# vi pv.yaml

apiVersion: v1

kind: PersistentVolume

metadata:

name: app-config

spec:

capacity:

storage: 2Gi

accessModes:

\- ReadWriteMany

hostPath:

path: "/srv/app-config"

\[root@master exam-test1]# kubectl apply -f pv.yaml

\[root@master exam-test1]# kubectl describe pv

Name: app-config

Labels: \<none>

Annotations: \<none>

Finalizers: \[kubernetes.io/pv-protection]

StorageClass:

Status: Available

Claim:

Reclaim Policy: Retain

Access Modes: RWX

VolumeMode: Filesystem

Capacity: 2Gi

Node Affinity: \<none>

Message:

Source:

Type: HostPath (bare host directory volume)

Path: /srv/app-config

HostPathType:

Events: \<none>

#### Ingress资源管理：创建Ingress（1分）

创建一个新的 nginx lngress资源：

（1）名称：pong

（2）Namespace：ing-internal

（3）使用服务端口 5678 在路径 /hello 上公开服务 hello

将kubectl describe ingress -n ing-internal命令的返回结果提交至答题框。

**答案**

\[root@master exam-test1]# kubectl describe ingress -n ing-internal

Name: pong

Labels: \<none>

Namespace: ing-internal

Address:

Ingress Class: nginx

Default backend: \<default>

Rules:

Host Path Backends

\---- ---- --------

\*

/hello test:5678

Annotations: nginx.ingress.kubernetes.io/rewrite-target: /

Events: \<none>

**操作步骤**

\[root@master exam-test1]# kubectl create ns ing-internal

namespace/ing-internal created

\[root@master exam-test1]# vi ingress.yaml

apiVersion: networking.k8s.io/v1

kind: Ingress

metadata:

name: pong

namespace: ing-internal

annotations:

nginx.ingress.kubernetes.io/rewrite-target: /

spec:

ingressClassName: nginx

rules:

\- http:

paths:

\- path: /hello

pathType: Prefix

backend:

service:

name: test

port:

number: 5678

\[root@master exam-test1]# kubectl apply -f ingress.yaml

ingress.networking.k8s.io/pong created

\[root@master exam-test1]# kubectl describe ingress -n ing-internal

Name: pong

Labels: \<none>

Namespace: ing-internal

Address:

Ingress Class: nginx

Default backend: \<default>

Rules:

Host Path Backends

\---- ---- --------

\*

/hello test:5678

Annotations: nginx.ingress.kubernetes.io/rewrite-target: /

Events: \<none>

### 任务3 部署Owncloud网盘服务（6分）

ownCloud 是一个开源免费专业的私有云存储项目，它能帮你快速在个人电脑或服务器上架设一套专属的私有云文件同步网盘，可以像 百度云那样实现文件跨平台同步、共享、版本控制、团队协作等。

#### 创建PV和PVC（1分）

编写yaml文件(文件名自定义)创建PV和PVC来提供持久化存储，以便保存 ownCloud 服务中的文件和数据。

要求：PV（名称为owncloud-pv，访问模式为读写，只能被单个节点挂载;存储为5Gi;存储类型为hostPath,存储路径自定义）；PVC（名称为owncloud-pvc，访问模式为读写，只能被单个节点挂载;申请存储空间大小为5Gi）。

将kubectl get pv,pvc命令的返回结果提交到答题框。

**操作步骤**

\# cat owncloud-pvc.yaml

apiVersion: v1

kind: PersistentVolume

metadata:

name: owncloud-pv

spec:

accessModes:

\- ReadWriteOnce

capacity:

storage: 5Gi

hostPath:

path: /data/owncloud

\---

apiVersion: v1

kind: PersistentVolumeClaim

metadata:

name: owncloud-pvc

spec:

accessModes:

\- ReadWriteOnce

resources:

requests:

storage: 5Gi

\# kubectl apply -f owncloud-pvc.yaml

**答案**

\[root@master \~]# kubectl get pv,pvc

NAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS REASON AGE

persistentvolume/app-config 2Gi RWX Retain Available 2m25s

persistentvolume/owncloud-pv 5Gi RWO Retain Bound default/owncloud-pvc 18s

NAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS AGE

persistentvolumeclaim/owncloud-pvc Bound owncloud-pv 5Gi RWO 18s

#### 配置ConfigMap（1分）

编写yaml文件(文件名自定义)创建一个configMap对象，名称为owncloud-config，指定OwnCloud的环境变量。登录账号对应的环境变量为OWNCLOUD\_ADMIN\_USERNAME,密码对应的环境变量为OWNCLOUD\_ADMIN\_PASSWORD。（变量值自定义）

将kubectl get ConfigMap命令的返回结果提交到答题框。

**操作步骤**

\# cat owncloud-configmap.yaml

apiVersion: v1

kind: ConfigMap

metadata:

name: owncloud-config

data:

OWNCLOUD\_ADMIN\_USERNAME: “admin”

OWNCLOUD\_ADMIN\_PASSWORD: “123456”

\# kubectl apply -f owncloud-configmap.yaml

**答案**

\[root@master \~]# kubectl get ConfigMap

NAME DATA AGE

istio-ca-root-cert 1 34m

kube-root-ca.crt 1 40m

owncloud-config 2 7s

#### 创建Secret（1分）

编写yaml文件(文件名自定义)创建一个Secret对象，名称为owncloud-db-password，以保存OwnCloud数据库的密码。对原始密码采用base64编码格式进行加密。

将kubectl get Secret命令的返回结果提交到答题框。

**操作步骤**

\# echo 123456 | base64

MTIzNDU2Cg==

\# cat owncloud-secret.yaml

apiVersion: v1

kind: Secret

metadata:

name: owncloud-db-password

type: Opaque

data:

password: MTIzNDU2Cg==

\# kubectl apply -f owncloud-secret.yaml

**答案**

\[root@master \~]# kubectl get Secret

NAME TYPE DATA AGE

owncloud-db-password Opaque 1 2s

#### 部署Owncloud Deployment应用（2分）

编写yaml文件(文件名自定义) 创建Deployment对象, 指定OwnCloud的容器和相关的环境变量。(Deployment资源命名为owncloud-deployment,镜像为Harbor仓库中的owncloud:latest，存储的挂载路径为/var/www/html,其它根据具体情况进行配置)

将kubectl describe pod $(kubectl get pod | grep owncloud-deployment | awk -F \ '{print $1}')命令的返回结果提交到答题框。

**操作步骤**

\# cat owncloud-deploy.yaml

apiVersion: apps/v1

kind: Deployment

metadata:

name: owncloud-deployment

spec:

replicas: 1

selector:

matchLabels:

app: owncloud

template:

metadata:

labels:

app: owncloud

spec:

containers:

\- name: owncloud

image: 192.168.100.91/library/owncloud:latest

imagePullPolicy: IfNotPresent

envFrom:

\- configMapRef:

name: owncloud-config

env:

\- name: OWNCLOUD\_DB\_PASSWORD

valueFrom:

secretKeyRef:

name: owncloud-db-password

key: password

ports:

\- containerPort: 80

volumeMounts:

\- name: owncloud-pv

mountPath: /var/www/html

volumes:

\- name: owncloud-pv

persistentVolumeClaim:

claimName: owncloud-pvc

\# kubectl apply -f owncloud-deploy.yaml

**答案**

\[root@master \~]# kubectl describe pod $(kubectl get pod | grep owncloud-deployment | awk -F \ '{print $1}')

Name: owncloud-deployment-7d5b747db7-mk2zf

Namespace: default

Priority: 0

Service Account: default

Node: node/192.168.100.74

Start Time: Tue, 18 Jun 2024 09:55:59 +0000

Labels: app=owncloud

pod-template-hash=7d5b747db7

security.istio.io/tlsMode=istio

service.istio.io/canonical-name=owncloud

service.istio.io/canonical-revision=latest

Annotations: k8s.v1.cni.cncf.io/network-status:

\[{

"name": "cbr0",

"interface": "eth0",

"ips": \[

"10.244.1.26"

],

"mac": "4a:e5:03:7a:2b:3a",

"default": true,

"dns": {}

}]

k8s.v1.cni.cncf.io/networks-status:

\[{

"name": "cbr0",

"interface": "eth0",

"ips": \[

"10.244.1.26"

],

"mac": "4a:e5:03:7a:2b:3a",

"default": true,

"dns": {}

}]

kubectl.kubernetes.io/default-container: owncloud

kubectl.kubernetes.io/default-logs-container: owncloud

prometheus.io/path: /stats/prometheus

prometheus.io/port: 15020

prometheus.io/scrape: true

sidecar.istio.io/status:

{"initContainers":\["istio-init"],"containers":\["istio-proxy"],"volumes":\["workload-socket","credential-socket","workload-certs","istio-env...

Status: Running

IP: 10.244.1.26

IPs:

IP: 10.244.1.26

Controlled By: ReplicaSet/owncloud-deployment-7d5b747db7

Init Containers:

istio-init:

Container ID: containerd://c5ae2544514459bad93c5362081ccc5676fdd6e4a66b0d84dbd1f3a8082e8f41

Image: docker.io/istio/proxyv2:1.17.2

Image ID: sha256:3944a6baf515cabc77e05986b76d498f77b5c47a65aa736cb6c7086f368b8339

Port: \<none>

Host Port: \<none>

Args:

istio-iptables

\-p

15001

\-z

15006

\-u

1337

\-m

REDIRECT

\-i

\*

\-x

\-b

\*

\-d

15090,15021,15020

\--log\_output\_level=default:info

State: Terminated

Reason: Completed

Exit Code: 0

Started: Tue, 18 Jun 2024 09:55:59 +0000

Finished: Tue, 18 Jun 2024 09:55:59 +0000

Ready: True

Restart Count: 0

Limits:

cpu: 2

memory: 1Gi

Requests:

cpu: 10m

memory: 40Mi

Environment: \<none>

Mounts:

/var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-sxj5s (ro)

Containers:

owncloud:

Container ID: containerd://d0a69114686b4b18b713a7f3bccb2ee3c2985f2d5de94045f9f35264d59dfe72

Image: 192.168.100.56/library/owncloud:latest

Image ID: 192.168.100.56/library/owncloud@sha256:5c77bfdf8cfaf99ec94309be2687032629f4f985d6bd388354dfd85475aa5f21

Port: 80/TCP

Host Port: 0/TCP

State: Running

Started: Tue, 18 Jun 2024 09:56:07 +0000

Ready: True

Restart Count: 0

Environment Variables from:

owncloud-config ConfigMap Optional: false

Environment:

OWNCLOUD\_DB\_PASSWORD: \<set to the key 'password' in secret 'owncloud-db-password'> Optional: false

Mounts:

/var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-sxj5s (ro)

/var/www/html from owncloud-pv (rw)

istio-proxy:

Container ID: containerd://3fac8e5b39188ed3be3b898449ae5020b49f5e720d9065fc71794be9a3a5bdda

Image: docker.io/istio/proxyv2:1.17.2

Image ID: sha256:3944a6baf515cabc77e05986b76d498f77b5c47a65aa736cb6c7086f368b8339

Port: 15090/TCP

Host Port: 0/TCP

Args:

proxy

sidecar

\--domain

$(POD\_NAMESPACE).svc.cluster.local

\--proxyLogLevel=warning

\--proxyComponentLogLevel=misc:error

\--log\_output\_level=default:info

\--concurrency

2

State: Running

Started: Tue, 18 Jun 2024 09:56:08 +0000

Ready: True

Restart Count: 0

Limits:

cpu: 2

memory: 1Gi

Requests:

cpu: 10m

memory: 40Mi

Readiness: http-get http://:15021/healthz/ready delay=1s timeout=3s period=2s #success=1 #failure=30

Environment:

JWT\_POLICY: third-party-jwt

PILOT\_CERT\_PROVIDER: istiod

CA\_ADDR: istiod.istio-system.svc:15012

POD\_NAME: owncloud-deployment-7d5b747db7-mk2zf (v1:metadata.name)

POD\_NAMESPACE: default (v1:metadata.namespace)

INSTANCE\_IP: (v1:status.podIP)

SERVICE\_ACCOUNT: (v1:spec.serviceAccountName)

HOST\_IP: (v1:status.hostIP)

PROXY\_CONFIG: {}

ISTIO\_META\_POD\_PORTS: \[

{"containerPort":80,"protocol":"TCP"}

]

ISTIO\_META\_APP\_CONTAINERS: owncloud

ISTIO\_META\_CLUSTER\_ID: Kubernetes

ISTIO\_META\_NODE\_NAME: (v1:spec.nodeName)

ISTIO\_META\_INTERCEPTION\_MODE: REDIRECT

ISTIO\_META\_WORKLOAD\_NAME: owncloud-deployment

ISTIO\_META\_OWNER: kubernetes://apis/apps/v1/namespaces/default/deployments/owncloud-deployment

ISTIO\_META\_MESH\_ID: cluster.local

TRUST\_DOMAIN: cluster.local

Mounts:

/etc/istio/pod from istio-podinfo (rw)

/etc/istio/proxy from istio-envoy (rw)

/var/lib/istio/data from istio-data (rw)

/var/run/secrets/credential-uds from credential-socket (rw)

/var/run/secrets/istio from istiod-ca-cert (rw)

/var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-sxj5s (ro)

/var/run/secrets/tokens from istio-token (rw)

/var/run/secrets/workload-spiffe-credentials from workload-certs (rw)

/var/run/secrets/workload-spiffe-uds from workload-socket (rw)

Conditions:

Type Status

Initialized True

Ready True

ContainersReady True

PodScheduled True

Volumes:

workload-socket:

Type: EmptyDir (a temporary directory that shares a pod's lifetime)

Medium:

SizeLimit: \<unset>

credential-socket:

Type: EmptyDir (a temporary directory that shares a pod's lifetime)

Medium:

SizeLimit: \<unset>

workload-certs:

Type: EmptyDir (a temporary directory that shares a pod's lifetime)

Medium:

SizeLimit: \<unset>

istio-envoy:

Type: EmptyDir (a temporary directory that shares a pod's lifetime)

Medium: Memory

SizeLimit: \<unset>

istio-data:

Type: EmptyDir (a temporary directory that shares a pod's lifetime)

Medium:

SizeLimit: \<unset>

istio-podinfo:

Type: DownwardAPI (a volume populated by information about the pod)

Items:

metadata.labels -> labels

metadata.annotations -> annotations

istio-token:

Type: Projected (a volume that contains injected data from multiple sources)

TokenExpirationSeconds: 43200

istiod-ca-cert:

Type: ConfigMap (a volume populated by a ConfigMap)

Name: istio-ca-root-cert

Optional: false

owncloud-pv:

Type: PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)

ClaimName: owncloud-pvc

ReadOnly: false

kube-api-access-sxj5s:

Type: Projected (a volume that contains injected data from multiple sources)

TokenExpirationSeconds: 3607

ConfigMapName: kube-root-ca.crt

ConfigMapOptional: \<nil>

DownwardAPI: true

QoS Class: Burstable

Node-Selectors: \<none>

Tolerations: node.kubernetes.io/not-ready:NoExecute op=Exists for 300s

node.kubernetes.io/unreachable:NoExecute op=Exists for 300s

Events: \<none>

#### 创建Service（1分）

编写yaml文件(文件名自定义)创建一个Service对象使用NodePort的方式将OwnCloud公开到集群外部，名称为owncloud-service。通过http://IP:端口号可查看owncloud。

将kubectl get svc -A命令的返回结果提交到答题框。

**操作步骤**

\# cat owncloud-svc.yaml

apiVersion: v1

kind: Service

metadata:

name: owncloud-service

spec:

selector:

app: owncloud

ports:

\- name: http

port: 80

type: NodePort

\# kubectl apply -f owncloud-svc.yaml

**答案**

\#kubectl get svc -A

NAMESPACE NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE

default kubernetes ClusterIP 10.96.0.1 \<none> 443/TCP 24h

default owncloud-service NodePort 10.98.228.242 \<none> 80:31024/TCP 17m

kube-system kube-dns ClusterIP 10.96.0.10 \<none> 53/UDP,53/TCP,9153/TCP 24h

kubernetes-dashboard dashboard-metrics-scraper ClusterIP 10.105.211.63 \<none> 8000/TCP 22h

kubernetes-dashboard kubernetes-dashboard NodePort 10.104.143.162 \<none> 443:30001/TCP 22h

### C模块1：企业级应用的自动化部署和运维（40分）

### 任务1、应用的自动化部署和运维

虚拟机与环境规划表3

| 设备名称  | 主机名            | 接口   | IP地址                                             | 角色                           |
| ----- | -------------- | ---- | ------------------------------------------------ | ---------------------------- |
| 云服务器1 | ansible        | eth0 | <p>公网IP:********</p><p>私网IP:192.168.100.*/24</p> | ansible                      |
| 云服务器2 | zabbix\_server | eth0 | <p>公网IP:********</p><p>私网IP:192.168.100.*/24</p> | zabbix\_server、zabbix\_agent |
| 云服务器3 | zabbix\_agent  | eth0 | <p>公网IP:********</p><p>私网IP:192.168.100.*/24</p> | zabbix\_agent                |

**说明：**

1\. 上表中的公网IP以自己云主机显示为准，每个人的公网IP不同。使用第三方软件远程连接云主机，使用公网IP连接。

2.华为云中云主机名字已命好，直接使用对应名字的云主机即可。

3.竞赛用到的软件包都在云主机/root下。

部署方式：在ansible节点采用playbook分别部署zabbix\_server节点和zabbix\_agent节点。

#### 安装ansible（2分）

修改主机名ansible节点主机名ansible, zabbix\_server节点主机名为zabbix\_server,zabbix\_agent节点主机名为zabbix\_agent,使用提供的软件包autoDeployment.tar在ansible节点安装ansible。

将ansible --version命令的返回结果提交到答题框。

**答案**

\[root@ansible \~]# ansible --version

ansible 2.9.27

config file = /etc/ansible/ansible.cfg

configured module search path = \[u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']

ansible python module location = /usr/lib/python2.7/site-packages/ansible

executable location = /usr/bin/ansible

python version = 2.7.5 (default, Oct 14 2020, 14:45:30) \[GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]

**操作步骤**

\[root@ansible \~]# hostnamectl set-hostname ansible

\[root@ansible \~]# logout

\[root@zabbix\_server \~]# hostnamectl set-hostname zabbix\_server

\[root@zabbix\_server \~]# logout

\[root@zabbix\_agent \~]# hostnamectl set-hostname zabbix\_agent

\[root@zabbix\_agent \~]# logout

\[root@ansible \~]# mkdir /opt/centos

\[root@ansible \~]# mount A\_module\_software\_package/CentOS-7-x86\_64-DVD-2009.iso /mnt/

mount: /dev/loop0 is write-protected, mounting read-only

\[root@ansible \~]# cp -rf /mnt/\* /opt/centos/

\[root@ansible \~]# umount /mnt

\[root@ansible \~]# tar -zxvf C\_module\_software\_package/autoDeployment.tar -C /opt/

\[root@ansible \~]# rm -rf /etc/yum.repos.d/\*

\[root@ansible \~]# vim /etc/yum.repos.d/local.repo

\[auto]

name=auto

baseurl=file:///opt/autoDeployment

enabled=1

gpgcheck=0

\[centos]

name=centos

baseurl=file:///opt/centos

enabled=1

gpgcheck=0

\[root@ansible \~]# yum clean all

\[root@ansible \~]# yum repolist

\[root@ansible \~]# yum -y install ansible

#### 配置免密登录（1分）

在ansible节点配置主机映射将IP地址映射为主机名，三台主机名分别为ansible、zabbix\_server、zabbix\_agent，在ansible节点配置SSH免密通信，通过scp分别把公钥发送给zabbix\_server节点和zabbix\_agent节点。

在ansible节点将ssh zabbix\_server "cat /root/.ssh/known\_hosts"命令的返回结果提交到答题框。

**答案**

\[root@ansible \~]# ssh zabbix\_server "cat /root/.ssh/known\_hosts"

ansible,192.168.100.161 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBD9m+x7fx/zq3hgourWx2peUIRUd8D+LQ3xlkQPyd5Q0atuwm+OJUDS0sG70nuPf5/pUWnemUJ6w60yoTFUDZOU=

zabbix\_server,192.168.100.77 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBPAGqIhH86lCQn/oLxQ+m2VYovx1Nm4ZKx6RD1dmTHRBvxzlwlp5SIy4dzag/O47cl5cUJCRJM6gf/RT6I36LmA=

zabbix\_agent,192.168.100.165 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBDh+PuRlCpoCYJsEzYnhRXPBkmJwidgTvHc8ojryWiJJkosN3dPmo2kcEVE2GVcbhJ3EP9szt9AREeXIzfD1/4w=

**操作步骤**

\[root@ansible \~]# vim /etc/hosts

192.168.100.161 ansible

192.168.100.77 zabbix\_server

192.168.100.165 zabbix\_agent

\[root@ansible \~]# ssh-copy-id ansible

\[root@ansible \~]# ssh-copy-id zabbix\_server

\[root@ansible \~]# ssh-copy-id zabbix\_agent

\[root@ansible \~]# scp /root/.ssh/known\_hosts zabbix\_server:/root/.ssh/known\_hosts

\[root@ansible \~]# scp /root/.ssh/known\_hosts zabbix\_agent:/root/.ssh/known\_hosts

#### 配置主机清单（2分）

在ansible节点配置主机清单，在清单中分别创建server主机组和agent主机组，server主机组主机为zabbix\_server的IP地址，agent主机组主机为zabbix\_agent的IP地址。

在ansible节点将ansible agent -m ping命令的返回结果提交到答题框。

**答案**

\[root@ansible \~]# ansible agent -m ping

192.168.100.165 | SUCCESS => {

"ansible\_facts": {

"discovered\_interpreter\_python": "/usr/bin/python"

},

"changed": false,

"ping": "pong"

}

**操作步骤**

\[root@ansible \~]# vim /etc/ansible/hosts

\#添加

\[server]

192.168.100.77

\[agent]

192.168.100.165

#### 创建ansible工作环境（2分）

在ansible节点配置ftp服务,然后创建目录/opt/zabbix/files,在files目录下为zabbix\_server节点和zabbix\_agent节点编写ftp.repo文件，最后在zabbix目录下创建repo.yaml文件，文件实现的功能要求如下：

1\)把zabbix\_server节点和zabbix\_agent节点自带的repo文件删除。

2\)把zabbix\_server节点和zabbix\_agent节点下的/etc/yum.repos.d目录的权限设置为755

3\)把ftp.repo文件同时发送给zabbix\_server节点和zabbix\_agent节点。

将ansible-playbook /opt/zabbix/repo.yaml命令的返回结果提交到答题框。

**答案**

\[root@ansible zabbix]# ansible-playbook /opt/zabbix/repo.yaml

PLAY \[配置repo文件] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[192.168.100.77]

ok: \[192.168.100.165]

TASK \[删除自带的repo文件] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

changed: \[192.168.100.165]

TASK \[给目录设置权限] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.165]

changed: \[192.168.100.77]

TASK \[把ftp.repo复制给zabbix\_server和zabbix\_agent] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

changed: \[192.168.100.165]

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

192.168.100.165 : ok=4 changed=3 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

192.168.100.77 : ok=4 changed=3 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

**操作步骤**

\[root@ansible \~]# yum -y install vsftpd

\[root@ansible \~]# vim /etc/vsftpd/vsftpd.conf

\[root@ansible \~]# systemctl restart vsftpd

\[root@ansible \~]# systemctl enable vsftpd

\[root@ansible \~]# mkdir -p /opt/zabbix/files

\[root@ansible \~]# cd /opt/zabbix/

\[root@ansible zabbix]# vim files/ftp.repo

\[centos]

name=centos

baseurl=ftp://192.168.100.161/centos

gpgcheck=0

enabled=1

\[ansible]

name=ansible

baseurl=ftp://192.168.100.161/autoDeployment

gpgcheck=0

enabled=1

\[root@ansible zabbix]# vim repo.yaml

\---

\- name: 配置repo文件

hosts: all

tasks:

\- name: 删除自带的repo文件

file:

path: /etc/yum.repos.d/

state: absent

\- name: 给目录设置权限

file:

path: /etc/yum.repos.d

state: directory

mode: 755

\- name: 把ftp.repo复制给zabbix\_server和zabbix\_agent

copy:

src: files/ftp.repo

dest: /etc/yum.repos.d/

#### 安装nginx和php（2分）

在ansible节点/opt/zabbix目录创建nginx\_php.yaml文件并执行，文件实现的功能要求如下：

1\)实现在zabbix\_server节点安装nginx和php74（相关软件包：php74-php-fpm,php74-php-common,php74-php-cli,php74-php-gd, php74-php-ldap,php74-php-mbstring,php74-php-mysqlnd,php74-php-xml, php74-php-bcmath,php74-php）。

2\)开启nginx和php74服务，并设置为开机自启动。

在ansible节点将ansible server -m shell -a "systemctl status nginx php74-php-fpm ; nginx -v ; php74 -v"命令的返回结果提交到答题框。

**答案**

\[root@ansible zabbix]# ansible server -m shell -a "systemctl status nginx php74-php-fpm ; nginx -v ; php74 -v"

192.168.100.77 | CHANGED | rc=0 >>

● nginx.service - nginx - high performance web server

Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)

Active: active (running) since Thu 2024-06-20 09:07:13 UTC; 2min 57s ago

Docs: http://nginx.org/en/docs/

Main PID: 9115 (nginx)

CGroup: /system.slice/nginx.service

├─9115 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.con

├─9116 nginx: worker process

├─9117 nginx: worker process

├─9118 nginx: worker process

└─9119 nginx: worker process

Jun 20 09:07:13 zabbix\_server systemd\[1]: Starting nginx - high performance web server...

Jun 20 09:07:13 zabbix\_server systemd\[1]: Started nginx - high performance web server.

● php74-php-fpm.service - The PHP FastCGI Process Manager

Loaded: loaded (/usr/lib/systemd/system/php74-php-fpm.service; enabled; vendor preset: disabled)

Active: active (running) since Thu 2024-06-20 09:07:14 UTC; 2min 57s ago

Main PID: 9247 (php-fpm)

Status: "Processes active: 0, idle: 5, Requests: 0, slow: 0, Traffic: 0req/sec"

CGroup: /system.slice/php74-php-fpm.service

├─9247 php-fpm: master process (/etc/opt/remi/php74/php-fpm.conf

├─9248 php-fpm: pool www

├─9249 php-fpm: pool www

├─9250 php-fpm: pool www

├─9251 php-fpm: pool www

└─9252 php-fpm: pool www

Jun 20 09:07:14 zabbix\_server systemd\[1]: Starting The PHP FastCGI Process Manager...

Jun 20 09:07:14 zabbix\_server systemd\[1]: Started The PHP FastCGI Process Manager.

PHP 7.4.33 (cli) (built: Feb 14 2023 08:49:52) ( NTS )

Copyright (c) The PHP Group

Zend Engine v3.4.0, Copyright (c) Zend Technologiesnginx version: nginx/1.22.1

**操作步骤**

\[root@ansible zabbix]# vim nginx\_php.yaml

\---

\- name: 安装nginx和php74

hosts: server

tasks:

\- name: 安装nginx

yum:

name: nginx

state: present

\- name: 安装php74

yum:

name:

\- php74-php-fpm

\- php74-php-common

\- php74-php-cli

\- php74-php-gd

\- php74-php-ldap

\- php74-php-mbstring

\- php74-php-mysqlnd

\- php74-php-xml

\- php74-php-bcmath

\- php74-php

state: present

\- service:

name: nginx

state: started

enabled: yes

\- service:

name: php74-php-fpm

state: started

enabled: yes

\[root@ansible zabbix]# ansible-playbook nginx\_php.yaml

PLAY \[安装nginx和php74] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[192.168.100.77]

TASK \[安装nginx] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[安装php74] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[service] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[service] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

192.168.100.77 : ok=5 changed=4 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

#### 安装zabbix服务器端 （3分）

在ansible节点/opt/zabbix目录创建zabbix\_server.yaml文件并执行，文件实现的功能要求如下：

1\)在zabbix\_server节点安装zabbix的server端、agent端和web端。

2\)分别启动zabbix-server和zabbix-agent。

在ansible节点将ansible server -a "systemctl status zabbix-server zabbix-agent"命令的返回结果提交到答题框。

**答案**

\[root@ansible zabbix]# ansible server -a "systemctl status zabbix-server zabbix-agent"

192.168.100.77 | CHANGED | rc=0 >>

● zabbix-server-mysql.service - Zabbix Server with MySQL DB

Loaded: loaded (/usr/lib/systemd/system/zabbix-server-mysql.service; disabled; vendor preset: disabled)

Active: active (running) since Thu 2024-06-20 09:14:48 UTC; 26s ago

Main PID: 11351 (zabbix\_server)

CGroup: /system.slice/zabbix-server-mysql.service

└─11351 /usr/sbin/zabbix\_server -f

Jun 20 09:14:48 zabbix\_server systemd\[1]: Started Zabbix Server with MySQL DB.

● zabbix-agent.service - Zabbix Monitor Agent

Loaded: loaded (/usr/lib/systemd/system/zabbix-agent.service; disabled; vendor preset: disabled)

Active: active (running) since Thu 2024-06-20 09:14:48 UTC; 26s ago

Main PID: 11448 (zabbix\_agentd)

CGroup: /system.slice/zabbix-agent.service

├─11448 /usr/sbin/zabbix\_agentd -f

├─11449 /usr/sbin/zabbix\_agentd: collector \[idle 1 sec

├─11450 /usr/sbin/zabbix\_agentd: listener #1 \[waiting for connection

├─11451 /usr/sbin/zabbix\_agentd: listener #2 \[waiting for connection

├─11452 /usr/sbin/zabbix\_agentd: listener #3 \[waiting for connection

└─11453 /usr/sbin/zabbix\_agentd: active checks #1 \[idle 1 sec

Jun 20 09:14:48 zabbix\_server systemd\[1]: Started Zabbix Monitor Agent.

Jun 20 09:14:48 zabbix\_server zabbix\_agentd\[11448]: Starting Zabbix Agent \[Zabbix server]. Zabbix 6.0.13 (revision fdfa8cef9ce).

Jun 20 09:14:48 zabbix\_server zabbix\_agentd\[11448]: Press Ctrl+C to exit.

**操作步骤**

\[root@ansible zabbix]# vim zabbix\_server.yaml

\---

\- name: 安装server端、agent端和web端

hosts: server

tasks:

\- name: 安装zabbix6.0-server,zabbix6.0-agent和zabbix6.0-web

yum:

name:

\- zabbix6.0-server

\- zabbix6.0-agent

\- zabbix6.0-web

state: present

\- name: 启动zabbix-server和zabbix-agent

service:

name: "\{{item\}}"

state: started

loop:

\- zabbix-server

\- zabbix-agent

\[root@ansible zabbix]# ansible-playbook zabbix\_server.yaml

PLAY \[安装server端、agent端和web端] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[192.168.100.77]

TASK \[安装zabbix6.0-server,zabbix6.0-agent和zabbix6.0-web] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[启动zabbix-server和zabbix-agent] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77] => (item=zabbix-server)

changed: \[192.168.100.77] => (item=zabbix-agent)

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

192.168.100.77 : ok=3 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

#### 安装数据库（3分）

在ansible节点/opt/zabbix目录创建mariadb.yaml文件并执行，文件实现的功能要求如下：

1\)在zabbix\_server节点安装MariaDB-server。

2\)启动数据库服务并设置为开机自启动。

在ansible节点将ansible server -m shell -a "systemctl status mariadb | head -n 6"命令的返回结果提交到答题框。

**答案**

\[root@ansible zabbix]# ansible server -m shell -a "systemctl status mariadb | head -n 6"

192.168.100.77 | CHANGED | rc=0 >>

● mariadb.service - MariaDB 10.5.20 database server

Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; vendor preset: disabled)

Drop-In: /etc/systemd/system/mariadb.service.d

└─migrated-from-my.cnf-settings.conf

Active: active (running) since Thu 2024-06-20 09:21:21 UTC; 57s ago

Docs: man:mariadbd(8)

**操作步骤**

\[root@ansible zabbix]# vim mariadb.yaml

\---

\- name: 安装数据库mariadb

hosts: server

tasks:

\- name: 安装数据库mariadb

yum:

name: MariaDB-server

state: present

\- name: 启动数据库并设置为开机自启动

service:

name: mariadb

state: started

enabled: yes

\[root@ansible zabbix]# ansible-playbook mariadb.yaml

PLAY \[安装数据库mariadb] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[192.168.100.77]

TASK \[安装数据库mariadb] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[启动数据库并设置为开机自启动] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

192.168.100.77 : ok=3 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

#### 配置数据库（3分）

在ansible节点/opt/zabbix目录创建mariadb\_cfg.yaml文件并执行，文件实现的功能要求如下：

1.设置mysql登录密码，密码为password,默认登录账号为root。

2.创建数据库zabbix。

3.创建用户zabbix密码为password并授权zabbix用户拥有zabbix数据库的所有权限。

4.分别导入zabbix数据库架构及数据，对应的文件分别为schema.sql、images.sql和data.sql（文件顺便不能乱）。

在ansible节点将ansible server -m shell -a "mysql -uroot -ppassword -e 'use zabbix;select \* from users\_groups;show grants for 'zabbix'@'localhost';'"命令的返回结果提交到答题框。

**答案**

\[root@ansible zabbix]# ansible server -m shell -a "mysql -uroot -ppassword -e 'use zabbix;select \* from users\_groups;show grants for 'zabbix'@'localhost';'"

192.168.100.77 | CHANGED | rc=0 >>

id usrgrpid userid

4 7 1

2 8 2

3 9 2

Grants for zabbix@localhost

GRANT USAGE ON \*.\* TO \`zabbix\`@\`localhost\` IDENTIFIED BY PASSWORD '\*2470C0C06DEE42FD1618BB99005ADCA2EC9D1E19'

GRANT ALL PRIVILEGES ON \`zabbix\`.\* TO \`zabbix\`@\`localhost\`

**操作步骤**

\[root@ansible zabbix]# vim mariadb\_cfg.yaml

\---

\- name: 对数据库进行配置

hosts: server

tasks:

\- name: 设置登录密码

shell: mysqladmin -uroot password password

\- name: 创建数据库zabbix

shell: mysql -uroot -ppassword -e "create database zabbix character set utf8 collate utf8\_bin;"

ignore\_errors: yes

\- name: 创建用户zabbix

shell: mysql -uroot -ppassword -e "grant all privileges on zabbix.\* to zabbix@localhost identified by 'password';"

ignore\_errors: yes

\- name: 刷新配置

shell: mysql -uroot -ppassword -e "flush privileges;"

ignore\_errors: yes

\- name: 导入schema.sql

shell: mysql -uroot -ppassword zabbix < /usr/share/zabbix-mysql/schema.sql

ignore\_errors: yes

\- name: 导入images.sql

shell: mysql -uroot -ppassword zabbix < /usr/share/zabbix-mysql/images.sql

ignore\_errors: yes

\- name: 导入data.sql

shell: mysql -uroot -ppassword zabbix < /usr/share/zabbix-mysql/data.sql

ignore\_errors: yes

\- name: 重启数据库服务

service:

name: mariadb

state: restarted

enabled: yes

\[root@ansible zabbix]# ansible-playbook mariadb\_cfg.yaml

PLAY \[对数据库进行配置] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[192.168.100.77]

TASK \[设置登录密码] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[创建数据库zabbix] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[创建用户zabbix] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[刷新配置] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[导入schema.sql] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[导入images.sql] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[导入data.sql] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[重启数据库服务] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

192.168.100.77 : ok=9 changed=8 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

#### 编辑zabbix配置文件（4分）

在ansible节点/opt/zabbix目录分别创建zabbix\_server.conf.j2和zabbix\_agentd.conf.j2,然后编写zsa.yml文件并执行，文件实现的功能要求如下：

1.用template分别把j2文件分别发送给zabbix\_server节点的相应位置。

2.重启相应服务使配置生效。

在ansible节点将ansible server -m shell -a "cat /etc/zabbix\_server.conf | grep -v '^#\\|^$'"命令的返回结果提交到答题框。

**答案**

\[root@ansible zabbix]# ansible server -m shell -a "cat /etc/zabbix\_server.conf | grep -v '^#\\|^$'"

192.168.100.77 | CHANGED | rc=0 >>

LogFile=/var/log/zabbixsrv/zabbix\_server.log

LogFileSize=0

PidFile=/run/zabbixsrv/zabbix\_server.pid

DBName=zabbix

DBUser=zabbix

DBPassword=password

DBSocket=/var/lib/mysql/mysql.sock

Timeout=4

AlertScriptsPath=/var/lib/zabbixsrv/alertscripts

ExternalScripts=/var/lib/zabbixsrv/externalscripts

LogSlowQueries=3000

TmpDir=/var/lib/zabbixsrv/tmp

StatsAllowedIP=127.0.0.1

**操作步骤**

\[root@ansible zabbix]# vim vars.yaml

dbname: zabbix

dbuser: zabbix

dbpassword: password

server: 192.168.100.77

serveractive: 192.168.100.77

hostname: "\{{ansible\_default\_ipv4\['address']\}}"

\[root@ansible zabbix]# scp zabbix\_server:/etc/zabbix\_server.conf ./zabbix\_server.conf.j2

\[root@ansible zabbix]# scp zabbix\_server:/etc/zabbix\_agentd.conf ./zabbix\_agentd.conf.j2

\[root@ansible zabbix]# vim zabbix\_server.conf.j2

\#修改

DBName=\{{dbname\}}

DBUser=\{{dbuser\}}

DBPassword=\{{dbpassword\}}

\[root@ansible zabbix]# vim zabbix\_agentd.conf.j2

\#修改

Server=\{{server\}}

ServerActive=\{{serveractive\}}

Hostname=\{{hostname\}}

\[root@ansible zabbix]# vim zsa.yaml

\---

\- hosts: server

vars\_files: ./vars.yaml

tasks:

\- name: 复制zabbix\_server.conf.j2

template:

src: ./zabbix\_server.conf.j2

dest: /etc/zabbix\_server.conf

\- name: 复制zabbix\_agentd.conf.j2

template:

src: ./zabbix\_agentd.conf.j2

dest: /etc/zabbix\_agentd.conf

\- name: 重启zabbix-server服务和zabbix-agent服务

service:

name: "\{{item\}}"

state: restarted

loop:

\- zabbix-server

\- zabbix-agent

\[root@ansible zabbix]# ansible-playbook zsa.yaml

PLAY \[server] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[192.168.100.77]

TASK \[复制zabbix\_server.conf.j2] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[复制zabbix\_agentd.conf.j2] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[192.168.100.77]

TASK \[重启zabbix-server服务和zabbix-agent服务] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77] => (item=zabbix-server)

changed: \[192.168.100.77] => (item=zabbix-agent)

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

192.168.100.77 : ok=4 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

#### 编辑php配置文件（4分）

在ansible节点/opt/zabbix目录创建php.ini.j2,php.ini.j2配置要求最大POST数据限制为16M,程序执行时间限制为300，PHP页面接受数据所需最大时间限制为300，把时区设为Asia/Shanghai，然后编写php.yaml文件并执行，文件实现的功能要求如下：

1.用template把j2文件发送给zabbix\_server节点的相应位置。

2.重启php74服务。

在ansible节点将cat /opt/zabbix/php.yaml && ansible server -m shell -a "cat /etc/opt/remi/php74/php.ini | grep -v '^;'|grep max"命令的返回结果提交到答题框。

**答案**

\[root@ansible zabbix]# cat /opt/zabbix/php.yaml && ansible server -m shell -a "cat /etc/opt/remi/php74/php.ini | grep -v '^;'|grep max"

\---

\- hosts: server

vars\_files: ./vars.yaml

tasks:

\- template:

src: /opt/zabbix/php.ini.j2

dest: /etc/opt/remi/php74/php.ini

\- service:

name: php74-php-fpm

state: restarted

192.168.100.77 | CHANGED | rc=0 >>

max\_execution\_time = 300

max\_input\_time = 300

log\_errors\_max\_len = 1024

post\_max\_size = 16M

upload\_max\_filesize = 2M

max\_file\_uploads = 20

odbc.max\_persistent = -1

odbc.max\_links = -1

mysqli.max\_persistent = -1

mysqli.max\_links = -1

pgsql.max\_persistent = -1

pgsql.max\_links = -1

session.gc\_maxlifetime = 1440

ldap.max\_links = -1

**操作步骤**

\[root@ansible zabbix]# vim vars.yaml

dbname: zabbix

dbuser: zabbix

dbpassword: password

server: 192.168.100.77

serveractive: 192.168.100.77

hostname: "\{{ansible\_default\_ipv4\['address']\}}"

post\_max\_size: 16M

max\_execution\_time: 300

max\_input\_time: 300

datetimezone: Asia/Shanghai

\[root@ansible zabbix]# scp zabbix\_server:/etc/opt/remi/php74/php.ini ./php.ini.j2

\[root@ansible zabbix]# vim php.ini.j2

\#修改

post\_max\_size = \{{post\_max\_size\}}

max\_execution\_time = \{{max\_execution\_time\}}

max\_input\_time = \{{max\_input\_time\}}

date.timezone = \{{datetimezone\}}

\[root@ansible zabbix]# vim php.yaml

\---

\- hosts: server

vars\_files: ./vars.yaml

tasks:

\- template:

src: /opt/zabbix/php.ini.j2

dest: /etc/opt/remi/php74/php.ini

\- service:

name: php74-php-fpm

state: restarted

\[root@ansible zabbix]# ansible-playbook php.yaml

PLAY \[server] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[192.168.100.77]

TASK \[template] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[service] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

192.168.100.77 : ok=3 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

#### 配置www.conf（3分）

在ansible节点/opt/zabbix目录创建www.conf.j2(把用户和组都设置为nginx)，然后编写www.yaml文件并执行，文件实现的功能要求如下：

用template把j2文件发送给zabbix\_server节点的相应位置。

在ansible节点将cat /opt/zabbix/www.yaml && ansible server -m shell -a "cat /etc/php-fpm.d/www.conf | grep nginx"命令的返回结果提交到答题框。

**答案**

\[root@ansible zabbix]# cat /opt/zabbix/www.yaml && ansible server -m shell -a "cat /etc/php-fpm.d/www.conf | grep nginx"

\---

\- hosts: server

vars\_files: ./vars.yaml

tasks:

\- template:

src: /opt/zabbix/www.conf.j2

dest: /etc/php-fpm.d/www.conf

192.168.100.77 | CHANGED | rc=0 >>

user = nginx

group = nginx

**操作步骤**

\[root@ansible zabbix]# vim vars.yaml

dbname: zabbix

dbuser: zabbix

dbpassword: password

server: 192.168.100.77

serveractive: 192.168.100.77

hostname: "\{{ansible\_default\_ipv4\['address']\}}"

post\_max\_size: 16M

max\_execution\_time: 300

max\_input\_time: 300

datetimezone: Asia/Shanghai

user: nginx

group: nginx

\[root@ansible zabbix]# scp zabbix\_server:/etc/php-fpm.d/www.conf ./www.conf.j2

\[root@ansible zabbix]# vim www.conf.j2

\#修改

user = \{{user\}}

group = \{{group\}}

\[root@ansible zabbix]# vim www.yaml

\---

\- hosts: server

vars\_files: ./vars.yaml

tasks:

\- template:

src: /opt/zabbix/www.conf.j2

dest: /etc/php-fpm.d/www.conf

\[root@ansible zabbix]# ansible-playbook www.yaml

PLAY \[server] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[192.168.100.77]

TASK \[template] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

192.168.100.77 : ok=2 changed=1 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

#### 编辑nginx配置文件(3分)

在ansible节点/opt/zabbix目录创建default.conf.j2(使用80端口，其它参数自行修改)，然后编写default.yaml文件并执行，文件实现的功能要求如下：

用template把j2文件发送给zabbix\_server节点的相应位置。

在ansible节点将cat /opt/zabbix/default.yaml && ansible server -m shell -a "cat /etc/nginx/conf.d/default.conf | grep -Ev ^'( #|$|\\\[)'"命令的返回结果提交到答题框。

**答案**

\[root@ansible zabbix]# cat /opt/zabbix/default.yaml && ansible server -m shell -a "cat /etc/nginx/conf.d/default.conf | grep -Ev ^'( #|$|\\\[)'"

\---

\- hosts: server

vars\_files: ./vars.yaml

tasks:

\- template:

src: /opt/zabbix/default.conf.j2

dest: /etc/nginx/conf.d/default.conf

192.168.100.77 | CHANGED | rc=0 >>

server {

listen 80;

server\_name localhost;

location / {

root /usr/share/zabbix/;

index index.html index.htm;

}

error\_page 500 502 503 504 /50x.html;

location = /50x.html {

root /usr/share/nginx/html;

}

location \~ \\.php$ {

root /usr/share/zabbix/;

fastcgi\_pass 127.0.0.1:9000;

fastcgi\_index index.php;

fastcgi\_param SCRIPT\_FILENAME $document\_root$fastcgi\_script\_name;

include fastcgi\_params;

}

}

**操作步骤**

\[root@ansible zabbix]# vim vars.yaml

dbname: zabbix

dbuser: zabbix

dbpassword: password

server: 192.168.100.77

serveractive: 192.168.100.77

stname: "\{{ansible\_default\_ipv4\['address']\}}"

post\_max\_size: 16M

max\_execution\_time: 300

max\_input\_time: 300

datetimezone: Asia/Shanghai

user: nginx

group: nginx

listen: 80

root: /usr/share/zabbix/

indexphp: index.php

fastcgi\_index: index.php

SCRIPT\_FILENAME: $document\_root$fastcgi\_script\_name

\[root@ansible zabbix]# scp zabbix\_server:/etc/nginx/conf.d/default.conf ./default.conf.j2

\[root@ansible zabbix]# cat default.conf.j2 | grep -Ev ^'( #|$|\\\[)'

\#修改

server {

listen \{{listen\}};

server\_name localhost;

location / {

root \{{root\}};

index index.html index.htm;

}

error\_page 500 502 503 504 /50x.html;

location = /50x.html {

root /usr/share/nginx/html;

}

location \~ \\.php$ {

root \{{root\}};

fastcgi\_pass 127.0.0.1:9000;

fastcgi\_index \{{fastcgi\_index\}};

fastcgi\_param SCRIPT\_FILENAME \{{SCRIPT\_FILENAME\}};

include fastcgi\_params;

}

}

\[root@ansible zabbix]# vim default.yaml

\---

\- hosts: server

vars\_files: ./vars.yaml

tasks:

\- template:

src: /opt/zabbix/default.conf.j2

dest: /etc/nginx/conf.d/default.conf

\[root@ansible zabbix]# ansible-playbook default.yaml

PLAY \[server] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[192.168.100.77]

TASK \[template] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

192.168.100.77 : ok=2 changed=1 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

#### 配置zabbix.conf(3分)

在ansible节点/opt/zabbix目录创建zabbix.conf.j2(用户和组都设置为nginx)，然后编写zabbix.yaml文件并执行，文件实现的功能要求如下：

1.用template把j2文件发送给zabbix\_server节点的相应位置。

2.重启相关服务。

在浏览器中输入http://公网IP/setup.php即可看到zabbix 6.0界面。

在ansible节点将cat /opt/zabbix/zabbix.yaml && curl http://zabbix\_server/setup.php | head -n 10命令的返回结果提交到答题框。

**答案**

\[root@ansible zabbix]# cat /opt/zabbix/zabbix.yaml && curl http://zabbix\_server/setup.php | head -n 10

\---

\- hosts: server

vars\_files: ./vars.yaml

tasks:

\- template:

src: /opt/zabbix/zabbix.conf.j2

dest: /etc/php-fpm.d/zabbix.conf

\- service:

name: "\{{item\}}"

state: restarted

loop:

\- nginx

\- zabbix-server

\- zabbix-agent

\- php74-php-fpm

\- mariadb

% Total % Received % Xferd Average Speed Time Time Time Current

Dload Upload Total Spent Left Speed

100 4703 0 4703 0 \<!DOCTYPE html>0 --:--:-- --:--:-- --:--:-- 0

\<html lang="en">

0 \<head>

\<meta http-equiv="X-UA-Compatible" content="IE=Edge"/>

\<meta charset="utf-8" />

\<meta name="viewport" content="width=device-width, initial-scale=1">

1 \<meta name="Author" content="Zabbix SIA" />

8 \<title>Installation\</title>

5 \<link rel="icon" href="favicon.ico">

k \<link rel="apple-touch-icon-precomposed" sizes="76x76" href="assets/img/apple-touch-icon-76x76-precomposed.png">

0 --:--:-- --:--:-- --:--:-- 191k

(23) Failed writing body

**操作步骤**

\[root@ansible zabbix]# scp zabbix\_server:/etc/php-fpm.d/zabbix.conf ./zabbix.conf.j2

\[root@ansible zabbix]# vim zabbix.conf.j2

\#修改

\[zabbix]

user = \{{user\}}

group = \{{group\}}

\[root@ansible zabbix]# vim zabbix.yaml

\---

\- hosts: server

vars\_files: ./vars.yaml

tasks:

\- template:

src: /opt/zabbix/zabbix.conf.j2

dest: /etc/php-fpm.d/zabbix.conf

\- service:

name: "\{{item\}}"

state: restarted

loop:

\- nginx

\- zabbix-server

\- zabbix-agent

\- php74-php-fpm

\- mariadb

\[root@ansible zabbix]# ansible-playbook zabbix.yaml

PLAY \[server] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[192.168.100.77]

TASK \[template] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77]

TASK \[service] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.77] => (item=nginx)

changed: \[192.168.100.77] => (item=zabbix-server)

changed: \[192.168.100.77] => (item=zabbix-agent)

changed: \[192.168.100.77] => (item=php74-php-fpm)

changed: \[192.168.100.77] => (item=mariadb)

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

192.168.100.77 : ok=3 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

#### 编写playbook(5分)

在ansible节点/opt/zabbix目录编辑agent.yaml文件并执行，文件实现的功能要求如下：

1.把ftp.repo远程复制到zabbix\_agent节点相应位置。

2.在zabbix\_agent节点安装zabbix-agent服务。

3.用template把zabbix\_agentd.conf.j2文件发送给zabbix\_agent节点的相应位置。

4.重启zabbix-agent服务。

在ansilbe节点将cat agent.yaml && ansible agent -m shell -a "systemctl status zabbix-agent"命令的返回结果提交到答题框。

**答案**

\[root@ansible zabbix]# cat agent.yaml && ansible agent -m shell -a "systemctl status zabbix-agent"

\---

\- hosts: agent

vars\_files: ./vars.yaml

tasks:

\- name: 删除/etc/yum.repos.d目录(删除repo文件)

file:

path: /etc/yum.repos.d

state: absent

\- name: 创建/etc/yum.repos.d目录

file:

path: /etc/yum.repos.d

state: directory

mode: 755

\- name: 把ftp.repo文件复制到/etc/yum.repos.d目录

copy:

src: ./files/ftp.repo

dest: /etc/yum.repos.d/

\- name: 安装zabbix6.0-agent

yum:

name: zabbix6.0-agent

state: present

\- name: 复制zabbix\_agentd.conf.j2文件

template:

src: zabbix\_agentd.conf.j2

dest: /etc/zabbix\_agentd.conf

\- name: 启动zabbix-agent服务

service:

name: zabbix-agent

state: restarted

192.168.100.165 | CHANGED | rc=0 >>

● zabbix-agent.service - Zabbix Monitor Agent

Loaded: loaded (/usr/lib/systemd/system/zabbix-agent.service; disabled; vendor preset: disabled)

Active: active (running) since Thu 2024-06-20 14:24:57 UTC; 2h 6min ago

Main PID: 24391 (zabbix\_agentd)

CGroup: /system.slice/zabbix-agent.service

├─24391 /usr/sbin/zabbix\_agentd -f

├─24392 /usr/sbin/zabbix\_agentd: collector \[idle 1 sec

├─24393 /usr/sbin/zabbix\_agentd: listener #1 \[waiting for connection

├─24394 /usr/sbin/zabbix\_agentd: listener #2 \[waiting for connection

├─24395 /usr/sbin/zabbix\_agentd: listener #3 \[waiting for connection

└─24396 /usr/sbin/zabbix\_agentd: active checks #1 \[idle 1 sec

Jun 20 14:24:57 zabbix\_agent systemd\[1]: Started Zabbix Monitor Agent.

Jun 20 14:24:57 zabbix\_agent zabbix\_agentd\[24391]: Starting Zabbix Agent \[192.168.100.165]. Zabbix 6.0.13 (revision fdfa8cef9ce).

Jun 20 14:24:57 zabbix\_agent zabbix\_agentd\[24391]: Press Ctrl+C to exit.

**操作步骤**

\[root@ansible zabbix]# vim agent.yaml

\---

\- hosts: agent

vars\_files: ./vars.yaml

tasks:

\- name: 删除/etc/yum.repos.d目录(删除repo文件)

file:

path: /etc/yum.repos.d

state: absent

\- name: 创建/etc/yum.repos.d目录

file:

path: /etc/yum.repos.d

state: directory

mode: 755

\- name: 把ftp.repo文件复制到/etc/yum.repos.d目录

copy:

src: ./files/ftp.repo

dest: /etc/yum.repos.d/

\- name: 安装zabbix6.0-agent

yum:

name: zabbix6.0-agent

state: present

\- name: 复制zabbix\_agentd.conf.j2文件

template:

src: zabbix\_agentd.conf.j2

dest: /etc/zabbix\_agentd.conf

\- name: 启动zabbix-agent服务

service:

name: zabbix-agent

state: restarted

\[root@ansible zabbix]# ansible-playbook agent.yaml

PLAY \[agent] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[192.168.100.165]

TASK \[删除/etc/yum.repos.d目录(删除repo文件)] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.165]

TASK \[创建/etc/yum.repos.d目录] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.165]

TASK \[把ftp.repo文件复制到/etc/yum.repos.d目录] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.165]

TASK \[安装zabbix6.0-agent] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.165]

TASK \[复制zabbix\_agentd.conf.j2文件] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.165]

TASK \[启动zabbix-agent服务] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

changed: \[192.168.100.165]

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

192.168.100.165 : ok=7 changed=6 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0

### 任务2、部署prometheus监控mysqld服务

虚拟机与环境规划表3

| 设备名称  | 主机名     | 接口   | IP地址                                             | 角色                         |
| ----- | ------- | ---- | ------------------------------------------------ | -------------------------- |
| 云服务器1 | monitor | eth0 | <p>公网IP:********</p><p>私网IP:192.168.100.*/24</p> | Prometheus，grafana，ansible |
| 云服务器2 | slave1  | eth0 | <p>公网IP:********</p><p>私网IP:192.168.100.*/24</p> | agent                      |
| 云服务器3 | slave2  | eth0 | <p>公网IP:********</p><p>私网IP:192.168.100.*/24</p> | agent                      |

**说明：**

1\. 上表中的公网IP以自己云主机显示为准，每个人的公网IP不同。使用第三方软件远程连接云主机，使用公网IP连接。

2.华为云中云主机名字已命好，直接使用对应名字的云主机即可。

3.竞赛用到的软件包都在云主机/root下。

#### 十五、安装ansible并创建test用户（1分）

修改monitor节点主机名为ansible,slave1节点主机名为slave1,slave2节点主机名为slave2，配置各节点主机映射将IP地址映射为主机名，使用提供的软件包autoDeployment.tar在monitor节点配置yun源并安装ansible。为所有节点添加test用户，设置用户密码为000000，为test用户设置免密sudo，配置ssh免密登录，使monitor节点能够免密登录所有节点的test用户。

在monitor节点将ansible --version && ssh test@slave2 "id && hostnamectl && sudo -lU test"命令的返回结果提交到答题框。

**答案**

\[root@monitor \~]# ansible --version && ssh test@slave2 "id && hostnamectl && sudo -lU test"

ansible 2.9.27

config file = /etc/ansible/ansible.cfg

configured module search path = \[u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']

ansible python module location = /usr/lib/python2.7/site-packages/ansible

executable location = /usr/bin/ansible

python version = 2.7.5 (default, Oct 14 2020, 14:45:30) \[GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]

**操作步骤**

vi /etc/hosts

192.168.100.10 monitor

192.168.100.20 slave1

192.168.100.30 slave2

scp /etc/hosts slave1:/etc/hosts

scp /etc/hosts slave2:/etc/hosts

使用网络源进行安装ansible

wget -O /etc/yum.repos.d/epel-7.repo http://mirrors.aliyun.com/repo/epel-7.repo

yum -y install epel-release

yum install -y ansible

为所有节点添加test用户并设置用户密码为000000

useradd test && echo "000000" | passwd --stdin test

为test用户设置免密sudo

echo "test ALL=(ALL) NOPASSWD: ALL" >> visudo

配置ssh免密登录

\[root@monitor ansible]# sudo -u test ssh-keygen -t rsa -N "" -f /home/test/.ssh/id\_rsa

Generating public/private rsa key pair.

Your identification has been saved in /home/test/.ssh/id\_rsa.

Your public key has been saved in /home/test/.ssh/id\_rsa.pub.

The key fingerprint is:

SHA256:osykMK8rFyhxPU5x2E/tjzJilQEw8mix4x8Duz6agB4 test@monitor

The key's randomart image is:

\+---\[RSA 2048]----+

\| o o+.. . |

\| \*o.o o . |

\| \*..o o + |

|.o.++ + . |

|o+oo+...S o |

|++.B.+o.o . . |

|+E+.=. . o |

|+++ |

|\*=.. |

\+----\[SHA256]-----+

\[root@monitor ansible]# sudo -u test ssh-copy-id test@slave1

\[root@monitor ansible]# sudo -u test ssh-copy-id test@slave2

#### 十六、初始化ansible（1分）

在monitor节点创建/root/ansible目录作为ansible工作目录，在该目录内创建ansible.cfg文件并完成以下配置，清单文件位置为/root/ansible/inventory，登录用户为test，登录时不需要输入密码，设置并行主机数量为2。允许test用户免密提权到root。

在monitor节点将cat /root/ansible/ansible.cfg命令的返回结果提交到答题框。

**答案**

\[root@monitor ansible]# cat /root/ansible/ansible.cfg

\[defaults]

inventory = /root/ansible/inventory

remote\_user = test

ask\_pass = flase

forks = 2

\[privilege\_escalation]

become=True

become\_method=sudo

become\_user=root

become\_ask\_pass=False

**操作步骤**

\[root@monitor \~]# mkdir -p /root/ansible

\[root@monitor \~]# cd ansible

\[root@monitor ansible]# vim ansible.cfg

\[defaults]

inventory = /root/ansible/inventory

remote\_user = test

ask\_pass = flase

forks = 2

\[privilege\_escalation]

become=True

become\_method=sudo

become\_user=root

become\_ask\_pass=False

echo "test ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/test

#### 十七、编写ansible主机清单（0.5分）

在monitor节点ansible工作目录下配置ansible主机清单，在清单中创建master主机组和node主机组,master主机组内添加monitor主机,node主机组内添加slave1和slave2主机，主机清单中需使用主机名不使用ip。

在monitor节点将cd /root/ansible/ && ansible slave2 -m ping && ansible all -m shell -a "id && users" | sed s/\\"//g命令的返回结果提交到答题框。

**答案**

\[root@monitor ansible]# cd /root/ansible/ && ansible slave2 -m ping && ansible all -m shell -a "id && users" | se

d s/\\"//g

slave2 | SUCCESS => {

"ansible\_facts": {

"discovered\_interpreter\_python": "/usr/bin/python"

},

"changed": false,

"ping": "pong"

}

slave1 | CHANGED | rc=0 >>

uid=0(root) gid=0(root) groups=0(root)

root root root

slave2 | CHANGED | rc=0 >>

uid=0(root) gid=0(root) groups=0(root)

root root root

monitor | CHANGED | rc=0 >>

uid=0(root) gid=0(root) groups=0(root)

root root

**操作步骤**

\[root@monitor ansible]# vim inventory

\[master]

monitor

\[node]

slave1

slave2

#### 十八、初始化monitor节点（2分）

在monitor节点ansible工作目录下编写prometheus.yaml剧本文件控制master主机组完成以下内容（请使用ansible除shell外的对应模块，后续题目均使用提供的软件包Prometheus.tar.gz）：

1\)使用ansible对应模块将selinux的开机启动状态设置为disabled。

2\)使用ansible对应模块安装时间同步服务chrony，并将该服务的作用域设置为0.0.0.0/0，启动服务并设置开机自启动。

3\)使用ansible对应模块将monitor节点提供的prometheus-2.37.0.linux-amd64.tar.gz文件解压缩到目标主机的/usr/local/下。

4\)使用ansible对应模块将解压出的文件夹重命名为prometheus。

编写完成后将cd /root/ansible/ && ansible-playbook prometheus.yaml && cat prometheus.yaml命令的返回结果提交到答题框。

**答案**

\[root@monitor ansible]# cd /root/ansible/ && ansible-playbook prometheus.yaml && cat prometheus.yaml

PLAY \[master] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[monitor]

TASK \[禁用selinux] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*

ok: \[monitor]

TASK \[安装并配置时间同步] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*\*\*\*\*\*\*

ok: \[monitor]

TASK \[配置时间同步] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*\*\*\*

ok: \[monitor]

TASK \[启动并设置为开机自启动] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*\*\*\*\*\*\*\*\*

changed: \[monitor]

TASK \[解压prometheus] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*

changed: \[monitor]

TASK \[重命名] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*

changed: \[monitor]

TASK \[重启 chronyd 服务] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*\*

changed: \[monitor]

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

monitor : ok=8 changed=4 unreachable=0 failed=0 skipped=0 rescued=0 ignored=

0

\---

\- hosts: master

tasks:

\- name: 禁用selinux

template:

src: /root/ansible/config.j2

dest: /etc/selinux/config

\- name: 安装并配置时间同步

yum:

name: chrony

state: present

\- name: 配置时间同步

template:

src: /root/ansible/chrony.conf.j2

dest: /etc/chrony.conf

\- name: 启动并设置为开机自启动

service:

name: chronyd

state: restarted

enabled: yes

\- name: 解压prometheus

unarchive:

src: /root/prometheus-2.37.0.linux-amd64.tar.gz

dest: /usr/local

remote\_src: yes

\- name: 重命名

shell: mv /usr/local/prometheus-2.37.0.linux-amd64 /usr/local/prometheus

\- name: 重启 chronyd 服务

service:

name: chronyd

state: restarted

**操作步骤**

\[root@monitor ansible]# vim prometheus.yaml

\---

\- hosts: master

tasks:

\- name: 禁用selinux

template:

src: /root/ansible/config.j2

dest: /etc/selinux/config

\- name: 安装并配置时间同步

yum:

name: chrony

state: present

\- name: 配置时间同步

template:

src: /root/ansible/chrony.conf.j2

dest: /etc/chrony.conf

\- name: 启动并设置为开机自启动

service:

name: chronyd

state: restarted

enabled: yes

\- name: 解压prometheus

unarchive:

src: /root/prometheus-2.37.0.linux-amd64.tar.gz

dest: /usr/local

remote\_src: yes

\- name: 重命名

shell: mv /usr/local/prometheus-2.37.0.linux-amd64 /usr/local/prometheus

\- name: 重启 chronyd 服务

service:

name: chronyd

state: restarted

\[root@monitor selinux]# cp /etc/selinux/config /root/ansible/config.j2

\[root@monitor ansible]# vim config.j2

SELINUX=disabled

\[root@monitor ansible]# cp /etc/chrony.conf /root/ansible/chrony.conf.j2

\[root@monitor ansible]# vim chrony.conf.j2

allow 0.0.0.0/0

\[root@monitor ansible]# ansible-playbook prometheus.yaml

#### 十九、启动node主机组服务（2分）

在monitor节点ansible工作目录下编写node\_exporter.yaml剧本文件控制node主机组完成以下内容（请使用ansible除shell外的对应模块）：

1\)使用ansible对应模块将node\_exporter-1.3.1.linux-amd64.tar.gz解压到目标主机组的/usr/local/下。

2\)使用ansible对应模块将解压出的文件夹重命名为node\_exporter。

3\)使用script模块将node\_exporter服务后台启动。

编写完成后在monitor节点将cd /root/ansible/ && ansible-playbook /root/ansible/node\_exporter.yaml && cat /root/ansible/node\_exporter.yaml && sleep 10 && ansible node -m shell -a "ps -aux | grep node"命令的返回结果提交到答题框。

**答案**

\[root@monitor ansible]# cd /root/ansible/ && ansible-playbook /root/ansible/node\_exporter.yaml && cat /root/ansib

le/node\_exporter.yaml && sleep 10 && ansible node -m shell -a "ps -aux | grep node"

PLAY \[node] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[slave1]

ok: \[slave2]

TASK \[解压node\_exporter] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*

ok: \[slave1]

ok: \[slave2]

TASK \[重命名] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*

changed: \[slave1]

changed: \[slave2]

TASK \[授权] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*

ok: \[slave2]

ok: \[slave1]

TASK \[后台启动服务] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*\*\*\*

changed: \[slave1]

changed: \[slave2]

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

slave1 : ok=5 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=

0

slave2 : ok=5 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=

0

\---

\- hosts: node

tasks:

\- name: 解压node\_exporter

unarchive:

src: /root/node\_exporter-1.3.1.linux-amd64.tar.gz

dest: /usr/local

remote\_src: yes

\- name: 重命名

shell: mv /usr/local/node\_exporter-1.3.1.linux-amd64 /usr/local/node\_exporter

\- name: 授权

file:

path: /root/start\_node\_exporter.sh

mode: '0755'

\- name: 后台启动服务

script: /root/start\_node\_exporter.sh

slave1 | CHANGED | rc=0 >>

root 6085 0.0 0.0 113280 1192 pts/1 S+ 15:19 0:00 /bin/sh -c ps -aux | grep node

root 6087 0.0 0.0 112808 944 pts/1 S+ 15:19 0:00 grep node

slave2 | CHANGED | rc=0 >>

root 6077 0.0 0.3 715956 12972 ? Sl 15:19 0:00 /usr/local/node\_exporter/node\_exporter

root 6145 0.0 0.0 113280 1188 pts/0 S+ 15:19 0:00 /bin/sh -c ps -aux | grep node

root 6147 0.0 0.0 112808 944 pts/0 S+ 15:19 0:00 grep node

**操作步骤**

\[root@monitor ansible]# vim start\_node\_exporter.sh

\#!/bin/bash

nohup /usr/local/node\_exporter/node\_exporter > /var/log/node\_exporter.log 2>&1 &

\[root@monitor ansible]# scp start\_node\_exporter.sh slave1:/root/

start\_node\_exporter.sh

\[root@monitor ansible]# scp start\_node\_exporter.sh slave2:/root/

start\_node\_exporter.sh

\[root@monitor ansible]# vim node\_exporter.yaml

\---

\- hosts: node

tasks:

\- name: 解压node\_exporter

unarchive:

src: /root/node\_exporter-1.3.1.linux-amd64.tar.gz

dest: /usr/local

remote\_src: yes

\- name: 重命名

shell: mv /usr/local/node\_exporter-1.3.1.linux-amd64 /usr/local/node\_exporter

\- name: 授权

file:

path: /root/start\_node\_exporter.sh

mode: '0755'

\- name: 后台启动服务

script: /root/start\_node\_exporter.sh

\[root@monitor ansible]# ansible-playbook node\_exporter.yaml

#### 二十、启动monitor主机组服务（3.5分）

在monitor节点ansible工作目录下prometheus.yml配置文件，将需要监控的所有节点信息添加到该文件中；编写prometheus.service启动文件，实现prometheus以服务的形式启动。完成后请编写start\_prometheus.yaml剧本文件控制master主机组完成以下内容（请使用ansible除shell外的对应模块）：

1\)使用ansible对应模块将prometheus.yml文件传输到目标节点prometheus配置目录下

2\)使用ansible对应模块将prometheus.service文件传输到目标节点服务配置文件目录下并启动prometheus。

3\)使用复制模块将grafana-8.1.2-1.x86\_64.rpm包发送到被控节点的/mnt/目录下，然后使用yum模块将该软件包安装，安装完成后启动grafana服务并设置开机自启动。

4\)使用浏览器登录prometheus查看prometheus是否成功监控所有node节点。

在monitor节点将cd /root/ansible/ && cat /root/ansible/prometheus.yml | grep : && ansible-playbook /root/ansible/start\_prometheus.yaml && cat /root/ansible/start\_prometheus.yaml && sleep 20 && systemctl status prometheus | head -n 7 && curl -L http://localhost:9090/api/v1/targets?state=active | sed s/\\"//g命令的返回结果提交到答题框。

**答案**

\[root@monitor ansible]# cd /root/ansible/ && cat /root/ansible/prometheus.yml | grep : && ansible-playbook /root/

ansible/start\_prometheus.yaml && cat /root/ansible/start\_prometheus.yaml && sleep 20 && systemctl status promethe

us | head -n 7 && curl -L http://localhost:9090/api/v1/targets?state=active | sed s/\\"//g

globel**:**

scrape\_interval**:** 15s

scrape\_configs**:**

\- job\_name**:** 'master'

static\_configs**:**

\- tsrgets**:** \['slave1**:**9100','slave2**:**9100']

PLAY \[master] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

TASK \[Gathering Facts] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

ok: \[monitor]

TASK \[复制文件] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*\*

ok: \[monitor]

TASK \[复制启动文件] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*\*\*\*

ok: \[monitor]

TASK \[启动prometheus服务] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*\*

changed: \[monitor]

TASK \[复制rpm包] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*

ok: \[monitor]

TASK \[安装grafana服务] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*\*

ok: \[monitor]

TASK \[启动grafana服务] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*\*

changed: \[monitor]

PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

monitor : ok=7 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=

0

\---

\- hosts: master

tasks:

\- name: 复制文件

copy:

src: /root/ansible/prometheus.yml

dest: /etc/prometheus/prometheus.yml

\- name: 复制启动文件

copy:

src: /root/ansible/prometheus.service

dest: /etc/systemd/system/prometheus.service

\- name: 启动prometheus服务

service:

name: prometheus

state: restarted

enabled: yes

\- name: 复制rpm包

copy:

src: /root/grafana-8.1.2-1.x86\_64.rpm

dest: /mnt/grafana-8.1.2-1.x86\_64.rpm

\- name: 安装grafana服务

yum:

name: /mnt/grafana-8.1.2-1.x86\_64.rpm

state: present

\- name: 启动grafana服务

service:

name: grafana-server

state: restarted

enabled: yes

● prometheus.service - Prometheus

Loaded: loaded (/etc/systemd/system/prometheus.service; enabled; vendor preset: disabled)

Active: active (running) since Sun 2024-09-29 09:18:09 CST; 2min 17s ago

Main PID: 2104 (prometheus)

CGroup: /system.slice/prometheus.service

└─2104 /usr/local/bin/prometheus/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/

lib/prometheus

% Total % Received % Xferd Average Speed Time Time Time Current

Dload Upload Total Spent Left Speed

100 1249 100 1249 0 0 210k 0 --:--:-- --:--:-- --:--:-- 243k

{status:success,data:{activeTargets:\[{discoveredLabels:{\_\_address\_\_:slave1:9100,\_\_metrics\_path\_\_:/metrics,\_\_scheme\_\_:http,\_\_s

crape\_interval\_\_:15s,\_\_scrape\_timeout\_\_:10s,job:prometheus},labels:{instance:slave1:9100,job:prometheus},scrapePool:prometheu

s,scrapeUrl:http://slave1:9100/metrics,globalUrl:http://slave1:9100/metrics,lastError:Get \http://slave1:9100/metrics\\: dial

tcp 192.168.100.20:9100: connect: connection refused,lastScrape:2024-09-29T09:20:15.381941014+08:00,lastScrapeDuration:0.0025

38257,health:down,scrapeInterval:15s,scrapeTimeout:10s},{discoveredLabels:{\_\_address\_\_:slave2:9100,\_\_metrics\_path\_\_:/metrics,

\_\_scheme\_\_:http,\_\_scrape\_interval\_\_:15s,\_\_scrape\_timeout\_\_:10s,job:prometheus},labels:{instance:slave2:9100,job:prometheus},s

crapePool:prometheus,scrapeUrl:http://slave2:9100/metrics,globalUrl:http://slave2:9100/metrics,lastError:Get \http://slave2:9

100/metrics\\: dial tcp 192.168.100.30:9100: connect: connection refused,lastScrape:2024-09-29T09:20:19.814547416+08:00,lastSc

rapeDuration:0.000867366,health:down,scrapeInterval:15s,scrapeTimeout:10s}],droppedTargets:\[]\}}

**操作步骤**

\[root@monitor ansible]# tar -zxvf /root/prometheus-2.37.0.linux-amd64.tar.gz /usr/local/bin/

\[root@monitor ansible]# scp /usr/local/bin/prometheus/prometheus.yml /root/ansible/prometheus.yml

vim cat prometheus.yml

\# my global config

global:

scrape\_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.

evaluation\_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

\# scrape\_timeout is set to the global default (10s).

\# Alertmanager configuration

alerting:

alertmanagers:

\- static\_configs:

\- targets:

\# - alertmanager:9093

\# Load rules once and periodically evaluate them according to the global 'evaluation\_interval'.

rule\_files:

\# - "first\_rules.yml"

\# - "second\_rules.yml"

\# A scrape configuration containing exactly one endpoint to scrape:

\# Here it's Prometheus itself.

scrape\_configs:

\# The job name is added as a label \`job=\` to any timeseries scraped from this config.

\- job\_name: "prometheus"

\# metrics\_path defaults to '/metrics'

\# scheme defaults to 'http'.

static\_configs:

\- targets: \['slave1:9100','slave2:9100']

\[root@monitor ansible]# cat prometheus.service

\[Unit]

Description=Prometheus

After=network.target

\[Service]

User=root

ExecStart=/usr/local/bin/prometheus/prometheus \\

\--config.file=/etc/prometheus/prometheus.yml \\

\--storage.tsdb.path=/var/lib/prometheus \\

\--web.external-url=http://192.168.100.10:9090\\

\[Install]

WantedBy=multi-user.target

\[root@monitor ansible]# cat start\_prometheus.yaml

\---

\- hosts: master

tasks:

\- name: 复制文件

copy:

src: /root/ansible/prometheus.yml

dest: /etc/prometheus/prometheus.yml

\- name: 复制启动文件

copy:

src: /root/ansible/prometheus.service

dest: /etc/systemd/system/prometheus.service

\- name: 启动prometheus服务

service:

name: prometheus

state: restarted

enabled: yes

\- name: 复制rpm包

copy:

src: /root/grafana-8.1.2-1.x86\_64.rpm

dest: /mnt/grafana-8.1.2-1.x86\_64.rpm

\- name: 安装grafana服务

yum:

name: /mnt/grafana-8.1.2-1.x86\_64.rpm

state: present

\- name: 启动grafana服务

service:

name: grafana-server

state: restarted

enabled: yes

\[root@monitor ansible]# ansible-playbook start\_prometheus.yaml

#### 二十一、使用prometheus监控mysqld服务（3.5分）

配置slave1节点yum使用monitor节点ftp源，将提供的mysqld\_exporter-0.14.0.linux-amd64.tar.gz发送到slave1节点/usr/local/目录下解压并重命名为mysqld\_exporter，使用yum安装mariadb服务启动并设为开机自启动。进入mariadb数据库中创建mysqld\_monitor用户并授权，然后创建mariadb配置文件，内容为数据库用户名密码。后台启动mysqld\_exporter组件确保9104端口启动。回到prometheus节点修改prometheus.yml文件并添加mysql被监控信息。重启prometheus，随后web界面刷新并查看mysqld被控信息。

在monitor节点将cat /usr/local/prometheus/prometheus.yml | grep : && ssh slave1 "ps -aux | grep mysql" && curl -L http://localhost:9090/api/v1/targets?state=active | sed s/\\"//g命令的返回结果提交到答题框。

**答案**

\[root@monitor ansible]# cat /usr/local/prometheus/prometheus.yml | grep : && ssh slave1 "ps -aux | grep mysql" && curl -L htt

p://localhost:9090/api/v1/targets?state=active | sed s/\\"//g

global**:**

scrape\_interval**:** 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.

evaluation\_interval**:** 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

alerting**:**

alertmanagers**:**

\- static\_configs**:**

\- targets**:**

\# - alertmanager**:**9093

rule\_files**:**

\# A scrape configuration containing exactly one endpoint to scrape**:**

scrape\_configs**:**

\- job\_name**:** "prometheus"

static\_configs**:**

\- targets**:** \['slave1**:**9100','slave2**:**9100']

mysql 1630 0.0 0.0 113412 1592 ? Ss 17:53 0:00 /bin/sh /usr/bin/mysqld\_safe --basedir=/usr

mysql 1795 0.1 2.3 968920 90364 ? Sl 17:53 0:01 /usr/libexec/mysqld --basedir=/usr --datadir=/var/lib/mysql

\--plugin-dir=/usr/lib64/mysql/plugin --log-error=/var/log/mariadb/mariadb.log --pid-file=/var/run/mariadb/mariadb.pid --sock

et=/var/lib/mysql/mysql.sock

root 1940 0.0 0.0 113280 1552 ? Ss 18:14 0:00 bash -c ps -aux | grep mysql

root 1946 0.0 0.0 112808 948 ? R 18:14 0:00 grep mysql

% Total % Received % Xferd Average Speed Time Time Time Current

Dload Upload Total Spent Left Speed

100 1154 100 1154 0 0 166k 0 --:--:-- --:--:-- --:--:-- 187k

{status:success,data:{activeTargets:\[{discoveredLabels:{\_\_address\_\_:slave1:9100,\_\_metrics\_path\_\_:/metrics,\_\_scheme\_\_:http,\_\_s

crape\_interval\_\_:15s,\_\_scrape\_timeout\_\_:10s,job:prometheus},labels:{instance:slave1:9100,job:prometheus},scrapePool:prometheu

s,scrapeUrl:http://slave1:9100/metrics,globalUrl:http://slave1:9100/metrics,lastError:,lastScrape:2024-09-29T10:14:15.3593974

34+08:00,lastScrapeDuration:0.013666011,health:up,scrapeInterval:15s,scrapeTimeout:10s},{discoveredLabels:{\_\_address\_\_:slave2

:9100,\_\_metrics\_path\_\_:/metrics,\_\_scheme\_\_:http,\_\_scrape\_interval\_\_:15s,\_\_scrape\_timeout\_\_:10s,job:prometheus},labels:{instan

ce:slave2:9100,job:prometheus},scrapePool:prometheus,scrapeUrl:http://slave2:9100/metrics,globalUrl:http://slave2:9100/metric

s,lastError:Get \http://slave2:9100/metrics\\: dial tcp 192.168.100.30:9100: connect: connection refused,lastScrape:2024-09-29

T10:14:19.814569397+08:00,lastScrapeDuration:0.001126103,health:down,scrapeInterval:15s,scrapeTimeout:10s}],droppedTargets:\[]

\}}

**操作步骤**

\[root@slave1 \~]# tar -zxvf node\_exporter-1.3.1.linux-amd64.tar.gz /usr/local/

\[root@slave1 local]# yum install -y mariadb\*

\[root@slave1 local]# sudo systemctl start mariadb

\[root@slave1 local]# sudo systemctl enable mariadb

\[root@slave1 local]# mysql -uroot

create user 'mysqld\_monitor'@'localhost' identified by '000000';

grant process,replication client on \*.\* to 'mysqld\_monitor'@'localhost';

flush privileges;

MariaDB \[(none)]> quit

\[root@slave1 local]# vim /etc/my.cnf

\[mysqld]

datadir=/var/lib/mysql

socket=/var/lib/mysql/mysql.sock

\# Disabling symbolic-links is recommended to prevent assorted security risks

symbolic-links=0

\# Settings user and group are ignored when systemd is used.

\# If you need to run mysqld under a different user or group,

\# customize your systemd unit file for mariadb according to the

\# instructions in http://fedoraproject.org/wiki/Systemd

\[mysqld\_safe]

log-error=/var/log/mariadb/mariadb.log

pid-file=/var/run/mariadb/mariadb.pid

\#

\# include all files from the config directory

\#

!includedir /etc/my.cnf.d

\[client]

user=mysqld\_monitor

password=000000

\[root@slave1 local]# cd /usr/local/node\_exporter

\[root@slave1 node\_exporter]# ./node\_exporter &

检查服务是否启动

\[root@slave1 \~]# ps aux | grep node\_exporter

root 1882 0.3 0.5 717620 20208 pts/0 Sl 18:05 0:01 ./**node\_exporter**

root 1906 0.0 0.0 112808 972 pts/1 S+ 18:11 0:00 grep --color=auto **node\_exporter**

\[root@slave1 \~]# netstat -tuln | grep 9100

tcp6 0 0 :::**9100** :::\* LISTEN

#### 二十二、安装alertmanager报警组件（3.5分）

在monitor节点将提供的alertmanager-0.21.0.linux-amd64.tar.gz解压到/usr/local/目录下并重命名为alertmanager。创建alertmanager.service启动文件，实现alertmanager以服务的形式启动，然后启动alertmanager查看9093端口。在prometheus.yml配置文件中添加alertmanager信息并重新启动prometheus服务，在slave1节点上停止node\_exporter服务。到web界面中查看警报管理器状态是否正常和slave1节点node\_exporter状态是否异常。

在monitor节点将cat /usr/local/prometheus/prometheus.yml | grep : && systemctl status alertmanager | head -n 7 && curl -L http://localhost:9093/api/v2/status | sed s/\\"//g && curl -L http://localhost:9090/api/v1/targets?state=active | sed s/\\"//g命令的返回结果提交到答题框。

**答案**

\[root@monitor ansible]# cat /usr/local/prometheus/prometheus.yml | grep : && systemctl status alertmanager | head -n 7 && cur

l -L http://localhost:9093/api/v2/status | sed s/\\"//g && curl -L http://localhost:9090/api/v1/targets?state=active | sed s/\\

"//g

global**:**

scrape\_interval**:** 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.

evaluation\_interval**:** 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

alerting**:**

alertmanagers**:**

\- static\_configs**:**

\- targets**:**

\# - alertmanager**:**9093

rule\_files**:**

\# A scrape configuration containing exactly one endpoint to scrape**:**

scrape\_configs**:**

\- job\_name**:** "prometheus"

static\_configs**:**

\- targets**:** \['slave1**:**9100','slave2**:**9100']

● alertmanager.service - Alertmanager

Loaded: loaded (/etc/systemd/system/alertmanager.service; enabled; vendor preset: disabled)

Active: active (running) since Sun 2024-09-29 10:30:10 CST; 2min 20s ago

Main PID: 5817 (alertmanager)

CGroup: /system.slice/alertmanager.service

└─5817 /usr/local/alertmanager/alertmanager --config.file=/usr/local/alertmanager/alertmanager.yml --storage.path=

/usr/local/alertmanager/data

% Total % Received % Xferd Average Speed Time Time Time Current

Dload Upload Total Spent Left Speed

100 1186 100 1186 0 0 152k 0 --:--:-- --:--:-- --:--:-- 165k

{cluster:{name:01J8XSTC9YXXEW2H80Z9R8M8TA,peers:\[{address:192.168.100.10:9094,name:01J8XSTC9YXXEW2H80Z9R8M8TA}],status:ready}

,config:{original:global:\n resolve\_timeout: 5m\n http\_config: {}\n smtp\_hello: localhost\n smtp\_require\_tls: true\n pag

erduty\_url: https://events.pagerduty.com/v2/enqueue\n opsgenie\_api\_url: https://api.opsgenie.com/\n wechat\_api\_url: https:/

/qyapi.weixin.qq.com/cgi-bin/\n victorops\_api\_url: https://alert.victorops.com/integrations/generic/20131114/alert/\nroute:\\

n receiver: web.hook\n group\_by:\n - alertname\n group\_wait: 10s\n group\_interval: 10s\n repeat\_interval: 1h\ninhibit\_r

ules:\n- source\_match:\n severity: critical\n target\_match:\n severity: warning\n equal:\n - alertname\n - dev\n -

instance\nreceivers:\n- name: web.hook\n webhook\_configs:\n - send\_resolved: true\n http\_config: {}\n url: http://12

7.0.0.1:5001/\n max\_alerts: 0\ntemplates: \[]\n},uptime:2024-09-29T10:30:10.495+08:00,versionInfo:{branch:HEAD,buildDate:20

200617-08:54:02,buildUser:root@dee35927357f,goVersion:go1.14.4,revision:4c6c03ebfe21009c546e4d1e9b92c371d67c021d,version:0.21

.0\}}

% Total % Received % Xferd Average Speed Time Time Time Current

Dload Upload Total Spent Left Speed

100 1153 100 1153 0 0 174k 0 --:--:-- --:--:-- --:--:-- 187k

{status:success,data:{activeTargets:\[{discoveredLabels:{\_\_address\_\_:slave1:9100,\_\_metrics\_path\_\_:/metrics,\_\_scheme\_\_:http,\_\_s

crape\_interval\_\_:15s,\_\_scrape\_timeout\_\_:10s,job:prometheus},labels:{instance:slave1:9100,job:prometheus},scrapePool:prometheu

s,scrapeUrl:http://slave1:9100/metrics,globalUrl:http://slave1:9100/metrics,lastError:,lastScrape:2024-09-29T10:32:30.3621434

26+08:00,lastScrapeDuration:0.021593482,health:up,scrapeInterval:15s,scrapeTimeout:10s},{discoveredLabels:{\_\_address\_\_:slave2

:9100,\_\_metrics\_path\_\_:/metrics,\_\_scheme\_\_:http,\_\_scrape\_interval\_\_:15s,\_\_scrape\_timeout\_\_:10s,job:prometheus},labels:{instan

ce:slave2:9100,job:prometheus},scrapePool:prometheus,scrapeUrl:http://slave2:9100/metrics,globalUrl:http://slave2:9100/metric

s,lastError:Get \http://slave2:9100/metrics\\: dial tcp 192.168.100.30:9100: connect: connection refused,lastScrape:2024-09-29

T10:32:19.81419848+08:00,lastScrapeDuration:0.001793577,health:down,scrapeInterval:15s,scrapeTimeout:10s}],droppedTargets:\[]}

}

**操作步骤**

\[root@monitor \~]# tar -zxvf alertmanager-0.21.0.linux-amd64.tar.gz

\[root@monitor \~]# mv alertmanager-0.21.0.linux-amd64 /usr/local/alertmanager

\[root@monitor ansible]# vim /etc/systemd/system/alertmanager.service

\[Unit]

Description=Alertmanager

Wants=network-online.target

After=network-online.target

\[Service]

Type=simple

ExecStart=/usr/local/alertmanager/alertmanager \\

\--config.file=/usr/local/alertmanager/alertmanager.yml \\

\--storage.path=/usr/local/alertmanager/data

Restart=on-failure

\[Install]

WantedBy=multi-user.target

\[root@monitor ansible]# sudo systemctl daemon-reload

\[root@monitor ansible]# sudo systemctl start alertmanager

\[root@monitor ansible]# sudo systemctl enable alertmanager

\[root@monitor ansible]# vim prometheus.yml

\# my global config

global:

scrape\_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.

evaluation\_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

\# scrape\_timeout is set to the global default (10s).

\# Alertmanager configuration

alerting:

alertmanagers:

\- static\_configs:

\- targets:

\- alertmanager:9093

\# Load rules once and periodically evaluate them according to the global 'evaluation\_interval'.

rule\_files:

\# - "first\_rules.yml"

\# - "second\_rules.yml"

\# A scrape configuration containing exactly one endpoint to scrape:

\# Here it's Prometheus itself.

scrape\_configs:

\# The job name is added as a label \`job=\` to any timeseries scraped from this config.

\- job\_name: "prometheus"

\# metrics\_path defaults to '/metrics'

\# scheme defaults to 'http'.

static\_configs:

\- targets: \['slave1:9100','slave2:9100']

\- job\_name: 'node\_exporter'

static\_configs:

\- targets: \['slave1:9104']

\[root@monitor ansible]#

\[root@monitor ansible]# sudo systemctl restart prometheus

查看服务是否运行

\[root@monitor ansible]# ss -tuln | grep 9093

tcp LISTEN 0 128 \[::]:**9093** \[::]:\*

#### 二十三、alertmanager告警规则编写（3.5分）

在monitor节点的/usr/local/prometheus/路径下编写node\_rules.yml告警文件并加入prometheus配置文件中，请根据以下信息仿照模板编写：

1\)内存大于50%报警规则，告警级别：warning。

2\)cpu资源利用率大于75%报警规则，告警级别：warning。

3\)主机磁盘每秒读取数据>50MB%报警规则，告警级别：warning。

4\)节点服务异常报警规则，告警级别：critical

在monitor节点重启prometheus服务和alertmanager服务，验证结果正确后将cat /usr/local/prometheus/node\_rules.yml && cat /usr/local/prometheus/prometheus.yml | grep - && curl -L http://localhost:9090/api/v1/rules?type=alert | sed s/\\"//g命令的返回结果提交到答题框。

**答案**

\[root@monitor ansible]# cat /usr/local/prometheus/node\_rules.yml && cat /usr/local/prometheus/prometheus.yml | grep - && curl

\-L http://localhost:9090/api/v1/rules?type=alert | sed s/\\"//g

groups:

\- name: node\_rules

rules:

\- alert: HighMemoryUsage

expr: (node\_memory\_Active\_bytes / node\_memory\_MemTotal\_bytes) \* 100 > 50

for: 5m

labels:

severity: warning

annotations:

summary: "内存使用率超过50%"

description: "内存使用率 \{{ $value \}}% 大于 50% (实例 \{{ $labels.instance \}})"

\- alert: HighCPUUsage

expr: (100 - (avg by(instance) (irate(node\_cpu\_seconds\_total{mode="idle"}\[5m])) \* 100)) > 75

for: 5m

labels:

severity: warning

annotations:

summary: "CPU使用率超过75%"

description: "CPU使用率 \{{ $value \}}% 大于 75% (实例 \{{ $labels.instance \}})"

\- alert: HighDiskRead

expr: irate(node\_disk\_read\_bytes\_total\[5m]) > (50 \* 1024 \* 1024)

for: 5m

labels:

severity: warning

annotations:

summary: "磁盘读取速率超过50MB/s"

description: "磁盘读取速率 \{{ $value \}} 超过 50MB/s (实例 \{{ $labels.instance \}})"

\- alert: NodeDown

expr: up == 0

for: 2m

labels:

severity: critical

annotations:

summary: "节点服务异常"

description: "节点 \{{ $labels.instance \}} 服务异常"

**-** static\_configs:

**-** targets:

\# **-** alertmanager:9093

\# **-** "first\_rules.yml"

\# **-** "second\_rules.yml"

**-** "usr/local/prometheus/node\_rules.yml"

**-** job\_name: "prometheus"

**-** targets: \['slave1:9100','slave2:9100']

% Total % Received % Xferd Average Speed Time Time Time Current

Dload Upload Total Spent Left Speed

100 41 100 41 0 0 7278 0 --:--:-- --:--:-- --:--:-- 8200

**操作步骤**

\[root@monitor ansible]# vim /usr/local/prometheus/node\_rules.yml

groups:

\- name: node\_rules

rules:

\- alert: HighMemoryUsage

expr: (node\_memory\_Active\_bytes / node\_memory\_MemTotal\_bytes) \* 100 > 50

for: 5m

labels:

severity: warning

annotations:

summary: "内存使用率超过50%"

description: "内存使用率 \{{ $value \}}% 大于 50% (实例 \{{ $labels.instance \}})"

\- alert: HighCPUUsage

expr: (100 - (avg by(instance) (irate(node\_cpu\_seconds\_total{mode="idle"}\[5m])) \* 100)) > 75

for: 5m

labels:

severity: warning

annotations:

summary: "CPU使用率超过75%"

description: "CPU使用率 \{{ $value \}}% 大于 75% (实例 \{{ $labels.instance \}})"

\- alert: HighDiskRead

expr: irate(node\_disk\_read\_bytes\_total\[5m]) > (50 \* 1024 \* 1024)

for: 5m

labels:

severity: warning

annotations:

summary: "磁盘读取速率超过50MB/s"

description: "磁盘读取速率 \{{ $value \}} 超过 50MB/s (实例 \{{ $labels.instance \}})"

\- alert: NodeDown

expr: up == 0

for: 2m

labels:

severity: critical

annotations:

summary: "节点服务异常"

description: "节点 \{{ $labels.instance \}} 服务异常"

\[root@monitor ansible]# vim /usr/local/prometheus/prometheus.yml

\# my global config

global:

scrape\_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.

evaluation\_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

\# scrape\_timeout is set to the global default (10s).

\# Alertmanager configuration

alerting:

alertmanagers:

\- static\_configs:

\- targets:

\# - alertmanager:9093

\# Load rules once and periodically evaluate them according to the global 'evaluation\_interval'.

rule\_files:

\# - "first\_rules.yml"

\# - "second\_rules.yml"

\- "usr/local/prometheus/node\_rules.yml"

\# A scrape configuration containing exactly one endpoint to scrape:

\# Here it's Prometheus itself.

scrape\_configs:

\# The job name is added as a label \`job=\` to any timeseries scraped from this config.

\- job\_name: "prometheus"

\# metrics\_path defaults to '/metrics'

\# scheme defaults to 'http'.

static\_configs:

\- targets: \['slave1:9100','slave2:9100']

\[root@monitor ansible]# systemctl restart prometheus
