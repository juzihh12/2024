# OpeStack Y版OVS网络+Ceph后端集成项目

## OpeStack-Y版OVS网络+Ceph后端集成项目

## 一. OpenStack部署

### 1. 环境准备

| 主机名        | IP                    | 磁盘                | CPU | memory |
| ---------- | --------------------- | ----------------- | --- | ------ |
| controller | 网卡1：10.0.0.10，网卡2：不配置 | sda：100G          | 2C  | 6G     |
| compute01  | 网卡1：10.0.0.11，网卡2：不配置 | sda：100G ，sdb：20G | 2C  | 4G     |
| compute02  | 网卡1：10.0.0.12，网卡2：不配置 | sda：100G ，sdb：20G | 2C  | 4G     |

| 操作系统        | 虚拟化工具    |
| ----------- | -------- |
| Ubuntu22.04 | VMware15 |

### 2. 配置离线环境

```
# 解压
tar zxvf openstackyoga.tar.gz -C /opt/

# 备份文件
cp /etc/apt/sources.list{,.bak}

# 配置离线源
cat > /etc/apt/sources.list << EOF
deb [trusted=yes] file:// /opt/openstackyoga/debs/
EOF

# 清空缓存
apt clean all

# 加载源
apt update
```

### 3. 环境准备

#### 3.1 配置网络

* controller节点

```
cat > /etc/netplan/00-installer-config.yaml << EOF
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses: [10.0.0.10/24]
      routes:
        - to: default
          via: 10.0.0.254
      nameservers:
        addresses: [114.114.114.114]
    ens38:
      dhcp4: false
  version: 2
EOF

# 生效网络
netplan apply
```

* compute01节点

```
cat > /etc/netplan/00-installer-config.yaml << EOF
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses: [10.0.0.11/24]
      routes:
        - to: default
          via: 10.0.0.254
      nameservers:
        addresses: [114.114.114.114]
    ens38:
      dhcp4: false
  version: 2
EOF

# 生效网络
netplan apply
```

* compute02节点

```
cat > /etc/netplan/00-installer-config.yaml << EOF
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses: [10.0.0.12/24]
      routes:
        - to: default
          via: 10.0.0.254
      nameservers:
        addresses: [114.114.114.114]
    ens38:
      dhcp4: false
  version: 2
EOF

# 生效网络
netplan apply
```

#### 3.2 配置主机名并配置解析

* controller节点更改主机名

```
hostnamectl set-hostname controller

# 切换窗口
bash
```

* compute01节点更改主机名

```
hostnamectl set-hostname compute01

# 切换窗口
bash
```

* compute02节点更改主机名

```
hostnamectl set-hostname compute02

# 切换窗口
bash
```

* 所有节点配置hosts解析

```
cat >> /etc/hosts << EOF
10.0.0.10 controller
10.0.0.11 compute01
10.0.0.12 compute02
EOF
```

#### 3.3 时间调整

* 所有节点

```
# 开启可配置服务
timedatectl set-ntp true

# 调整时区为上海
timedatectl set-timezone Asia/Shanghai

# 将系统时间同步到硬件时间
hwclock --systohc
```

* 控制节点

```
# 安装服务
apt install -y chrony

# 配置文件
vim /etc/chrony/chrony.conf
20 server controller iburst maxsources 2
61 allow all
63 local stratum 10

# 重启服务
systemctl restart chronyd
```

* 计算节点

```
# 安装服务
apt install -y chrony

# 配置文件
vim /etc/chrony/chrony.conf
20 pool controller iburst maxsources 4

# 重启服务
systemctl restart chronyd
```

#### 3.4 安装openstack客户端

* controller节点

```
apt install -y python3-openstackclient
```

#### 3.5 安装部署MariaDB

* controller节点

```
apt install -y mariadb-server python3-pymysql
```

* 配置mariadb配置文件

```
cat > /etc/mysql/mariadb.conf.d/99-openstack.cnf << EOF
[mysqld]
bind-address = 0.0.0.0

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOF
```

* 重启根据配置文件启动

```
service mysql restart
```

* 初始化配置数据库

```
mysql_secure_installation
输入数据库密码：回车
可以在没有适当授权的情况下登录到MariaDB root用户，当前已收到保护：n
设置root用户密码：n
删除匿名用户：y
不允许远程root登录：n
删除测试数据库：y
重新加载数据库：y
```

#### 3.6 安装部署RabbitMQ

* 控制节点操作
* controller节点安装服务

```
apt install -y rabbitmq-server
```

* 创建openstack用户
  * 用户名为：openstack
  * 密码：000000

```
rabbitmqctl add_user openstack 000000
```

* 允许openstack用户进行配置、写入和读取访问

```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

#### 3.7 安装部署Memcache

* 控制节点操作
* controller节点安装服务

```
apt install -y memcached python3-memcache
```

* 配置监听地址

```
vim /etc/memcached.conf
35 -l 0.0.0.0
```

* 重启服务

```
service memcached restart
```

### 4. 部署配置keystone

* 控制节点操作
* 创建数据库与用户给予keystone使用

```
# 创建数据库
CREATE DATABASE keystone;

# 创建用户
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystoneang';
```

* controller节点安装服务

```
apt install -y keystone
```

* 配置keystone文件

```
# 备份配置文件
cp /etc/keystone/keystone.conf{,.bak}

# 过滤覆盖文件
grep -Ev "^$|#" /etc/keystone/keystone.conf.bak > /etc/keystone/keystone.conf

vim /etc/keystone/keystone.conf
[DEFAULT]
log_dir = /var/log/keystone
[application_credential]
[assignment]
[auth]
[cache]
[catalog]
[cors]
[credential]
[database]
connection = mysql+pymysql://keystone:keystoneang@controller/keystone
[domain_config]
[endpoint_filter]
[endpoint_policy]
[eventlet_server]
[extra_headers]
Distribution = Ubuntu
[federation]
[fernet_receipts]
[fernet_tokens]
[healthcheck]
[identity]
[identity_mapping]
[jwt_tokens]
[ldap]
[memcache]
[oauth1]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[policy]
[profiler]
[receipt]
[resource]
[revoke]
[role]
[saml]
[security_compliance]
[shadow_users]
[token]
provider = fernet
[tokenless_auth]
[totp]
[trust]
[unified_limit]
[wsgi]
```

* 填充数据库

```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

* 调用用户和组的密钥库
  * 这些选项是为了允许在另一个操作系统用户/组下运行密钥库

```
# 用户
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone

# 组
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

* 在Queens发布之前，keystone需要在两个单独的端口上运行，以容纳Identity v2 API，后者通常在端口35357上运行单独的仅限管理员的服务。随着v2 API的删除，keystones可以在所有接口的同一端口上运行5000

```
keystone-manage bootstrap --bootstrap-password 000000 --bootstrap-admin-url http://controller:5000/v3/ --bootstrap-internal-url http://controller:5000/v3/ --bootstrap-public-url http://controller:5000/v3/ --bootstrap-region-id RegionOne
```

* 编辑/etc/apache2/apache2.conf文件并配置ServerName选项以引用控制器节点

```
echo "ServerName controller" >> /etc/apache2/apache2.conf
```

* 重新启动Apache服务生效配置

```
service apache2 restart
```

* 配置OpenStack认证环境变量

```
cat > /etc/keystone/admin-openrc.sh << EOF
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=000000
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF
```

* 加载环境变量

```
source /etc/keystone/admin-openrc.sh
```

* 创建服务项目，后期组件将使用这个项目

```
openstack project create --domain default --description "Service Project" service
```

* 验证

```
openstack token issue
```

### 5. 部署配置glance镜像

* 控制节点操作
* 创建数据库与用户给予glance使用

```
# 创建数据库
CREATE DATABASE glance;

# 创建用户
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glanceang';
```

* 创建glance浏览用户

```
openstack user create --domain default --password glance glance
```

* 将管理员角色添加到浏览用户和服务项目

```
openstack role add --project service --user glance admin
```

* 创建浏览服务实体

```
openstack service create --name glance --description "OpenStack Image" image
```

* 创建镜像服务API端点

```
openstack endpoint create --region RegionOne image public http://controller:9292

openstack endpoint create --region RegionOne image internal http://controller:9292

openstack endpoint create --region RegionOne image admin http://controller:9292
```

* 安装glance镜像服务

```
apt install -y glance
```

* 配置glance配置文件

```
# 备份配置文件
cp /etc/glance/glance-api.conf{,.bak}

# 过滤覆盖配置文件
grep -Ev "^$|#" /etc/glance/glance-api.conf.bak > /etc/glance/glance-api.conf

# 配置项信息
vim /etc/glance/glance-api.conf
[DEFAULT]
[barbican]
[barbican_service_user]
[cinder]
[cors]
[database]
connection = mysql+pymysql://glance:glanceang@controller/glance
[file]
[glance.store.http.store]
[glance.store.rbd.store]
[glance.store.s3.store]
[glance.store.swift.store]
[glance.store.vmware_datastore.store]
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
[healthcheck]
[image_format]
disk_formats = ami,ari,aki,vhd,vhdx,vmdk,raw,qcow2,vdi,iso,ploop.root-tar
[key_manager]
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = glance
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[paste_deploy]
flavor = keystone
[profiler]
[store_type_location_strategy]
[task]
[taskflow_executor]
[vault]
[wsgi]
```

* 填充数据库

```
su -s /bin/sh -c "glance-manage db_sync" glance
```

* 重启glance服务生效配置

```
service glance-api restart
```

* 上传镜像验证

```
# 下载镜像
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img

# 上传镜像命令
glance image-create --name "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --visibility=public

# 查看镜像运行状态
root@controller:~# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 12a404ea-5751-41c6-a319-8f63de543cd8 | cirros | active |
+--------------------------------------+--------+--------+
```

### 6. 部署配置placement元数据

* 作用：placement服务跟踪每个供应商的库存和使用情况。例如，在一个计算节点创建一个实例的可消费资源如计算节点的资源提供者的CPU和内存，磁盘从外部共享存储池资源提供商和IP地址从外部IP资源提供者。
* 创建数据库与用户给予placement使用

```
# 创建数据库
CREATE DATABASE placement;

# 创建用户
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'placementang';
```

* 创建服务用户

```
openstack user create --domain default --password placement placement
```

* 将Placement用户添加到具有管理员角色的服务项目中

```
openstack role add --project service --user placement admin
```

* 在服务目录中创建Placement API条目

```
openstack service create --name placement --description "Placement API" placement
```

* 创建Placement API服务端点

```
openstack endpoint create --region RegionOne placement public http://controller:8778

openstack endpoint create --region RegionOne placement internal http://controller:8778

openstack endpoint create --region RegionOne placement admin http://controller:8778
```

* 安装placement服务

```
apt install -y placement-api
```

* 配置placement文件

```
# 备份配置文件
cp /etc/placement/placement.conf{,.bak}

# 过滤覆盖文件
grep -Ev "^$|#" /etc/placement/placement.conf.bak > /etc/placement/placement.conf

# 配置文件
vim /etc/placement/placement.conf
[DEFAULT]
[api]
auth_strategy = keystone
[cors]
[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = placement
[oslo_middleware]
[oslo_policy]
[placement]
[placement_database]
connection = mysql+pymysql://placement:placementang@controller/placement
[profiler]
```

* 填充数据库

```
su -s /bin/sh -c "placement-manage db sync" placement
```

* 重启apache加载placement配置

```
service apache2 restart
```

* 验证

```
root@controller:~# placement-status upgrade check
+-------------------------------------------+
| Upgrade Check Results                     |
+-------------------------------------------+
| Check: Missing Root Provider IDs          |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Incomplete Consumers               |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Policy File JSON to YAML Migration |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
```

### 7. 部署配置nova计算服务

#### 7.1 控制节点配置

* 创建数据库与用户给予nova使用

```
# 存放nova交互等数据
CREATE DATABASE nova_api;

# 存放nova资源等数据
CREATE DATABASE nova;

# 存放nova等元数据
CREATE DATABASE nova_cell0;

# 创建管理nova_api库的用户
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'novaang';

# 创建管理nova库的用户
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'novaang';

# 创建管理nova_cell0库的用户
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'novaang';
```

* 创建nova用户

```
openstack user create --domain default --password nova nova
```

* 将管理员角色添加到nova用户

```
openstack role add --project service --user nova admin
```

* 创建nova服务实体

```
openstack service create --name nova --description "OpenStack Compute" compute
```

* 创建计算API服务端点

```
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1

openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1

openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
```

* 安装服务

```
apt install -y nova-api nova-conductor nova-novncproxy nova-scheduler
```

* 配置nova文件

```
# 备份配置文件
cp /etc/nova/nova.conf{,.bak}

# 过滤提取文件
grep -Ev "^$|#" /etc/nova/nova.conf.bak > /etc/nova/nova.conf

# 配置结果
vim /etc/nova/nova.conf
[DEFAULT]
log_dir = /var/log/nova
lock_path = /var/lock/nova
state_path = /var/lib/nova
transport_url = rabbit://openstack:000000@controller:5672/
my_ip = 10.0.0.10
[api]
auth_strategy = keystone
[api_database]
connection = mysql+pymysql://nova:novaang@controller/nova_api
[barbican]
[barbican_service_user]
[cache]
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[cyborg]
[database]
connection = mysql+pymysql://nova:novaang@controller/nova
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://controller:9292
[guestfs]
[healthcheck]
[hyperv]
[image_cache]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova
[libvirt]
[metrics]
[mks]
[neutron]
[notifications]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[pci]
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement
[powervm]
[privsep]
[profiler]
[quota]
[rdp]
[remote_debug]
[scheduler]
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip
[workarounds]
[wsgi]
[zvm]
[cells]
enable = False
[os_region_name]
openstack =
```

* 填充nova\_api数据库

```
su -s /bin/sh -c "nova-manage api_db sync" nova
```

* 注册cell0数据库

```
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

* 创建cell1单元格

```
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```

* 填充nova数据库

```
su -s /bin/sh -c "nova-manage db sync" nova
```

* 验证nova、cell0和cell1是否正确注册

```
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

* 重启相关nova服务加载配置文件

```
# 处理api服务
service nova-api restart
# 处理资源调度服务
service nova-scheduler restart
# 处理数据库服务
service nova-conductor restart
# 处理vnc远程窗口服务
service nova-novncproxy restart
```

#### 7.2 计算节点配置

* compute01节点
* 安装nova-compute服务

```
apt install -y nova-compute
```

* 配置nova文件

```
# 备份配置文件
cp /etc/nova/nova.conf{,.bak}

# 过滤覆盖配置文件
grep -Ev "^$|#" /etc/nova/nova.conf.bak > /etc/nova/nova.conf

# 完整配置
vim /etc/nova/nova.conf
[DEFAULT]
log_dir = /var/log/nova
lock_path = /var/lock/nova
state_path = /var/lib/nova
transport_url = rabbit://openstack:000000@controller
my_ip = 10.0.0.11
[api]
auth_strategy = keystone
[api_database]
[barbican]
[barbican_service_user]
[cache]
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[cyborg]
[database]
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://controller:9292
[guestfs]
[healthcheck]
[hyperv]
[image_cache]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova
[libvirt]
[metrics]
[mks]
[neutron]
[notifications]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[pci]
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement
[powervm]
[privsep]
[profiler]
[quota]
[rdp]
[remote_debug]
[scheduler]
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://10.0.0.10:6080/vnc_auto.html
[workarounds]
[wsgi]
[zvm]
[cells]
enable = False
[os_region_name]
openstack =
```

* 检测是否支持硬件加速
  * 如果结果返回0，需要配置如下

```
# 确定计算节点是否支持虚拟机的硬件加速
egrep -c '(vmx|svm)' /proc/cpuinfo

# 如果结果返回 “0” ，那么需要配置如下
vim /etc/nova/nova-compute.conf
[libvirt]
virt_type = qemu
```

* 重启服务生效nova配置

```
service nova-compute restart
```

* compute02节点
* 安装nova-compute服务

```
apt install -y nova-compute
```

* 配置nova文件

```
# 备份配置文件
cp /etc/nova/nova.conf{,.bak}

# 过滤覆盖配置文件
grep -Ev "^$|#" /etc/nova/nova.conf.bak > /etc/nova/nova.conf

# 完整配置
vim /etc/nova/nova.conf
[DEFAULT]
log_dir = /var/log/nova
lock_path = /var/lock/nova
state_path = /var/lib/nova
transport_url = rabbit://openstack:000000@controller
my_ip = 10.0.0.12
[api]
auth_strategy = keystone
[api_database]
[barbican]
[barbican_service_user]
[cache]
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[cyborg]
[database]
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://controller:9292
[guestfs]
[healthcheck]
[hyperv]
[image_cache]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova
[libvirt]
[metrics]
[mks]
[neutron]
[notifications]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[pci]
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement
[powervm]
[privsep]
[profiler]
[quota]
[rdp]
[remote_debug]
[scheduler]
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://10.0.0.10:6080/vnc_auto.html
[workarounds]
[wsgi]
[zvm]
[cells]
enable = False
[os_region_name]
openstack =
```

* 检测是否支持硬件加速
  * 如果结果返回0，需要配置如下

```
# 确定计算节点是否支持虚拟机的硬件加速
egrep -c '(vmx|svm)' /proc/cpuinfo

# 如果结果返回 “0” ，那么需要配置如下
vim /etc/nova/nova-compute.conf
[libvirt]
virt_type = qemu
```

* 重启服务生效nova配置

```
service nova-compute restart
```

#### 7.3 配置主机发现

* 控制节点节点
* 查看有那些可用的计算节点

```
openstack compute service list --service nova-compute
```

* 发现计算主机

```
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

* 配置每5分钟主机发现一次

```
vim /etc/nova/nova.conf
'''
[scheduler]
discover_hosts_in_cells_interval = 300
'''
```

* 重启生效配置

```
service nova-api restart
```

* 校验nova服务

```
root@controller:~# openstack compute service list
+--------------------------------------+----------------+------------+----------+---------+-------+----------------------------+
| ID                                   | Binary         | Host       | Zone     | Status  | State | Updated At                 |
+--------------------------------------+----------------+------------+----------+---------+-------+----------------------------+
| 68178099-13c5-4464-9a55-71ea0dd30bf5 | nova-scheduler | controller | internal | enabled | up    | 2022-09-29T05:45:49.000000 |
| bd2a33be-1457-41c1-8ce8-3d4a8cb43551 | nova-conductor | controller | internal | enabled | up    | 2022-09-29T05:45:49.000000 |
| 98b4584d-f9bf-4c10-9fd8-331899ebf70b | nova-compute   | compute01  | nova     | enabled | up    | 2022-09-29T05:45:53.000000 |
| f809da57-8999-4ba4-8a32-5b60991f8838 | nova-compute   | compute02  | nova     | enabled | up    | 2022-09-29T05:45:56.000000 |
+--------------------------------------+----------------+------------+----------+---------+-------+----------------------------+
```

### 8. 配置基于OVS的Neutron网络服务

#### 8.1 控制节点配置

* 创建数据库与用给予neutron使用

```
# 创建数据库
CREATE DATABASE neutron;

# 创建用户
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'neutronang';
```

* 创建neutron用户

```
openstack user create --domain default --password neutron neutron
```

* 向neutron用户添加管理员角色

```
openstack role add --project service --user neutron admin
```

* 创建neutron实体

```
openstack service create --name neutron --description "OpenStack Networking" network
```

* 创建neutron的api端点

```
openstack endpoint create --region RegionOne network public http://controller:9696

openstack endpoint create --region RegionOne network internal http://controller:9696

openstack endpoint create --region RegionOne network admin http://controller:9696
```

* 配置内核转发

```
cat >> /etc/sysctl.conf << EOF
# 用于控制系统是否开启对数据包源地址的校验，关闭
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
# 开启二层转发设备
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
```

* 加载模块
  * 作用：桥接流量转发到iptables链

```
modprobe br_netfilter
```

* 生效内核配置

```
sysctl -p
```

* 安装ovs服务

```
apt install -y neutron-server neutron-plugin-ml2  neutron-l3-agent neutron-dhcp-agent  neutron-metadata-agent neutron-openvswitch-agent
```

* 配置neutron.conf文件
  * 用于提供neutron主体服务

```
# 备份配置文件
cp /etc/neutron/neutron.conf{,.bak}

# 过滤提取配置文件
grep -Ev "^$|#" /etc/neutron/neutron.conf.bak > /etc/neutron/neutron.conf

# 完整配置
vim /etc/neutron/neutron.conf
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
auth_strategy = keystone
state_path = /var/lib/neutron
dhcp_agent_notification = true
allow_overlapping_ips = true
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
transport_url = rabbit://openstack:000000@controller
[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"
[cache]
[cors]
[database]
connection = mysql+pymysql://neutron:neutronang@controller/neutron
[healthcheck]
[ironic]
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron
[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = nova
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[placement]
[privsep]
[quotas]
[ssl]
```

* 配置ml2\_conf.ini文件
  * 用户提供二层网络插件服务

```
# 备份配置文件
cp  /etc/neutron/plugins/ml2/ml2_conf.ini{,.bak}

# 过滤覆盖文件
grep -Ev "^$|#" /etc/neutron/plugins/ml2/ml2_conf.ini.bak > /etc/neutron/plugins/ml2/ml2_conf.ini

# 完整配置
vim /etc/neutron/plugins/ml2/ml2_conf.ini
[DEFAULT]
[ml2]
type_drivers = flat,vlan,vxlan,gre
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
[ml2_type_flat]
flat_networks = physnet1
[ml2_type_geneve]
[ml2_type_gre]
[ml2_type_vlan]
[ml2_type_vxlan]
vni_ranges = 1:1000
[ovs_driver]
[securitygroup]
enable_ipset = true
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
[sriov_driver]
```

* 配置openvswitch\_agent.ini文件
  * 提供ovs代理服务

```
# 备份文件
cp /etc/neutron/plugins/ml2/openvswitch_agent.ini{,.bak}

# 过滤覆盖文件
grep -Ev "^$|#" /etc/neutron/plugins/ml2/openvswitch_agent.ini.bak > /etc/neutron/plugins/ml2/openvswitch_agent.ini

# 完整配置
vim /etc/neutron/plugins/ml2/openvswitch_agent.ini
[DEFAULT]
[agent]
l2_population = True
tunnel_types = vxlan
prevent_arp_spoofing = True
[dhcp]
[network_log]
[ovs]
local_ip = 10.0.0.10
bridge_mappings = physnet1:br-ens38
[securitygroup]
```

* 配置l3\_agent.ini文件
  * 提供三层网络服务

```
# 备份文件
cp /etc/neutron/l3_agent.ini{,.bak}

# 过滤覆盖文件
grep -Ev "^$|#" /etc/neutron/l3_agent.ini.bak > /etc/neutron/l3_agent.ini

# 完整配置
vim /etc/neutron/l3_agent.ini
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
external_network_bridge =
[agent]
[network_log]
[ovs]
```

* 配置dhcp\_agent文件
  * 提供dhcp动态网络服务

```
# 备份文件
cp /etc/neutron/dhcp_agent.ini{,.bak}

# 过滤覆盖文件
grep -Ev "^$|#" /etc/neutron/dhcp_agent.ini.bak > /etc/neutron/dhcp_agent.ini

# 完整配置
vim /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
[agent]
[ovs]
```

* 配置metadata\_agent.ini文件
  * 提供元数据服务
  * 元数据什么？
    * 用来支持如指示存储位置、历史数据、资源查找、文件记录等功能。元数据算是一种电子式目录，为了达到编制目录的目的，必须在描述并收藏数据的内容或特色，进而达成协助数据检索的目的。

```
# 备份文件
cp  /etc/neutron/metadata_agent.ini{,.bak}

# 过滤覆盖文件
grep -Ev "^$|#" /etc/neutron/metadata_agent.ini.bak > /etc/neutron/metadata_agent.ini

# 完整配置
vim /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = ws
[agent]
[cache]
```

* 配置nova文件
  * 主要识别neutron配置，从而能调用网络

```
vim /etc/nova/nova.conf
'''
[default]
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSlnterfaceDriver

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
service_metadata_proxy = true
metadata_proxy_shared_secret = ws
'''
```

* 填充数据库

```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

* 重启nova-api服务生效neutron配置

```
service nova-api restart
```

* 新建一个外部网络桥接

```
ovs-vsctl add-br br-ens38
```

* 将外部网络桥接映射到网卡
  * 这里绑定第二张网卡，属于业务网卡

```
ovs-vsctl add-port br-ens38 ens38
```

* 重启neutron相关服务生效配置

```
# 提供neutron服务
service neutron-server restart
# 提供ovs服务
service neutron-openvswitch-agent restart
# 提供地址动态服务
service neutron-dhcp-agent restart
# 提供元数据服务
service neutron-metadata-agent restart
# 提供三层网络服务
service neutron-l3-agent restart
```

#### 8.2 计算节点配置

* compute01节点
* 配置内核转发

```
cat >> /etc/sysctl.conf << EOF
# 用于控制系统是否开启对数据包源地址的校验，关闭
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
# 开启二层转发设备
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
```

* 加载模块
  * 作用：桥接流量转发到iptables链

```
modprobe br_netfilter
```

* 生效内核配置

```
sysctl -p
```

* 安装neutron-ovs服务

```
apt install -y neutron-openvswitch-agent
```

* 配置neutron文件
  * 提供neutron主体服务

```
# 备份文件
cp /etc/neutron/neutron.conf{,.bak}

# 过滤提取文件
grep -Ev "^$|#" /etc/neutron/neutron.conf.bak > /etc/neutron/neutron.conf

# 完整配置
vim /etc/neutron/neutron.conf
[DEFAULT]
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
state_path = /var/lib/neutron
allow_overlapping_ips = true
transport_url = rabbit://openstack:000000@controller
[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"
[cache]
[cors]
[database]
[healthcheck]
[ironic]
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron
[nova]
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[placement]
[privsep]
[quotas]
[ssl]
```

* 配置openvswitch\_agent.ini文件
  * 提供ovs网络服务

```
# 备份文件
cp /etc/neutron/plugins/ml2/openvswitch_agent.ini{,.bak}

# 过滤提取文件
grep -Ev "^$|#" /etc/neutron/plugins/ml2/openvswitch_agent.ini.bak > /etc/neutron/plugins/ml2/openvswitch_agent.ini

# 完整配置
vim /etc/neutron/plugins/ml2/openvswitch_agent.ini
[DEFAULT]
[agent]
l2_population = True
tunnel_types = vxlan
prevent_arp_spoofing = True
[dhcp]
[network_log]
[ovs]
local_ip = 10.0.0.11
bridge_mappings = physnet1:br-ens38
[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

* 配置nova文件识别neutron配置

```
vim /etc/nova/nova.conf
'''
[DEFAULT]
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSlnterfaceDriver
vif_plugging_is_fatal = true
vif_pligging_timeout = 300

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
'''
```

* 重启nova服务识别网络配置

```
service nova-compute restart
```

* 新建一个外部网络桥接

```
ovs-vsctl add-br br-ens38
```

* 将外部网络桥接映射到网卡
  * 这里绑定第二张网卡，属于业务网卡

```
ovs-vsctl add-port br-ens38 ens38
```

* 重启服务加载ovs配置

```
service neutron-openvswitch-agent restart
```

* compute02节点
* 配置内核转发

```
cat >> /etc/sysctl.conf << EOF
# 用于控制系统是否开启对数据包源地址的校验，关闭
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
# 开启二层转发设备
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
```

* 加载模块
  * 作用：桥接流量转发到iptables链

```
modprobe br_netfilter
```

* 生效内核配置

```
sysctl -p
```

* 安装neutron-ovs服务

```
apt install -y neutron-openvswitch-agent
```

* 配置neutron文件
  * 提供neutron主体服务

```
# 备份文件
cp /etc/neutron/neutron.conf{,.bak}

# 过滤提取文件
grep -Ev "^$|#" /etc/neutron/neutron.conf.bak > /etc/neutron/neutron.conf

# 完整配置
vim /etc/neutron/neutron.conf
[DEFAULT]
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
state_path = /var/lib/neutron
allow_overlapping_ips = true
transport_url = rabbit://openstack:000000@controller
[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"
[cache]
[cors]
[database]
[healthcheck]
[ironic]
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron
[nova]
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[placement]
[privsep]
[quotas]
[ssl]
```

* 配置openvswitch\_agent.ini文件
  * 提供ovs网络服务

```
# 备份文件
cp /etc/neutron/plugins/ml2/openvswitch_agent.ini{,.bak}

# 过滤提取文件
grep -Ev "^$|#" /etc/neutron/plugins/ml2/openvswitch_agent.ini.bak > /etc/neutron/plugins/ml2/openvswitch_agent.ini

# 完整配置
vim /etc/neutron/plugins/ml2/openvswitch_agent.ini
[DEFAULT]
[agent]
l2_population = True
tunnel_types = vxlan
prevent_arp_spoofing = True
[dhcp]
[network_log]
[ovs]
local_ip = 10.0.0.12
bridge_mappings = physnet1:br-ens38
[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

* 配置nova文件识别neutron配置

```
vim /etc/nova/nova.conf
'''
[DEFAULT]
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSlnterfaceDriver
vif_plugging_is_fatal = true
vif_pligging_timeout = 300

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
'''
```

* 重启nova服务识别网络配置

```
service nova-compute restart
```

* 新建一个外部网络桥接

```
ovs-vsctl add-br br-ens38
```

* 将外部网络桥接映射到网卡
  * 这里绑定第二张网卡，属于业务网卡

```
ovs-vsctl add-port br-ens38 ens38
```

* 重启服务加载ovs配置

```
service neutron-openvswitch-agent restart
```

#### 8.3 校验neutron

* 校验命令

```
root@controller:~# openstack network agent list
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| 5695085f-b03f-4ff2-b13f-a8e59036ca15 | Open vSwitch agent | controller | None              | :-)   | UP    | neutron-openvswitch-agent |
| 77f6b5e6-a761-49c6-8694-de4d3d52509f | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
| 87139cbc-27ee-4885-807e-96800816adca | Open vSwitch agent | compute01  | None              | :-)   | UP    | neutron-openvswitch-agent |
| 891696fa-01af-4fd9-87f0-ad3d432f05d0 | L3 agent           | controller | nova              | :-)   | UP    | neutron-l3-agent          |
| 91959f9b-db89-4021-b55e-888f71edb0b3 | DHCP agent         | controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
| e5598aa0-e71c-4a74-a11f-b415a2e4fdbb | Open vSwitch agent | compute02  | None              | :-)   | UP    | neutron-openvswitch-agent |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
```

### 9. 配置dashboard仪表盘服务

* 安装服务

```
apt install -y openstack-dashboard
```

* 配置local\_settings.py文件

```
vim /etc/openstack-dashboard/local_settings.py
'''
# 配置仪表板以在控制器节点上使用OpenStack服务
OPENSTACK_HOST = "controller"

# 在Dashboard configuration部分中，允许主机访问Dashboard
ALLOWED_HOSTS = ["*"]

# 配置memcached会话存储服务
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

# 启用Identity API版本3
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

# 启用对域的支持
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

# 配置API版本
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}

# 将Default配置为通过仪表板创建的用户的默认域
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

# 将用户配置为通过仪表板创建的用户的默认角色
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

# 启用卷备份
OPENSTACK_CINDER_FEATURES = {
    'enable_backup': True,
}

# 配置时区
TIME_ZONE = "Asia/Shanghai"
'''
```

* 重新加载web服务器配置

```
systemctl reload apache2
```

* 浏览器访问：http://conntroller/horizon

![https://www.notion.soimages/dashboard.png](https://www.notion.soimages/dashboard.png)

### 10. 部署配置cinder卷存储

#### 10.1 控制节点配置

* 创建数据库与用户给予cinder组件使用

```
# 创建cinder数据库
CREATE DATABASE cinder;

# 创建cinder用户
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'cinderang';
```

* 创建cinder用户

```
openstack user create --domain default --password cinder cinder
```

* 添加cinder用户到admin角色

```
openstack role add --project service --user cinder admin
```

* 创建cinder服务实体

```
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
```

* 创建cinder服务API端点

```
openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s

openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s

openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
```

* 安装cinder相关服务

```
apt install -y cinder-api cinder-scheduler
```

* 配置cinder.conf文件

```
# 备份文件
cp  /etc/cinder/cinder.conf{,.bak}

# 过滤覆盖文件
grep -Ev "^$|#" /etc/cinder/cinder.conf.bak > /etc/cinder/cinder.conf

# 完整配置
vim /etc/cinder/cinder.conf
[DEFAULT]
transport_url = rabbit://openstack:000000@controller
auth_strategy = keystone
my_ip = 10.0.0.10
[database]
connection = mysql+pymysql://cinder:cinderang@controller/cinder
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = cinder
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

* 填充数据库

```
su -s /bin/sh -c "cinder-manage db sync" cinder
```

* 配置nova服务可调用cinder服务

```
vim /etc/nova/nova.conf
'''
[cinder]
os_region_name = RegionOne
'''
```

* 重启nova服务生效cinder服务

```
service nova-api restart
```

* 重新启动块存储服务

```
service cinder-scheduler restart
```

* 平滑重启apache服务识别cinder页面

```
service apache2 reload
```

#### 10.2 计算节点配置

* compute01节点
* 安装支持的实用程序包

```
apt install -y lvm2 thin-provisioning-tools
```

* 创建LVM物理卷
  * 磁盘根据自己名称指定

```
pvcreate /dev/sdb
```

* 创建LVM卷组 cinder-volumes

```
vgcreate cinder-volumes /dev/sdb
```

* 修改lvm.conf文件
  * 作用：添加接受/dev/sdb设备并拒绝所有其他设备的筛选器

```
vim /etc/lvm/lvm.conf
devices {
...
filter = [ "a/sdb/", "r/.*/"]
```

* 安装cinder软件包

```
apt install -y cinder-volume tgt
```

* 配置cinder.conf配置文件

```
# 备份配置文件
cp /etc/cinder/cinder.conf{,.bak}

# 过滤覆盖文件
grep -Ev "^$|#" /etc/cinder/cinder.conf.bak > /etc/cinder/cinder.conf

# 完整配置文件
vim /etc/cinder/cinder.conf
[DEFAULT]
transport_url = rabbit://openstack:000000@controller
auth_strategy = keystone
my_ip = 10.0.0.11
enabled_backends = lvm
glance_api_servers = http://controller:9292
[database]
connection = mysql+pymysql://cinder:cinderang@controller/cinder
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = cinder
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = tgtadm
volume_backend_name = lvm
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

* 指定卷路径

```
vim /etc/tgt/conf.d/tgt.conf
include /var/lib/cinder/volumes/*
```

* 重新启动块存储卷服务，包括其依赖项

```
service tgt restart

service cinder-volume restart
```

* compute02节点
* 安装支持的实用程序包

```
apt install -y lvm2 thin-provisioning-tools
```

* 创建LVM物理卷
  * 磁盘根据自己名称指定

```
pvcreate /dev/sdb
```

* 创建LVM卷组 cinder-volumes

```
vgcreate cinder-volumes /dev/sdb
```

* 修改lvm.conf文件
  * 作用：添加接受/dev/sdb设备并拒绝所有其他设备的筛选器

```
vim /etc/lvm/lvm.conf
devices {
...
filter = [ "a/sdb/", "r/.*/"]
```

* 安装cinder软件包

```
apt install -y cinder-volume tgt
```

* 配置cinder.conf配置文件

```
# 备份配置文件
cp /etc/cinder/cinder.conf{,.bak}

# 过滤覆盖文件
grep -Ev "^$|#" /etc/cinder/cinder.conf.bak > /etc/cinder/cinder.conf

# 完整配置文件
vim /etc/cinder/cinder.conf
[DEFAULT]
transport_url = rabbit://openstack:000000@controller
auth_strategy = keystone
my_ip = 10.0.0.12
enabled_backends = lvm
glance_api_servers = http://controller:9292
[database]
connection = mysql+pymysql://cinder:cinderang@controller/cinder
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = cinder
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = tgtadm
volume_backend_name = lvm
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

* 指定卷路径

```
vim /etc/tgt/conf.d/tgt.conf
include /var/lib/cinder/volumes/*
```

* 重新启动块存储卷服务，包括其依赖项

```
service tgt restart

service cinder-volume restart
```

#### 10.3 校验cinder

* 校验命令

```
root@controller:~# openstack volume service list
+------------------+---------------+------+---------+-------+----------------------------+
| Binary           | Host          | Zone | Status  | State | Updated At                 |
+------------------+---------------+------+---------+-------+----------------------------+
| cinder-scheduler | controller    | nova | enabled | up    | 2022-09-29T07:58:33.000000 |
| cinder-volume    | compute01@lvm | nova | enabled | up    | 2022-09-29T07:58:29.000000 |
| cinder-volume    | compute02@lvm | nova | enabled | up    | 2022-09-29T07:58:34.000000 |
```

### 11. 运维实战

* 控制节点操作

#### 11.1 加载openstack环境变量

```
source /etc/keystone/admin-openrc.sh
```

#### 11.2 创建路由器

```
openstack router create Ext-Router
```

#### 11.3 创建Vxlan网络

* 创建vxlan网络

```
openstack network create --provider-network-type vxlan Intnal
```

* 创建vxlan子网

```
openstack subnet create Intsubnal --network Intnal --subnet-range 166.66.66.0/24 --gateway 166.66.66.1 --dns-nameserver 114.114.114.114
```

#### 11.4 将内部网络添加到路由器

* 添加命令

```
openstack router add subnet Ext-Router Intsubnal
```

#### 11.5 创建Flat网络

* 创建flat网络

```
openstack network create --provider-physical-network physnet1 --provider-network-type flat  --external Extnal
```

* 创建flat子网

```
openstack subnet create Extsubnal --network Extnal --subnet-range 10.0.0.0/24  --allocation-pool start=10.0.0.30,end=10.0.0.200 --gateway 10.0.0.254 --dns-nameserver 114.114.114.114 --no-dhcp
```

#### 11.6 设置路由器网关接口

```
openstack router set Ext-Router --external-gateway Extnal
```

#### 11.7 开放安全组

```
# 开放icmp协议
openstack security group rule create --proto icmp default

# 开放22端口
openstack security group rule create --proto tcp --dst-port 22:22 default

# 查看安全组规则
openstack security group rule list
```

#### 11.8 上传镜像

```
openstack image create cirros04 --disk-format qcow2  --file  cirros-0.4.0-x86_64-disk.img
```

#### 11.9 创建云主机

* 创建ssh-key密钥

```
ssh-keygen -N ""
```

* 创建密钥

```
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```

* 创建云主机类型

```
openstack flavor create --vcpus 1 --ram 512 --disk 1 C1-512MB-1G
```

* 创建云主机

```
openstack server create --flavor C1-512MB-1G --image cirros04 --security-group default --nic net-id=$(vxlan网络id) --key-name mykey vm01
```

* 分配浮动地址

```
openstack floating ip create Extnal
```

* 将分配的浮动IP绑定云主机

```
openstack server add floating ip vm01 $(分配出的地址)
```

* VNC查看实例

```
openstack console url show vm01
```

#### 11.10 创建卷类型

```
openstack volume type create lvm
```

#### 11.11 卷类型添加元数据

```
cinder --os-username admin --os-tenant-name admin type-key lvm set volume_backend_name=lvm
```

#### 11.12 查看卷类型

```
openstack volume type list
```

#### 11.13 创建卷

* 指定lvm卷类型创建卷

```
openstack volume create lvm01 --type lvm --size 1
```

#### 11.14 卷绑定云主机

* 将卷绑定云主机

```
nova volume-attach vm01 卷ID
```

## 二. Ceph集群部署

### 1. 环境准备

| 主机名   | IP        | 磁盘                        | CPU | memory |
| ----- | --------- | ------------------------- | --- | ------ |
| node1 | 10.0.0.18 | sda：100G，sdb：50G，sdc：50G  | 2C  | 4G     |
| node2 | 10.0.0.19 | sda：100G ，sdb：50G，sdc：50G | 2C  | 4G     |
| node3 | 10.0.0.20 | sda：100G ，sdb：50G，sdc：50G | 2C  | 4G     |

| 操作系统        | 虚拟化工具    |
| ----------- | -------- |
| Ubuntu22.04 | VMware15 |

#### 1.1 配置地址

* node1节点

```
cat > /etc/netplan/00-installer-config.yaml << EOF
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses: [10.0.0.18/24]
      gateway4: 10.0.0.254
  version: 2
EOF

# 生效网络
netplan apply
```

* node2节点

```
cat > /etc/netplan/00-installer-config.yaml << EOF
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses: [10.0.0.19/24]
      gateway4: 10.0.0.254
  version: 2
EOF

# 生效网络
netplan apply
```

* node3节点

```
cat > /etc/netplan/00-installer-config.yaml << EOF
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses: [10.0.0.20/24]
      gateway4: 10.0.0.254
  version: 2
EOF

# 生效网络
netplan apply
```

#### 1.2 更改主机名

* node1节点

```
hostnamectl set-hostname node1

# 切换窗口
bash
```

* node2节点

```
hostnamectl set-hostname node2

# 切换窗口
bash
```

* node3节点

```
hostnamectl set-hostname node3

# 切换窗口
bash
```

### 2. 配置hosts解析（所有节点）

> cat >> /etc/hosts <\<EOF 10.0.0.18 node1 10.0.0.19 node2 10.0.0.20 node3 EOF

### 3. 制作离线源（所有节点）

> 解压离线包并配置本地仓库
>
> ```
> tar zxvf ceph_quincy.tar.gz -C /opt/
>
> cp /etc/apt/sources.list{,.bak}
>
> cat > /etc/apt/sources.list << EOF
> deb [trusted=yes] file:// /opt/ceph_quincy/debs/
> EOF
>
> apt-get clean all
> apt-get update
> ```

### 4. 配置时间同步

> 所有节点更改时区
>
> ```
> # 可配置开启
> timedatectl set-ntp true
>
> # 配置上海时区
> timedatectl set-timezone Asia/Shanghai
>
> # 系统时钟与硬件时钟同步
> hwclock --systohc
> ```
>
> * node1节点
>
> ```
> # 安装服务
> apt install -y chrony
>
> # 配置文件
> vim /etc/chrony/chrony.conf
> 20 server controller iburst maxsources 2
> 61 allow all
> 63 local stratum 10
>
> # 重启服务
> systemctl restart chronyd
> ```
>
> * node2、node3节点
>
> ```
> # 安装服务
> apt install -y chrony
>
> # 配置文件
> vim /etc/chrony/chrony.conf
> 20 pool controller iburst maxsources 4
>
> # 重启服务
> systemctl restart chronyd
> ```

### 5. 安装docker(所有节点)

> apt -y install docker-ce

### 6. 安装cephadm(node1)

> apt install -y cephadm

### 7. 导入ceph镜像(所有节点)

> 将准备好的离线镜像全部导入
>
> ```
> docker load -i cephadm_images_v17.tar
> ```

#### 7.1 搭建制作本地仓库(node1)

> 启动仓库镜像
>
> ```
> # 导入镜像
> docker load -i registry.tar
>
> # 启动
> docker run -d --name registry -p 5000:5000 --restart always 3a0f7b0a13ef
> ```
>
> * 配置仓库地址
>
> ```
> cat >> /etc/docker/daemon.json << EOF
> {
> "insecure-registries":["10.0.0.18:5000"]
> }
> EOF
>
> systemctl daemon-reload
> systemctl restart docker
> ```
>
> * 打地址标签
>
> ```
> docker tag 0912465dcea5 10.0.0.18:5000/ceph:v17
> ```
>
> * 推入仓库
>
> ```
> docker push 10.0.0.18:5000/ceph:v17
> ```

#### 7.2 配置私有仓库

> node2、node3节点配置私有仓库
>
> ```
> cat >> /etc/docker/daemon.json << EOF
> {
> "insecure-registries":["10.0.0.18:5000"]
> }
> EOF
>
> systemctl daemon-reload
> systemctl restart docker
> ```

### 8. 引导集群(node1)

> 初始化mon节点
>
> ```
> mkdir -p /etc/ceph
>
> cephadm --image 10.0.0.18:5000/ceph:v17 bootstrap --mon-ip 10.0.0.18 --initial-dashboard-user admin --initial-dashboard-password 000000 --skip-pull
>
> ps:
> # 要部署其他监视器
> ceph orch apply mon "test01,test02,test03"
>
> # 删除集群
> cephadm rm-cluster --fsid d92b85c0-3ecd-11ed-a617-3f7cf3e2d6d8 --force
> ```

### 9. 安装ceph-common工具(node1)

> 安装服务
>
> ```
> apt install -y ceph-common
> ```

### 10. 添加主机到集群(node1)

> 传输ceph密钥
>
> ```
> ssh-copy-id -f -i /etc/ceph/ceph.pub node2
>
> ssh-copy-id -f -i /etc/ceph/ceph.pub node3
> ```
>
> * 集群机器发现
>
> ```
> ceph orch host add node2
>
> ceph orch host add node3
> ```

### 11. 部署OSD

* 存储数据

> node1机器
>
> ```
> # 查看可用的磁盘设备
> ceph orch device ls
>
> # 添加到ceph集群中,在未使用的设备上自动创建osd
> ceph orch apply osd --all-available-devices
>
> PS:
> # 从特定主机上的特定设备创建OSD：
> ceph orch daemon add osd node1:/dev/sdb
> ceph orch daemon add osd node2:/dev/sdb
> ceph orch daemon add osd node3:/dev/sdb
>
> # 查看osd磁盘
> ceph -s
>
> ceph df
> ```

### 12. 访问仪表盘查看状态

* 访问：https://10.0.0.18:8443/

![https://www.notion.soimages/ceph.png](https://www.notion.soimages/ceph.png)

* 访问：https://10.0.0.18:3000/

![https://www.notion.soimages/grafana.png](https://www.notion.soimages/grafana.png)

## 三. OpenStack对接Ceph平台

### 1. 创建后端需要的存储池

* node1节点操作

#### 1.1 cinder卷的存储池

```
ceph osd pool create volumes 32
```

#### 1.2 glance存储池

```
ceph osd pool create images 32
```

#### 1.3 备份存储池

```
ceph osd pool create backups 32
```

#### 1.4 创建实例存储池

```
ceph osd pool create vms 32
```

### 2. 创建后端用户

#### 2.1 创建密钥

* node1节点操作
*
* 切换到ceph目录

```
cd /etc/ceph/
```

* **在ceph上创建cinder、glance、cinder-backup、nova用户创建密钥，允许访问使用Ceph存储池**

#### 2.1.1 创建用户client.cinder

* 对volumes存储池有rwx权限，对vms存储池有rwx权限，对images池有rx权限

```
ceph auth get-or-create client.cinder mon "allow r" osd "allow class-read object_prefix rbd_children,allow rwx pool=volumes,allow rwx pool=vms,allow rx pool=images"

# class-read：x的子集，授予用户调用类读取方法的能力

# object_prefix 通过对象名称前缀。下例将访问限制为任何池中名称仅以 rbd_children 为开头的对象。
```

#### 2.1.2 创建用户client.glance

* 对images存储池有rwx权限

```
ceph auth get-or-create client.glance mon "allow r" osd "allow class-read object_prefix rbd_children,allow rwx pool=images"
```

#### 2.1.3 创建用户client.cinder-backup

* 对backups存储池有rwx权限

```
ceph auth get-or-create client.cinder-backup mon "profile rbd" osd "profile rbd pool=backups"

# 使用 rbd profile 为新的 cinder-backup 用户帐户定义访问权限。然后，客户端应用使用这一帐户基于块来访问利用了 RADOS 块设备的 Ceph 存储。
```

#### 2.2 创建存放目录

* controller节点

```
mkdir /etc/ceph/
```

* compute01节点

```
mkdir /etc/ceph/
```

* compute02节点

```
mkdir /etc/ceph/
```

#### 2.3 导出密钥

* node1节点
* 导出glance密钥

```
ceph auth get client.glance -o ceph.client.glance.keyring
```

* 导出cinder密钥

```
ceph auth get client.cinder -o ceph.client.cinder.keyring
```

* 导出cinder-backup密钥

```
ceph auth get client.cinder-backup -o ceph.client.cinder-backup.keyring
```

#### 2.4 拷贝密钥

* node1节点操作

#### 2.4.1 控制节点准备

* 拷贝glance密钥

```
scp ceph.client.glance.keyring root@controller:/etc/ceph/
```

* 拷贝cinder密钥

```
scp ceph.client.cinder.keyring root@controller:/etc/ceph/
```

* 拷贝ceph集群认证配置文件

```
scp ceph.conf root@controller:/etc/ceph/
```

#### 2.4.2 计算节点准备

* 拷贝cinder密钥

```
scp ceph.client.cinder.keyring root@compute01:/etc/ceph/

scp ceph.client.cinder.keyring root@compute02:/etc/ceph/
```

* 拷贝cinder-backup密钥（backup服务节点）

```
scp ceph.client.cinder-backup.keyring root@compute01:/etc/ceph/

scp ceph.client.cinder-backup.keyring root@compute02:/etc/ceph/
```

* 拷贝ceph集群认证配置文件

```
scp ceph.conf root@compute01:/etc/ceph/

scp ceph.conf root@compute02:/etc/ceph/
```

### 3. 计算节点添加libvirt密钥

#### 3.1 compute01添加密钥

* 生成密钥（PS：注意，如果有多个计算节点，它们的UUID必须一致）

```
cd /etc/ceph/

UUID=$(uuidgen)

cat >> secret.xml << EOF
<secret ephemeral='no' private='no'>
  <uuid>$UUID</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF
```

* 执行命令写入secret

```
[root@compute01 ~]# virsh secret-define --file secret.xml
Secret bf168fa8-8d5b-4991-ba4c-12ae622a98b1 created
```

* 加入key

```
# 将key值复制出来
[root@compute01 ~]# cat ceph.client.cinder.keyring
AQALyS1jHz4dDRAAEmt+c8JlXWyzxmCx5vobZg==

[root@compute01 ~]# virsh secret-set-value --secret ${UUID} --base64 $(cat ceph.client.cinder.keyring | grep key | awk -F ' ' '{print $3}')
```

* 查看添加后端密钥

```
virsh secret-list
```

#### 3.2 compute02添加密钥

* 生成密钥（PS：注意，如果有多个计算节点，它们的UUID必须一致）

```
cd /etc/ceph/

UUID=bf168fa8-8d5b-4991-ba4c-12ae622a98b1

cat >> secret.xml << EOF
<secret ephemeral='no' private='no'>
  <uuid>$UUID</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF
```

* 执行命令写入secret

```
[root@compute02 ~]# virsh secret-define --file secret.xml
Secret bf168fa8-8d5b-4991-ba4c-12ae622a98b1 created
```

* 加入key

```
# 将key值复制出来
[root@compute02 ~]# cat ceph.client.cinder.keyring
AQALyS1jHz4dDRAAEmt+c8JlXWyzxmCx5vobZg==

[root@compute02 ~]# virsh secret-set-value --secret ${UUID} --base64 $(cat ceph.client.cinder.keyring | grep key | awk -F ' ' '{print $3}')

# 忽略报错信息
```

* 查看添加后端密钥

```
virsh secret-list
```

### 4. 安装ceph客户端

* 主要作用是OpenStack可调用Ceph资源
* controller节点

```
apt install -y ceph-common
```

* compute01节点

```
apt install -y ceph-common
```

* compute02节点

```
apt install -y ceph-common
```

### 5. 配置glance后端存储

* controller节点
* 更改glance密钥属性

```
chown glance.glance /etc/ceph/ceph.client.glance.keyring
```

* 修改配置文件

```
vim /etc/glance/glance-api.conf
[glance_store]
#stores = file,http
#default_store = file
#filesystem_store_datadir = /var/lib/glance/images/
stores = rbd,file,http
default_store = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
```

* 安装缺失aws的模块

```
apt install -y python3-boto3
```

* 重启生效ceph配置

```
service glance-api restart
```

* 上传镜像

```
openstack image create cirros04_v1 --disk-format qcow2 --file cirros-0.4.0-x86_64-disk.img
```

* 到node1节点验证镜像

```
rbd images ls
```

### 6. 配置cinder后端存储

* 更改cinder密钥属性（controller、compute01、compute02节点）

```
chown cinder.cinder /etc/ceph/ceph.client.cinder.keyring
```

* 修改配置文件（controller节点）

```
vim /etc/cinder/cinder.conf
[DEFAULT]
# 指定存储类型，否则在创建卷时，类型为 __DEFAULT__
default_volume_type = ceph

# 重启服务生效配置
service cinder-scheduler restart
```

* 修改配置文件（compute01、compute02存储节点）

```
vim /etc/cinder/cinder.conf
[DEFAULT]
enabled_backends = ceph,lvm

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
glance_api_version = 2
rbd_user = cinder
rbd_secret_uuid = bf168fa8-8d5b-4991-ba4c-12ae622a98b1
volume_backend_name = ceph

# 重启服务生效配置
service cinder-volume restart
```

* 创建卷类型（controller节点）

```
openstack volume type create ceph
```

* 设置卷类型元数据（controller节点）

```
cinder --os-username admin --os-tenant-name admin type-key ceph set volume_backend_name=ceph
```

* 查看存储类型（controller节点）

```
openstack volume type list
```

* 创建卷测试（controller节点）

```
openstack volume create ceph01 --type ceph --size 1
```

* 查看volumes存储池是否存在卷

```
rbd ls volumes
```

### 7. 配置卷备份

* compute01、compute02节点
* 安装服务

```
apt install cinder-backup -y
```

* 更改密钥属性

```
chown cinder.cinder /etc/ceph/ceph.client.cinder-backup.keyring
```

* 修改配置文件

```
vim /etc/cinder/cinder.conf
[DEFAULT]
backup_driver = cinder.backup.drivers.ceph.CephBackupDriver
backup_ceph_conf=/etc/ceph/ceph.conf
backup_ceph_user = cinder-backup
backup_ceph_chunk_size = 4194304
backup_ceph_pool = backups
backup_ceph_stripe_unit = 0
backup_ceph_stripe_count = 0
restore_discard_excess_bytes = true
```

* 重启生效配置

```
service cinder-backup restart
```

* 创建卷备份（controller节点）

```
openstack volume backup create --name ceph_backup ceph01
```

* 验证卷备份（node1节点）

```
rbd ls backups
```

### 8. 配置nova集成ceph

* compute01、compute02节点
* 修改配置文件

```
vim /etc/nova/nova.conf
[DEFAULT]
live_migration_flag = "VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE"

[libvirt]
images_type = rbd
images_rbd_pool = vms
images_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = bf168fa8-8d5b-4991-ba4c-12ae622a98b1
```

* 安装qemu支持rbd

```
apt install -y qemu-block-extra
```

* 重启nova服务生效配置

```
service nova-compute restart
```

* 创建实例测试（controller节点）

```
openstack server create --flavor C1-512MB-1G --image cirros04_v1 --security-group default --nic net-id=$(vxlan网络id) --key-name mykey vm02

# 安全组对应admin项目ID
```

* 验证是否到ceph中的vms存储池

```
rbd ls vms
```

#### 8.1 热迁移配置

* compute01、compute02节点
* 配置监听地址

```
vim /etc/libvirt/libvirtd.conf
listen_tls = 0
listen_tcp = 1
tcp_port = "16509"
listen_addr = "10.0.0.12"     # 注意自己的主机地址
auth_tcp = "none"
```

* 开启监听地址

```
vim /etc/default/libvirtd
LIBVIRTD_ARGS="--listen"
```

* 屏蔽libvirtd服务

```
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
```

* 重启libvirtd生效配置

```
service libvirtd restart
```

* 重启计算节点nova服务

```
service nova-compute restart
```

* 测试是否能互相通信连接
  * 互通测试再进行热迁移
* compute01连接compute02

```
virsh -c qemu+tcp://compute02/system
```

* compute02连接compute01

```
virsh -c qemu+tcp://compute01/system
```

* 查看云主机

```
openstack server list
```

* 查看需要迁移的云主机详细信息

```
openstack server show fdb31a02-9c44-481b-9d22-224c776e2304
```

* 热迁移到另一个计算节点

```
nova live-migration fdb31a02-9c44-481b-9d22-224c776e2304 compute01
```
