# openstack-train

### 整体规划

```jsx
   **     节点                   内网ip                       外网ip
     controller            192.168.20.100              192.168.100.100
      compute              192.168.20.200              192.168.100.200**
```

### **controller**

修改主机名称

```jsx
[root@localhost ~]# hostnamectl set-hostname controller
[root@localhost ~]# bash
```

修改网卡配置

**外网网卡—ens33**

\[root@localhost \~]# vi /etc/sysconfig/network-scripts/ifcfg-ens33

```jsx
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=c7ec36cd-254d-45b7-8d4c-93176e1633d0
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.100.100
NETMASK=255.255.255.0
GATEWAY=192.168.100.254
DNS1=114.114.114.114
DNS2=8.8.8.8
```

> BOOTPROTO设置为static静态模式 即手动配置不自动获取

> ONBOOT指定网络接口是否在系统启动时自动启动，yes 表示自动启动，n**o** 表示不自动启动

**内网网卡—ens34（内网网卡不需要设置网关）**

\[root@localhost \~]# vi /etc/sysconfig/network-scripts/ifcfg-ens34

```jsx
YPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens32
UUID=373532c6-9e2e-4f2d-9139-3a7e173958b0
DEVICE=ens32
ONBOOT=yes
IPADDR=192.168.20.100
NETMASK=255.255.255.0
```

重启网络配置使其生效

```jsx
[root@localhost ~]# systemctl restart network
```

查看重启后的网卡ip

```jsx
[root@controller ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ef:23:ac brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.100/24 brd 192.168.100.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::e6b7:e547:4777:89d9/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::ced6:fa2:c4e:ce43/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ef:23:b6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.20.100/24 brd 192.168.20.255 scope global noprefixroute ens34
       valid_lft forever preferred_lft forever
    inet6 fe80::708e:880f:93e:9c63/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::8382:5a58:74c3:bd21/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever

```

关闭防火墙、设置永久关闭、查看其状态（实验中可以关闭；生产环境中添加规则进行端口放行操作）

```jsx
systemctl stop firewalld  && systemctl disable firewalld  &&  systemctl status firewalld
```

关闭selinux 修改配置文件使其永久关闭

```jsx
[root@controller ~]# getenforce

Enforcing

[root@controller ~]# vi /etc/selinux/config

SELINUX=permissive

[root@controller ~]# setenforce permissive

[root@controller ~]# getenforce

Permissive
```

配置本地yum仓库

\[root@controller \~]# vi /etc/yum.repos.d/dvd.repo

```jsx
[base]
name=base
baseurl=file:///mnt/cdrom
enabled=1
gpgcheck=0
```

> **\[base]**：定义了这个仓库的 ID，为 **`base`**。这是 YUM 配置文件的标准格式，用方括号括起来那部分表示仓库的唯一标识符。

> **name=base**：这是该仓库的名称，可以是任意描述性的名称。这里简单地命名为 **`base`**。

> **baseurl=file:///mnt/cdrom**：这是该仓库的基础 URL，指定了该仓库的位置。这里使用 **`file:///mnt/cdrom`**，表示该仓库位于本地文件系统的 **`/mnt/cdrom`** 目录中。通常这用于指向一个挂载的 CD 或 DVD。

> **enabled=1**：表示启用这个仓库。**`1`** 表示启用，**`0`** 表示禁用。

> **gpgcheck=0**：表示不进行 GPG 签名检查。**`0`** 表示禁用 GPG 检查，**`1`** 表示启用 GPG 检查。

创建挂载目录并进行挂载

```jsx
[root@controller ~]# mkdir /mnt/cdrom

[root@controller ~]# mount /dev/cdrom /mnt/cdrom/
mount: /dev/sr0 is write-protected, mounting read-only
```

查看挂载信息

```jsx
[root@controller ~]# df -h

Filesystem Size Used Avail Use% Mounted on
/dev/mapper/centos-root 47G 1010M 46G 3% /
devtmpfs 476M 0 476M 0% /dev
tmpfs 488M 0 488M 0% /dev/shm
tmpfs 488M 7.6M 480M 2% /run
tmpfs 488M 0 488M 0% /sys/fs/cgroup
/dev/sda1 1014M 130M 885M 13% /boot
tmpfs 98M 0 98M 0% /run/user/0
/dev/sr0 4.2G 4.2G 0 100% /mnt/cdrom
```

配置本地域名解析

\[root@controller \~]# vi /etc/hosts

```jsx
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.20.100 controller
192.168.20.200 compute
```

测试域名解析是否正确

```jsx
[root@controller ~]# ping compute -c4
PING compute (192.168.20.200) 56(84) bytes of data.
64 bytes from compute (192.168.20.200): icmp_seq=1 ttl=64 time=0.579 ms
64 bytes from compute (192.168.20.200): icmp_seq=2 ttl=64 time=0.349 ms
64 bytes from compute (192.168.20.200): icmp_seq=3 ttl=64 time=1.09 ms
64 bytes from compute (192.168.20.200): icmp_seq=4 ttl=64 time=1.22 ms

--- compute ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 0.349/0.812/1.226/0.362 ms
[root@controller ~]# ping controller -c4
PING controller (192.168.20.100) 56(84) bytes of data.
64 bytes from controller (192.168.20.100): icmp_seq=1 ttl=64 time=0.023 ms
64 bytes from controller (192.168.20.100): icmp_seq=2 ttl=64 time=0.046 ms
64 bytes from controller (192.168.20.100): icmp_seq=3 ttl=64 time=0.044 ms
64 bytes from controller (192.168.20.100): icmp_seq=4 ttl=64 time=0.038 ms

--- controller ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 2999ms
rtt min/avg/max/mdev = 0.023/0.037/0.046/0.011 ms

```

配置时间同步服务（由于系统自带，故无需安装）

\[root@controller \~]# vi /etc/chrony.conf

```jsx
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst
server ntp1.aliyun.com iburst
server time.cloud.tencent.com iburst
# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.
allow 192.168.20.0/24

# Serve time even if not synchronized to a time source.
local stratum 1

# Specify file containing keys for NTP authentication.
#keyfile /etc/chrony.keys

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking

```

> server ntp1.aliyun.com iburst server time.cloud.tencent.com iburst

> 添加阿里和腾讯的NTP服务器

> allow 192.168.20.0/24 设置允许某个网段的chrony客户端使用本机NTP服务

重启chrony服务并设置为开机自启动

```jsx
systemctl restart chronyd && systemctl enable chronyd
```

查看当前客户端与NTP服务器的连接情况

```jsx
[root@controller ~]# chronyc sources
210 Number of sources = 6
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^+ 119.28.183.184                2   9   377   346   +212us[-1281us] +/-   40ms
^* 139.199.215.251               2   9   373   164  -3060us[-4631us] +/-   72ms
^- time.cloudflare.com           3   9   365   351    +12ms[  +10ms] +/-  131ms
^? 111.203.6.13                  1   9     3   160    -18ms[  -18ms] +/-   37ms
^+ 120.25.115.20                 2   9   213   667  -2707us[-4065us] +/-   32ms
^+ 106.55.184.199                2   8   375   160    -47us[  -47us] +/-   75ms
```

配置FTP服务器（用来共享controller上的软件仓库）

* 下载FTP服务

```jsx
[root@controller ~]# yum install -y vsftpd
```

* 配置主配置文件

\[root@controller \~]# vi /etc/vsftpd/vsftpd.conf

```jsx
anon_root=/opt
```

> anon\_root=/opt 将匿名用户访问的主目录指向软件仓库所在的实际目录

重新启动FTP服务并设置为开机自启动

```jsx
systemctl restart vsftpd &&  systemctl enable vsftpd
```

删除yum文件夹中自带的yum源

```jsx
rm -rf /etc/yum.repos.d/C*
```

创建一个openstack挂载点

```jsx
mkdir /opt/openstack
```

将openStack软件包上传至/opt目录下，并进行挂载

```jsx
mount /opt/openStack-train.iso /opt/openstack/
```

设置永久挂载

```jsx
[root@controller ~]# vi /etc/fstab
/opt/openStack-train.iso /opt/openstack iso9660 defaults 0 0
```

编写本地openstack的yum仓库

```jsx
[root@controller ~]# vi /etc/yum.repos.d/dvd.repo

[base]
name=base
baseurl=file:///opt/openstack/base
enabled=1
gpgcheck=0
[updates]
name=updates
baseurl=file:///opt/openstack/updates
enabled=1
gpgcheck=0
[extras]
name=extras
baseurl=file:///opt/openstack/extras
enabled=1
gpgcheck=0
[virt]
name=virt
baseurl=file:///opt/openstack/virt
enabled=1
gpgcheck=0
[train]
name=train
baseurl=file:///opt/openstack/train
enabled=1
gpgcheck=0
```

清除原有的yum源缓存并重新建立缓存

```jsx
[root@controller ~]# yum clean all

[root@controller ~]# yum makecache
```

安装网络工具软件

```jsx
yum -y install net-tools
```

> net-tools 传统的网络配置和管理工具

### **compute**

修改主机名称

```jsx
[root@localhost ~]# hostnamectl set-hostname compute
[root@localhost ~]# bash
[root@compute~]#
```

修改网卡配置

**外网网卡—ens33**

\[root@compute\~]# vi /etc/sysconfig/network-scripts/ifcfg-ens33

```jsx
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=c7ec36cd-254d-45b7-8d4c-93176e1633d0
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.100.200
NETMASK=255.255.255.0
GATEWAY=192.168.100.254
DNS1=114.114.114.114
DNS2=8.8.8.8
```

> BOOTPROTO设置为static静态模式 即手动配置不自动获取

> ONBOOT指定网络接口是否在系统启动时自动启动，yes 表示自动启动，n**o** 表示不自动启动

**内网网卡—ens34（内网网卡不需要设置网关）**

\[root@compute\~]# vi /etc/sysconfig/network-scripts/ifcfg-ens34

```jsx
YPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens32
UUID=373532c6-9e2e-4f2d-9139-3a7e173958b0
DEVICE=ens32
ONBOOT=yes
IPADDR=192.168.20.200
NETMASK=255.255.255.0
```

重启网络配置使其生效

```jsx
[root@compute~]# systemctl restart network
```

查看重启后的网卡ip

```jsx
[root@compute ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:e3:f1:82 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.200/24 brd 192.168.100.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ced6:fa2:c4e:ce43/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:e3:f1:8c brd ff:ff:ff:ff:ff:ff
    inet 192.168.20.200/24 brd 192.168.20.255 scope global noprefixroute ens34
       valid_lft forever preferred_lft forever
    inet6 fe80::8382:5a58:74c3:bd21/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

关闭防火墙、设置永久关闭、查看其状态（实验中可以关闭；生产环境中添加规则进行端口放行操作）

```jsx
systemctl stop firewalld  && systemctl disable firewalld  &&  systemctl status firewalld
```

关闭selinux 修改配置文件使其永久关闭

```jsx
[root@compute ~]# getenforce

Enforcing

[root@compute ~]# vi /etc/selinux/config

SELINUX=permissive

[root@compute ~]# setenforce permissive

[root@compute ~]# getenforce

Permissive
```

配置本地域名解析

\[root@compute \~]# vi /etc/hosts

```jsx
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.20.100 controller
192.168.20.200 compute
```

可以复制controller节点的yum文件进行修改

```jsx
[root@compute ~]# scp root@controller:/etc/yum.repos.d/dvd.repo /etc/yum.repos.d/
The authenticity of host 'controller (192.168.10.100)' can't be established.
ECDSA key fingerprint is SHA256:SXgEq91biP2JH4JRkSq0qFu6wo5kDFrpCnN+0T0P1zU.
ECDSA key fingerprint is MD5:94:68:f4:2d:54:68:9d:b8:4f:78:18:cb:c9:7d:73:9a.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'controller,192.168.10.100' (ECDSA) to the list of known hosts.
root@controller's password:
dvd.repo 100% 388 111.2KB/s 00:00
```

配置yum源使其指向controller节点FTP服务器中的软件仓库

```jsx
root@compute yum.repos.d]# vi dvd.repo

[base]
name=base
baseurl=ftp://controller/openstack/base/
enabled=1
gpgcheck=0
[extras]
name=extras
baseurl=ftp://controller/openstack/extras/
enabled=1
gpgcheck=0
[updates]
name=updates
baseurl=ftp://controller/openstack/updates/
enabled=1
gpgcheck=0
[train]
name=train
baseurl=ftp://controller/openstack/train/
enabled=1
gpgcheck=0
[virt]
name=virt
baseurl=ftp://controller/openstack/virt/
enabled=1
gpgcheck=0
```

删除yum文件夹中自带的yum源

```jsx
rm -rf /etc/yum.repos.d/C*
```

清除原有的yum源缓存并重新建立缓存

```jsx
[root@controller ~]# yum clean all

[root@controller ~]# yum makecache
```

安装网络工具软件

```jsx
yum -y install net-tools
```

> net-tools 传统的网络配置和管理工具

配置compute端时间同步服务

```jsx
vi /etc/chrony.conf

server controller iburst
```

重启chrony服务并设置为开机自启动

```jsx
systemctl restart chronyd && systemctl enable chronyd
```

查看当前compute节点与controller节点的同步情况

```jsx
[root@compute ~]# chronyc sources
210 Number of sources = 5
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* dns1.synet.edu.cn             1  10   377    70  -6987us[-6742us] +/-   24ms
^+ time.neu.edu.cn               1  10   377   792   +492us[ +529us] +/-   17ms
^- time.cloudflare.com           3   9   377   211  +7335us[+7539us] +/-  133ms
^? 196.81.48.144.rdns.as486>     2   9     7  1036   +370ms[ +370ms] +/-  509ms
^+ controller                    3   6   377    21  -3259us[-3259us] +/-   39ms
```

## **controller**

云计算平台基础框架

将现有软件除内核外全部升级到最新版本

```jsx
[root@controller ~]# yum -y upgrade
```

将升级完找回的repo文件，全部删除

```jsx
[root@controller ~]# rm -rf /etc/yum.repos.d/C*
```

安装openstack云计算平台管理客户端（安装完成后可以使用openstack命令管理云计算平台）

```jsx
[root@controller ~]# yum install -y python-openstackclient
```

安装openstack selinux防火墙管理包

```jsx
[root@controller ~]# yum install -y openstack-selinux
```

### MariaDB数据库服务

下载mariadb服务

```jsx
yum install -y mariadb-server python2-PyMySQL
```

创建数据库配置文件并进行编写

```jsx
vi /etc/my.cnf.d/openstack.cnf

[mysqld]
bind-address=192.168.20.100
default-storage-engine=innodb
innodb_file_per_table=on
max_connections=4096
collation-server=utf8_general_ci
character-set-server=utf8
```

> **bind-address=192.168.20.100**：设置 MySQL 服务器监听的 IP 地址，这意味着 MySQL 服务器将只监听来自该 IP 地址的连接请求。

> **default-storage-engine=innodb**：设置默认的存储引擎为 **`InnoDB`**，这是一种具有事务支持的存储引擎，适用于大多数应用场景。

> **innodb\_file\_per\_table=on**：启用 **`InnoDB`** 表的独立表空间文件，这样每个表的数据会存储在单独的 **`.ibd`** 文件中，有助于管理和备份。

> **max\_connections=4096**：设置 MySQL 服务器允许的最大连接数为 **`4096`**，以支持更多的并发连接。

> **collation-server=utf8\_general\_ci**：设置服务器的默认排序规则为 **`utf8_general_ci`**，用于字符串比较。

> **character-set-server=utf8**：设置服务器的默认字符集为 **`utf8`**，用于存储和传输字符数据。

重启数据库并设置为开机自启动

```jsx
systemctl restart mariadb && systemctl enable mariadb
```

初始化数据库

```jsx
[root@controller ~]# mysql_secure_installation 

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none): 回车
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] y
New password: 
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!

By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

登录数据库进行测试

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 16
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> quit
Bye
```

### RabbitMQ消息队列服务

下载rabbitmq服务

启动消息队列服务并设置为开机自启动

```jsx
yum install -y rabbitmq-server
```

```jsx
systemctl start rabbitmq-server && systemctl enable rabbitmq-server
```

为rabbitmq系统添加一个名为“rabbitmq”的用户并设置密码为“123456”，以后可以通过该服务进行使用消息队列服务

```jsx
rabbitmqctl add_user rabbitmq 123456
```

为rabbitmq用户授予消息队列服务中所有资源的配置、写入、读取权限

```jsx
rabbitmqctl set_permissions rabbitmq ".*" ".*" ".*"
```

检查消息队列服务运行情况（查看对外端口25672和5672是否处于监听状态）

```jsx
[root@controller ~]# netstat -tnlup | grep 5672
tcp        0      0 0.0.0.0:25672           0.0.0.0:*               LISTEN      52240/beam.smp      
tcp6       0      0 :::5672                 :::*                    LISTEN      52240/beam.smp   
```

### Memcached缓存服务

安装memcached缓存服务

```jsx
yum install -y memcached python-mecached
```

配置缓存服务，设置对控制节点的访问进行监听

```jsx
vi /etc/sysconfig/memcached 

OPTIONS="-l 127.0.0.1,::1,controller"
```

> 使 Memcached 监听 **`127.0.0.1`**、**`::1`** 和 **`controller`** 这三个地址

> **`127.0.0.1`**：IPv4 本地回环地址。

> **`::1`**：IPv6 本地回环地址。

> **`controller`**：假设这是你的主机名，该主机名需要在 **`/etc/hosts`** 文件中正确解析。

重启服务并设置为开机自启动

```jsx
systemctl restart memcached && systemctl enable memcached
```

检查缓存服务运行情况（查看对外端口11211是否处于监听状态）

```jsx
[root@controller ~]# netstat -tnlup | grep 11211
tcp        0      0 192.168.20.100:11211    0.0.0.0:*               LISTEN      54918/memcached     
tcp        0      0 127.0.0.1:11211         0.0.0.0:*               LISTEN      54918/memcached     
tcp6       0      0 ::1:11211               :::*                    LISTEN      54918/memcached   
```

### 安装etcd分布式键-值对存储系统

安装etcd服务

```jsx
yum install -y etcd
```

修改主配置文件(修改并取消注释)

```jsx
vi /etc/etcd/etcd.conf 

ETCD_LISTEN_PEER_URLS="http://192.168.20.100:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.20.100:2379,http://127.0.0.1:2379"
ETCD_NAME="controller"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.20.100:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.20.100:2379"
ETCD_INITIAL_CLUSTER="controller=http://192.168.20.100:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"
```

> **ETCD\_LISTEN\_PEER\_URLS**: 指定 etcd 节点用于监听来自同伴节点的请求的 URL。在这里设置为 **`http://192.168.20.100:2380`**，表示 etcd 将监听来自同伴节点的请求，IP 地址为 **`192.168.20.100`**，端口为 **`2380`**。

> **ETCD\_LISTEN\_CLIENT\_URLS**: 指定 etcd 节点用于监听来自客户端的请求的 URL。在这里设置为 **`http://192.168.20.100:2379,http://127.0.0.1:2379`**，表示 etcd 将监听来自客户端的请求，IP 地址为 **`192.168.20.100`** 和 **`127.0.0.1`**，端口为 **`2379`**。

> **ETCD\_NAME**: 设置当前 etcd 节点的名称。在这里设置为 **`controller`**。

> **ETCD\_INITIAL\_ADVERTISE\_PEER\_URLS**: 设置当前节点在集群中广告其对等节点的地址。在这里设置为 **`http://192.168.20.100:2380`**，表示该节点在集群中广告自己的地址为 **`http://192.168.20.100:2380`**。

> **ETCD\_ADVERTISE\_CLIENT\_URLS**: 设置当前节点在集群中广告其客户端访问地址。在这里设置为 **`http://192.168.20.100:2379`**，表示该节点在集群中广告自己的客户端访问地址为 **`http://192.168.20.100:2379`**。

> **ETCD\_INITIAL\_CLUSTER**: 设置初始集群配置。在这里设置为 **`controller=http://192.168.20.100:2380`**，表示初始集群中只有一个节点，节点名称为 **`controller`**，对应的地址为 **`http://192.168.20.100:2380`**。

> **ETCD\_INITIAL\_CLUSTER\_TOKEN**: 设置用于初始化集群的 token。在这里设置为 **`etcd-cluster-01`**。

> **ETCD\_INITIAL\_CLUSTER\_STATE**: 设置集群的初始状态。在这里设置为 **`new`**，表示新的集群。

> 重启服务并设置为开机自启动

```jsx
systemctl restart etcd && systemctl enable etcd
```

检查etcd服务运行情况（查看对外端口2379和2380是否处于监听状态）

```jsx
[root@controller ~]# netstat -tnlup | grep etcd
tcp        0      0 192.168.20.100:2379     0.0.0.0:*               LISTEN      55411/etcd          
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      55411/etcd          
tcp        0      0 192.168.20.100:2380     0.0.0.0:*               LISTEN      55411/etcd         
```

### keystone服务

安装keystone服务

```jsx
yum install -y openstack-keystone httpd mod_wsgi
```

> openstack-keystone 为keystone服务的安装包；httpd为apache服务安装包； mod\_wsgi为web服务器支持的wsgi的插件

查看用户信息

```jsx
[root@controller ~]# cat /etc/passwd | grep keystone
keystone:x:163:163:OpenStack Keystone Daemons:/var/lib/keystone:/sbin/nologin
```

查看用户组信息

```jsx
[root@controller ~]# cat /etc/group | grep keystone
keystone:x:163:
```

进入MariaDB数据库服务器

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 17
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

创建一个名为“keystone”的数据库：
MariaDB [(none)]> CREATE DATABASE keystone;
Query OK, 1 row affected (0.002 sec)

将新建的数据库管理权限授予本地登录和远程登录的“keystone”用户：%为远程主机
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.002 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> QUIT
Bye
```

修改keystone配置文件

```jsx
vi /etc/keystone/keystone.conf

修改[database]部分实现与数据库的连接
connection = mysql+pymysql://keystone:123456@controller/keystone
修改[token]配置令牌加密方式
provider = fernet
```

> keystone加密方式一共3种：UUID、PKI、Fernet

> Fernet Token 当前主流令牌加密格式，是一种轻量级的消息格式，存储形式为字符串，包含用户信息、权限信息、过期时间信息等，在验证时将用解密密钥对其进行解密以获得原始信息

同步数据库

```jsx
su keystone -s /bin/sh -c "keystone-manage db_sync"
```

> su 切换keystone用户执行命令

> \-s 指定使用的编译器

> \-c 执行引号内的命令

检查数据库

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 12
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

查看已有数据库
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.000 sec)

使用‘keystone’数据库
MariaDB [(none)]> use keystone;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed

查看该数据库中的所有表
MariaDB [keystone]> show tables;
+------------------------------------+
| Tables_in_keystone                 |
+------------------------------------+
| access_rule                        |
| access_token                       |
| application_credential             |
| application_credential_access_rule |
| application_credential_role        |
| assignment                         |
| config_register                    |
| consumer                           |
| credential                         |
| endpoint                           |
| endpoint_group                     |
| federated_user                     |
| federation_protocol                |
| group                              |
| id_mapping                         |
| identity_provider                  |
| idp_remote_ids                     |
| implied_role                       |
| limit                              |
| local_user                         |
| mapping                            |
| migrate_version                    |
| nonlocal_user                      |
| password                           |
| policy                             |
| policy_association                 |
| project                            |
| project_endpoint                   |
| project_endpoint_group             |
| project_option                     |
| project_tag                        |
| region                             |
| registered_limit                   |
| request_token                      |
| revocation_event                   |
| role                               |
| role_option                        |
| sensitive_config                   |
| service                            |
| service_provider                   |
| system_assignment                  |
| token                              |
| trust                              |
| trust_role                         |
| user                               |
| user_group_membership              |
| user_option                        |
| whitelisted_config                 |
+------------------------------------+
48 rows in set (0.001 sec)

查看远程‘keystone’用户是否授予对‘keystone’数据库的完全控制权限
MariaDB [keystone]> show grants for 'keystone'@'%';
+---------------------------------------------------------------------------------------------------------+
| Grants for keystone@%                                                                                   |
+---------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'keystone'@'%' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9' |
| GRANT ALL PRIVILEGES ON `keystone`.* TO 'keystone'@'%'                                                  |
+---------------------------------------------------------------------------------------------------------+
2 rows in set (0.000 sec)

查看本地‘keystone’用户是否授予对‘keystone’数据库的完全控制权限
MariaDB [keystone]> show grants for 'keystone'@'localhost';
+-----------------------------------------------------------------------------------------------------------------+
| Grants for keystone@localhost                                                                                   |
+-----------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'keystone'@'localhost' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9' |
| GRANT ALL PRIVILEGES ON `keystone`.* TO 'keystone'@'localhost'                                                  |
+-----------------------------------------------------------------------------------------------------------------+
2 rows in set (0.000 sec)

MariaDB [keystone]> quit
Bye
```

初始化fernet秘钥库

```jsx
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
```

> 会自动创建/etc/keystone/fernet-keys目录，并生成两个fernet秘钥

验证

> 会自动创建/etc/keystone/credential-keys该目录，并生成两个fernet秘钥
>
> ```jsx
> [root@controller ~]# ls /etc/keystone/fernet-keys/
> 0  1  
> ```

```jsx
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

验证

```jsx
[root@controller ~]# ls /etc/keystone/credential-keys/
0  1
```

初始化用户身份认证信息

```jsx
keystone-manage bootstrap --bootstrap-password 123456 --bootstrap-admin-url http://controller:5000/v3 --bootstrap-internal-url http://controller:5000/v3 --bootstrap-public-url http://controller:5000/v3 --bootstrap-region-id RegionOne
```

> \--bootstrap-username 设置登录用户名，如果没有此参数则默认登录用户为“admin”用户

> \--bootstrap-password 设置用户的密码

> \--bootstrap-admin-url 设置admin用户使用的服务端点

> \--bootstrap-internal-url 设置内部用户使用的服务端点

> \--bootstrap-public-url 设置公共用户使用的服务端点

> \--bootstrap-region-id 设置区域id名称，用于配置集群服务

为apache服务添加wsgi支持（软链接映射文件）

```jsx
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

> 使wsgi-keystone.conf 文件可以成为apache服务器的配置文件之一，从而时apache服务器可以安装的“mod\_wsgi”插件支持WSGI服务

修改apache服务器的配置文件

```jsx
vi /etc/httpd/conf/httpd.conf

将其设置为web服务器所在的域名或IP地址，表示将其设置为本机域名
ServerName controller
```

重启apache服务并设置为开机自启动

```jsx
systemctl restart httpd && systemctl enable httpd
```

创建初始化环境变量文件(新建一个文件用于存储身份凭证)

```jsx
vi admin-login

定义云平台的用户名
export OS_USERNAME=admin
定义云平台的用户名的登录密码
export OS_PASSWORD=123456
定义云平台的项目名
export OS_PROJECT_NAME=admin
定义用户属于的域的名称
export OS_USER_DOMAIN_NAME=Default
定义项目属于的域的名称
export OS_PROJECT_DOMAIN_NAME=Default
定义认证地址
export OS_AUTH_URL=http://controller:5000/v3
定义Keystone的版本号
export OS_IDENTITY_API_VERSION=3
定义镜像管理应用的版本号
export OS_IMAGE_API_VERSION=2
```

导入环境变量进行验证

```jsx
source admin-login
```

查看现有的环境变量(export -p)

```jsx
export -p

declare -x HISTCONTROL="ignoredups"
declare -x HISTSIZE="1000"
declare -x HOME="/root"
declare -x HOSTNAME="controller"
declare -x LANG="en_US.UTF-8"
declare -x LESSOPEN="||/usr/bin/lesspipe.sh %s"
declare -x LOGNAME="root"
declare -x LS_COLORS="rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=01;05;37;41:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arc=01;31:*.arj=01;31:*.taz=01;31:*.lha=01;31:*.lz4=01;31:*.lzh=01;31:*.lzma=01;31:*.tlz=01;31:*.txz=01;31:*.tzo=01;31:*.t7z=01;31:*.zip=01;31:*.z=01;31:*.Z=01;31:*.dz=01;31:*.gz=01;31:*.lrz=01;31:*.lz=01;31:*.lzo=01;31:*.xz=01;31:*.bz2=01;31:*.bz=01;31:*.tbz=01;31:*.tbz2=01;31:*.tz=01;31:*.deb=01;31:*.rpm=01;31:*.jar=01;31:*.war=01;31:*.ear=01;31:*.sar=01;31:*.rar=01;31:*.alz=01;31:*.ace=01;31:*.zoo=01;31:*.cpio=01;31:*.7z=01;31:*.rz=01;31:*.cab=01;31:*.jpg=01;35:*.jpeg=01;35:*.gif=01;35:*.bmp=01;35:*.pbm=01;35:*.pgm=01;35:*.ppm=01;35:*.tga=01;35:*.xbm=01;35:*.xpm=01;35:*.tif=01;35:*.tiff=01;35:*.png=01;35:*.svg=01;35:*.svgz=01;35:*.mng=01;35:*.pcx=01;35:*.mov=01;35:*.mpg=01;35:*.mpeg=01;35:*.m2v=01;35:*.mkv=01;35:*.webm=01;35:*.ogm=01;35:*.mp4=01;35:*.m4v=01;35:*.mp4v=01;35:*.vob=01;35:*.qt=01;35:*.nuv=01;35:*.wmv=01;35:*.asf=01;35:*.rm=01;35:*.rmvb=01;35:*.flc=01;35:*.avi=01;35:*.fli=01;35:*.flv=01;35:*.gl=01;35:*.dl=01;35:*.xcf=01;35:*.xwd=01;35:*.yuv=01;35:*.cgm=01;35:*.emf=01;35:*.axv=01;35:*.anx=01;35:*.ogv=01;35:*.ogx=01;35:*.aac=01;36:*.au=01;36:*.flac=01;36:*.mid=01;36:*.midi=01;36:*.mka=01;36:*.mp3=01;36:*.mpc=01;36:*.ogg=01;36:*.ra=01;36:*.wav=01;36:*.axa=01;36:*.oga=01;36:*.spx=01;36:*.xspf=01;36:"
declare -x MAIL="/var/spool/mail/root"
declare -x OLDPWD
declare -x OS_AUTH_URL="http://controller:5000/v3"
declare -x OS_IDENTITY_API_VERSION="3"
declare -x OS_IMAGE_API_VERSION="2"
declare -x OS_PASSWORD="123456"
declare -x OS_PROJECT_DOMAIN_NAME="Default"
declare -x OS_PROJECT_NAME="admin"
declare -x OS_USERNAME="admin"
declare -x OS_USER_DOMAIN_NAME="Default"
declare -x PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin"
declare -x PWD="/root"
declare -x SELINUX_LEVEL_REQUESTED=""
declare -x SELINUX_ROLE_REQUESTED=""
declare -x SELINUX_USE_CURRENT_RANGE=""
declare -x SHELL="/bin/bash"
declare -x SHLVL="1"
declare -x SSH_CLIENT="192.168.100.1 8304 22"
declare -x SSH_CONNECTION="192.168.100.1 8304 192.168.100.100 22"
declare -x SSH_TTY="/dev/pts/0"
declare -x TERM="xterm"
declare -x USER="root"
declare -x XDG_RUNTIME_DIR="/run/user/0"
declare -x XDG_SESSION_ID="1"
```

创建名为“project”的项目

```jsx
openstack project create --domain default project

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description |                                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | b8b677d25ac04cdd950c0782050aed60 |
| is_domain   | False                            |
| name        | project                          |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

查看OpenStack中现有的项目列表

```jsx
openstack project list

+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| a9001f880f424f79929111ef200030fc | admin   |
| b8b677d25ac04cdd950c0782050aed60 | project |
+----------------------------------+---------+
```

创建名为“user”的角色

```jsx
openstack role create user

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| domain_id   | None                             |
| id          | 4fb78ac4adff43ef8127afdf81f84694 |
| name        | user                             |
| options     | {}                               |
+-------------+----------------------------------+
```

查看OpenStack中现有的角色列表

```jsx
openstack role list

+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| 4fb78ac4adff43ef8127afdf81f84694 | user   |
| 57adb6fe8ac04387853003b18c9c7240 | member |
| a794d5d1b9014757b8e61e7c97f0ff27 | admin  |
| dd525862a0ba4b3090b00e18436abdfd | reader |
+----------------------------------+--------+
```

查看OpenStack中现有的域列表

```jsx
openstack domain list

]+---------+---------+---------+--------------------+
| ID      | Name    | Enabled | Description        |
+---------+---------+---------+--------------------+
| default | Default | True    | The default domain |
+---------+---------+---------+--------------------+
```

查看OpenStack中现有的用户列表

```jsx
openstack user list

+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| fcc523d60cc74e19ad13d4f7b029df70 | admin |
+----------------------------------+-------+
```

查看keystone服务端是否运行正常

```jsx
curl http://controller:5000

{"versions": {"values": [{"status": "stable", "updated": "2019-07-19T00:00:00Z", "media-types": [{"base": "application/json", "type": "application/vnd.openstack.identity-v3+json"}], "id": "v3.13", "links": [{"href": "http://controller:5000/v3/", "rel": "self"}]}]}}
```

### Glance服务

安装glance服务

```jsx
yum install -y openstack-glance
```

查看用户信息

```jsx
[root@controller ~]# cat /etc/passwd | grep glance
glance:x:161:161:OpenStack Glance Daemons:/var/lib/glance:/sbin/nologin
```

查看用户组信息

```jsx
[root@controller ~]# cat /etc/group | grep glance
glance:x:161:
```

进入MariaDB数据库服务器

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 19
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

创建一个名为“glance”的数据库
MariaDB [(none)]> CREATE DATABASE glance;
Query OK, 1 row affected (0.003 sec)

将新建的数据库管理权限授予本地登录和远程登录的“glance”用户：%为远程主机
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.007 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> QUIT
Bye
```

备份配置文件

```jsx
cp /etc/glance/glance-api.conf /etc/glance/glance-api.conf.bak
```

去掉配置文件中的注释行和空行，并生成新的配置文件

```jsx
grep -Ev '^$|#' /etc/glance/glance-api.conf.bak > /etc/glance/glance-api.conf
```

> E表示采用正则表达式进行读取；^$|#（^$ ^表示一行的开头，$表示一行的结尾，整体表示为匹配空行；| 这里表示或者； #表示注释行）v表示反向匹配

```jsx
vi /etc/glance/glance-api.conf

[DEFAULT]
[cinder]
[cors]
实现与数据库的连接
mysql+pymysql://: 使用 PyMySQL 作为驱动程序连接 MySQL 数据库。
glance:123456: 数据库用户名和密码。
controller: 数据库主机名。
/glance: 数据库名称。
[database]
connection = mysql+pymysql://glance:123456@controller/glance
[file]
[glance.store.http.store]
[glance.store.rbd.store]
[glance.store.sheepdog.store]
[glance.store.swift.store]
[glance.store.vmware_datastore.store]
指定后端存储系统，默认使用文件存储
stores: 可用的存储系统，这里只配置了 file。
default_store: 默认存储系统，这里设置为 file。
filesystem_store_datadir: 文件存储的数据目录。
[glance_store]
stores = file
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
[image_format]
实现与keystone的交互
auth_url: Keystone 的认证 URL。
memcached_servers: Memcached 服务器地址，用于缓存认证信息。
auth_type: 认证类型，这里使用密码认证。
username 和 password: Keystone 中 Glance 服务的用户名和密码。
project_name: 项目名称。
user_domain_name 和 project_domain_name: 用户和项目的域名，默认值为 Default。
[keystone_authtoken]
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
username = glance
password = 123456
project_name = project
user_domain_name = Default
project_domain_name = Default
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
flavor：定义虚拟机实例的硬件配置模板。
flavor = keystone
[profiler]
[store_type_location_strategy]
[task]
[taskflow_executor]
```

同步数据库：

```jsx
su glance -s /bin/sh -c"glance-manage db_sync"

INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
/usr/lib/python2.7/site-packages/pymysql/cursors.py:170: Warning: (1280, u"Name 'alembic_version_pkc' ignored for PRIMARY key.")
  result = self._query(query)
INFO  [alembic.runtime.migration] Running upgrade  -> liberty, liberty initial
INFO  [alembic.runtime.migration] Running upgrade liberty -> mitaka01, add index on created_at and updated_at columns of 'images' table
INFO  [alembic.runtime.migration] Running upgrade mitaka01 -> mitaka02, update metadef os_nova_server
INFO  [alembic.runtime.migration] Running upgrade mitaka02 -> ocata_expand01, add visibility to images
INFO  [alembic.runtime.migration] Running upgrade ocata_expand01 -> pike_expand01, empty expand for symmetry with pike_contract01
INFO  [alembic.runtime.migration] Running upgrade pike_expand01 -> queens_expand01
INFO  [alembic.runtime.migration] Running upgrade queens_expand01 -> rocky_expand01, add os_hidden column to images table
INFO  [alembic.runtime.migration] Running upgrade rocky_expand01 -> rocky_expand02, add os_hash_algo and os_hash_value columns to images table
INFO  [alembic.runtime.migration] Running upgrade rocky_expand02 -> train_expand01, empty expand for symmetry with train_contract01
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
Upgraded database to: train_expand01, current revision(s): train_expand01
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
Database migration is up to date. No migration needed.
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade mitaka02 -> ocata_contract01, remove is_public from images
INFO  [alembic.runtime.migration] Running upgrade ocata_contract01 -> pike_contract01, drop glare artifacts tables
INFO  [alembic.runtime.migration] Running upgrade pike_contract01 -> queens_contract01
INFO  [alembic.runtime.migration] Running upgrade queens_contract01 -> rocky_contract01
INFO  [alembic.runtime.migration] Running upgrade rocky_contract01 -> rocky_contract02
INFO  [alembic.runtime.migration] Running upgrade rocky_contract02 -> train_contract01
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
Upgraded database to: train_contract01, current revision(s): train_contract01
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
Database is synced successfully.
```

检查数据库

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 22
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

转换到“glance”的数据库中
MariaDB [(none)]> use glance;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed

查看当前数据库中的所有表
MariaDB [glance]> show tables;
+----------------------------------+
| Tables_in_glance                 |
+----------------------------------+
| alembic_version                  |
| image_locations                  |
| image_members                    |
| image_properties                 |
| image_tags                       |
| images                           |
| metadef_namespace_resource_types |
| metadef_namespaces               |
| metadef_objects                  |
| metadef_properties               |
| metadef_resource_types           |
| metadef_tags                     |
| migrate_version                  |
| task_info                        |
| tasks                            |
+----------------------------------+
15 rows in set (0.000 sec)

查看远程‘keystone’用户是否授予对‘keystone’数据库的完全控制权限
MariaDB [glance]> show grants for 'glance'@'%';
+-------------------------------------------------------------------------------------------------------+
| Grants for glance@%                                                                                   |
+-------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'glance'@'%' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9' |
| GRANT ALL PRIVILEGES ON `glance`.* TO 'glance'@'%'                                                    |
+-------------------------------------------------------------------------------------------------------+
2 rows in set (0.000 sec)

查看本地‘keystone’用户是否授予对‘keystone’数据库的完全控制权限
MariaDB [glance]> show grants for 'glance'@'localhost';
+---------------------------------------------------------------------------------------------------------------+
| Grants for glance@localhost                                                                                   |
+---------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'glance'@'localhost' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9' |
| GRANT ALL PRIVILEGES ON `glance`.* TO 'glance'@'localhost'                                                    |
+---------------------------------------------------------------------------------------------------------------+
2 rows in set (0.000 sec)

MariaDB [glance]> quit
Bye
```

导入环境变量进行验证

```jsx
source admin-login
```

查看现有的环境变量(export -p)

```jsx
declare -x HISTCONTROL="ignoredups"
declare -x HISTSIZE="1000"
declare -x HOME="/root"
declare -x HOSTNAME="controller"
declare -x LANG="en_US.UTF-8"
declare -x LESSOPEN="||/usr/bin/lesspipe.sh %s"
declare -x LOGNAME="root"
declare -x LS_COLORS="rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=01;05;37;41:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arc=01;31:*.arj=01;31:*.taz=01;31:*.lha=01;31:*.lz4=01;31:*.lzh=01;31:*.lzma=01;31:*.tlz=01;31:*.txz=01;31:*.tzo=01;31:*.t7z=01;31:*.zip=01;31:*.z=01;31:*.Z=01;31:*.dz=01;31:*.gz=01;31:*.lrz=01;31:*.lz=01;31:*.lzo=01;31:*.xz=01;31:*.bz2=01;31:*.bz=01;31:*.tbz=01;31:*.tbz2=01;31:*.tz=01;31:*.deb=01;31:*.rpm=01;31:*.jar=01;31:*.war=01;31:*.ear=01;31:*.sar=01;31:*.rar=01;31:*.alz=01;31:*.ace=01;31:*.zoo=01;31:*.cpio=01;31:*.7z=01;31:*.rz=01;31:*.cab=01;31:*.jpg=01;35:*.jpeg=01;35:*.gif=01;35:*.bmp=01;35:*.pbm=01;35:*.pgm=01;35:*.ppm=01;35:*.tga=01;35:*.xbm=01;35:*.xpm=01;35:*.tif=01;35:*.tiff=01;35:*.png=01;35:*.svg=01;35:*.svgz=01;35:*.mng=01;35:*.pcx=01;35:*.mov=01;35:*.mpg=01;35:*.mpeg=01;35:*.m2v=01;35:*.mkv=01;35:*.webm=01;35:*.ogm=01;35:*.mp4=01;35:*.m4v=01;35:*.mp4v=01;35:*.vob=01;35:*.qt=01;35:*.nuv=01;35:*.wmv=01;35:*.asf=01;35:*.rm=01;35:*.rmvb=01;35:*.flc=01;35:*.avi=01;35:*.fli=01;35:*.flv=01;35:*.gl=01;35:*.dl=01;35:*.xcf=01;35:*.xwd=01;35:*.yuv=01;35:*.cgm=01;35:*.emf=01;35:*.axv=01;35:*.anx=01;35:*.ogv=01;35:*.ogx=01;35:*.aac=01;36:*.au=01;36:*.flac=01;36:*.mid=01;36:*.midi=01;36:*.mka=01;36:*.mp3=01;36:*.mpc=01;36:*.ogg=01;36:*.ra=01;36:*.wav=01;36:*.axa=01;36:*.oga=01;36:*.spx=01;36:*.xspf=01;36:"
declare -x MAIL="/var/spool/mail/root"
declare -x OLDPWD
declare -x OS_AUTH_URL="http://controller:5000/v3"
declare -x OS_IDENTITY_API_VERSION="3"
declare -x OS_IMAGE_API_VERSION="2"
declare -x OS_PASSWORD="123456"
declare -x OS_PROJECT_DOMAIN_NAME="Default"
declare -x OS_PROJECT_NAME="admin"
declare -x OS_USERNAME="admin"
declare -x OS_USER_DOMAIN_NAME="Default"
declare -x PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin"
declare -x PWD="/root"
declare -x SELINUX_LEVEL_REQUESTED=""
declare -x SELINUX_ROLE_REQUESTED=""
declare -x SELINUX_USE_CURRENT_RANGE=""
declare -x SHELL="/bin/bash"
declare -x SHLVL="1"
declare -x SSH_CLIENT="192.168.100.1 8304 22"
declare -x SSH_CONNECTION="192.168.100.1 8304 192.168.100.100 22"
declare -x SSH_TTY="/dev/pts/0"
declare -x TERM="xterm"
declare -x USER="root"
declare -x XDG_RUNTIME_DIR="/run/user/0"
declare -x XDG_SESSION_ID="1"
```

在“default”域中创建用户glance密码为‘123456’

```jsx
openstack user create --domain default --password 123456 glance

+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 7c4087ac47184343997d023de6d357d4 |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

为“glance”用户分配“admin”角色(授予了glance用户操作project项目时的admin权限)

```jsx
openstack role add --project project --user glance admin
```

创建一个名为“glance”类型为“image”的服务

```jsx
openstack service create --name glance image

+---------+----------------------------------+
| Field   | Value                            |
+---------+----------------------------------+
| enabled | True                             |
| id      | b645abce902b46f9b1b4f39a80601cb1 |
| name    | glance                           |
| type    | image                            |
+---------+----------------------------------+
```

创建公众用户访问的服务端点

```jsx
openstack endpoint create --region RegionOne glance public http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 484f573d093545048cfe6d7de14c7609 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b645abce902b46f9b1b4f39a80601cb1 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

创建内部组件访问的服务端点

```jsx
openstack endpoint create --region RegionOne glance internal http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6770c81a2b4e4de98ce89ff16ff69bd1 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b645abce902b46f9b1b4f39a80601cb1 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

创建内部组件访问的服务端点

```jsx
openstack endpoint create --region RegionOne glance admin http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | fc5b67bed2304918bdf51630d6fc09f5 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b645abce902b46f9b1b4f39a80601cb1 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

启动glance服务并设置为开机自启动

```jsx
systemctl start openstack-glance-api && systemctl enable openstack-glance-api
```

查看端口占用情况

```jsx
[root@controller ~]# netstat -tnlup|grep 9292
tcp        0      0 0.0.0.0:9292            0.0.0.0:*               LISTEN      5923/python2 
```

查看glance服务运行状态

```jsx
[root@controller ~]# systemctl status openstack-glance-api
● openstack-glance-api.service - OpenStack Image Service (code-named Glance) API server
   Loaded: loaded (/usr/lib/systemd/system/openstack-glance-api.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2024-05-25 05:08:28 PDT; 1min 0s ago
 Main PID: 5923 (glance-api)
   CGroup: /system.slice/openstack-glance-api.service
           ├─5923 /usr/bin/python2 /usr/bin/glance-api
           ├─5962 /usr/bin/python2 /usr/bin/glance-api
           └─5963 /usr/bin/python2 /usr/bin/glance-api

May 25 05:08:28 controller systemd[1]: Started OpenStack Image Service (code-named Glance) API server.
May 25 05:08:30 controller glance-api[5923]: /usr/lib/python2.7/site-packages/paste/deploy/loadwsgi.py:22: PkgResourcesDeprecationWarning...arately.
May 25 05:08:30 controller glance-api[5923]: return pkg_resources.EntryPoint.parse("x=" + s).load(False)
Hint: Some lines were ellipsized, use -l to show in full.
```

调用glance创建一个镜像（qcow2 为生成镜像磁盘的格式；bare 为容器格式）

```jsx
openstack image create --file cirros-0.5.1-x86_64-disk.img --disk-format qcow2 --container-format bare --public cirros

+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

| Field | Value |

+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

| checksum | 1d3062cd89af34e419f7100277f38b2b |

| container_format | bare |

| created_at | 2023-11-01T08:42:05Z |

| disk_format | qcow2 |

| file | /v2/images/d8e1e740-1c7e-474e-99df-8dc3c154a2ec/file |

| id | d8e1e740-1c7e-474e-99df-8dc3c154a2ec |

| min_disk | 0 |

| min_ram | 0 |

| name | cirros |

| owner | f1f032530bd44b0ab9382f4504edb232 |

| properties | os_hash_algo='sha512', os_hash_value='553d220ed58cfee7dafe003c446a9f197ab5edf8ffc09396c74187cf83873c877e7ae041cb80f3b91489acf687183adcd689b53b38e3ddd22e627e7f98a09c46', os_hidden='False' |

| protected | False |

| schema | /v2/schemas/image |

| size | 16338944 |

| status | active |

| tags | |

| updated_at | 2023-11-01T08:42:09Z |

| virtual_size | None |

| visibility | public |

+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

查看镜像列表【镜像状态（Status ）为可用（active ）状态】

```jsx
openstack image list

+--------------------------------------+--------+--------+

| ID | Name | Status |

+--------------------------------------+--------+--------+

| d8e1e740-1c7e-474e-99df-8dc3c154a2ec | cirros | active |

+--------------------------------------+--------+--------+
```

查看镜像的物理文件

```jsx
ll /var/lib/glance/images/

total 15956

- rw-r-----. 1 glance glance 16338944 Nov 1 04:42 d8e1e740-1c7e-474e-99df-8dc3c154a2ec
```

自检

```jsx
查看用户列表是否存在glance
openstack user list

+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| fcc523d60cc74e19ad13d4f7b029df70 | admin  |
| 7c4087ac47184343997d023de6d357d4 | glance |
+----------------------------------+--------+

查看角色ID
openstack role list

+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| 4fb78ac4adff43ef8127afdf81f84694 | user   |
| 57adb6fe8ac04387853003b18c9c7240 | member |
| a794d5d1b9014757b8e61e7c97f0ff27 | admin  |
| dd525862a0ba4b3090b00e18436abdfd | reader |
+----------------------------------+--------+

查看glane用户是否和admin角色绑定
openstack role assignment list

+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+
| Role                             | User                             | Group | Project                          | Domain | System | Inherited |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+
| a794d5d1b9014757b8e61e7c97f0ff27 | 7c4087ac47184343997d023de6d357d4 |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | fcc523d60cc74e19ad13d4f7b029df70 |       | a9001f880f424f79929111ef200030fc |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | fcc523d60cc74e19ad13d4f7b029df70 |       |                                  |        | all    | False     |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+

查看服务列表
openstack service list

+----------------------------------+----------+----------+
| ID                               | Name     | Type     |
+----------------------------------+----------+----------+
| 4a1a975ce099473a9f8a0b56ba24bbcc | keystone | identity |
| b645abce902b46f9b1b4f39a80601cb1 | glance   | image    |
+----------------------------------+----------+----------+

glance的3个服务端点是否建立
openstack endpoint list

+----------------------------------+-----------+--------------+--------------+---------+-----------+---------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                       |
+----------------------------------+-----------+--------------+--------------+---------+-----------+---------------------------+
| 2330a6b94c92439b89cd64e40b4fce61 | RegionOne | keystone     | identity     | True    | admin     | http://contrpller:5000/v3 |
| 484f573d093545048cfe6d7de14c7609 | RegionOne | glance       | image        | True    | public    | http://controller:9292    |
| 6770c81a2b4e4de98ce89ff16ff69bd1 | RegionOne | glance       | image        | True    | internal  | http://controller:9292    |
| 91b53fd5293b44889afc309a7c676e41 | RegionOne | keystone     | identity     | True    | internal  | http://controller:5000/v3 |
| 925e27d07cb248489a0938626bad1d0a | RegionOne | keystone     | identity     | True    | public    | http://controller:5000/v3 |
| fc5b67bed2304918bdf51630d6fc09f5 | RegionOne | glance       | image        | True    | admin     | http://controller:9292    |
+----------------------------------+-----------+--------------+--------------+---------+-----------+---------------------------+

查看keystone服务端是否运行正常
curl controller:9292

{"versions": [{"status": "CURRENT", "id": "v2.9", "links": [{"href": "http://controller:9292/v2/", "rel": "self"}]}, {"status": "SUPPORTED", "id": "v2.7", "links": [{"href": "http://controller:9292/v2/", "rel": "self"}]}, {"status": "SUPPORTED", "id": "v2.6", "links": [{"href": "http://controller:9292/v2/", "rel": "self"}]}, {"status": "SUPPORTED", "id": "v2.5", "links": [{"href": "http://controller:9292/v2/", "rel": "self"}]}, {"status": "SUPPORTED", "id": "v2.4", "links": [{"href": "http://controller:9292/v2/", "rel": "self"}]}, {"status": "SUPPORTED", "id": "v2.3", "links": [{"href": "http://controller:9292/v2/", "rel": "self"}]}, {"status": "SUPPORTED", "id": "v2.2", "links": [{"href": "http://controller:9292/v2/", "rel": "self"}]}, {"status": "SUPPORTED", "id": "v2.1", "links": [{"href": "http://controller:9292/v2/", "rel": "self"}]}, {"status": "SUPPORTED", "id": "v2.0", "links": [{"href": "http://controller:9292/v2/", "rel": "self"}]}]}
```

### Placement服务

安装placement服务

```jsx
yum install -y openstack-placement-api
```

查看用户信息

```jsx
[root@controller ~]# cat /etc/passwd | grep Placement
placement:x:993:990:OpenStack Placement:/:/bin/bash
```

查看用户组信息

```jsx
[root@controller ~]# cat /etc/group | grep placement
placement:x:990:
```

进入MariaDB数据库服务器

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 8
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

创建一个名为“placement”的数据库
MariaDB [(none)]> CREATE DATABASE placement;
Query OK, 1 row affected (0.001 sec)

将新建的数据库管理权限授予本地登录和远程登录的“placement”用户：%为远程主机
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.002 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> QUIT
Bye
```

备份主配置文件

```jsx
cp /etc/placement/placement.conf /etc/placement/placement.conf.bak
```

去掉配置文件中的注释行和空行，并生成新的配置文件

```jsx
grep -Ev '^$|#' /etc/placement/placement.conf.bak > /etc/placement/placement.conf
```

> E表示采用正则表达式进行读取；^$|#（^$ ^表示一行的开头，$表示一行的结尾，整体表示为匹配空行；| 这里表示或者； #表示注释行）v表示反向匹配

编辑新的配置文件

```jsx
vi /etc/placement/placement.conf

[DEFAULT]
**[DEFAULT]**: 这个部分包含 OpenStack 服务的默认配置
[api]
**[api]**: 这个部分包含 API 相关的配置。`auth_strategy=keystone`: 指定使用 Keystone 作为认证策略。Keystone 是 OpenStack 的身份认证服务。
auth_strategy=keystone
[cors]
**[cors]**: 这个部分通常用于配置跨域资源共享 (CORS) 相关的设置
[keystone_authtoken]
auth_url=http://controller:5000
memcached_servers=controller:11211
auth_type=password
project_domain_name=Default
user_domain_name=Default
project_name=project
username=placement
password=123456
**[keystone_authtoken]**: 这个部分包含与 Keystone 认证服务交互的设置。
- `auth_url=http://controller:5000`: 指定 Keystone 认证服务的 URL。
- `memcached_servers=controller:11211`: 指定用于缓存认证令牌的 Memcached 服务器地址。
- `auth_type=password`: 指定使用密码认证。
- `project_domain_name=Default`: 指定项目域的名称为 "Default"。
- `user_domain_name=Default`: 指定用户域的名称为 "Default"。
- `project_name=project`: 指定项目名称为 "project"。
- `username=placement`: Keystone 用户名。
- `password=123456`: Keystone 用户的密码。
[oslo_policy]
**[oslo_policy]**: 这个部分用于配置与策略相关的设置。
[placement]
**[placement]**: 这个部分包含 Placement 服务的特定配置
[placement_database]
connection=mysql+pymysql://plancement:123456@controller/placement
**[placement_database]**: 这个部分包含 Placement 服务的数据库连接配置,可以实现与数据库的连接
- `connection=mysql+pymysql://plancement:123456@controller/plancement`: 指定用于连接数据库的 URI。使用 PyMySQL 驱动连接到 MySQL 数据库。URI 的组成部分包括：
  - `mysql+pymysql`: 指定使用 MySQL 数据库和 PyMySQL 驱动。
  - `plancement:123456`: 数据库用户名为 `plancement`，密码为 `123456`。
  - `controller`: 数据库主机名为 `controller`。
  - `/plancement`: 数据库名称为 `plancement`。
[profiler]
**[profiler]**: 这个部分通常用于配置性能分析和调试相关的设置。
```

编辑Apache的配置文件

```jsx

vi /etc/httpd/conf.d/00-placement-api.conf

Listen 8778

<VirtualHost *:8778>
  WSGIProcessGroup placement-api
  WSGIApplicationGroup %{GLOBAL}
  WSGIPassAuthorization On
  WSGIDaemonProcess placement-api processes=3 threads=1 user=placement group=placement
  WSGIScriptAlias / /usr/bin/placement-api
  <IfVersion >= 2.4>
    ErrorLogFormat "%M"
  </IfVersion>
  <Directory /usr/bin>
    Require all granted
  </Directory>
使得 Apache 服务器允许任何人访问 `/usr/bin` 目录中的文件
- `<Directory /usr/bin>`: 这行表示开始配置位于 `/usr/bin` 目录下的资源的访问控制。
  - **/usr/bin**: 这是一个目录路径。在这个目录下包含了很多可执行文件和脚本。
- `Require all granted`: 这行表示允许所有用户访问 `/usr/bin` 目录中的资源。
  - **Require**: 这是一个指令，用于指定访问控制要求。
  - **all granted**: 这是一个访问控制表达式，表示无论是谁都被允许访问。
  ErrorLog /var/log/placement/placement-api.log
  #SSLEngine On
  #SSLCertificateFile ...
  #SSLCertificateKeyFile ...
</VirtualHost>

Alias /placement-api /usr/bin/placement-api
<Location /placement-api>
  SetHandler wsgi-script
  Options +ExecCGI
  WSGIProcessGroup placement-api
  WSGIApplicationGroup %{GLOBAL}
  WSGIPassAuthorization On
</Location>
```

查看Apache的版本号

```jsx
httpd -v

Server version: Apache/2.4.6 (CentOS)
Server built:   May 30 2023 14:01:11
```

同步数据库

```jsx
su placement -s /bin/sh -c "placement-manage db sync"
```

进入数据库

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 10
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

转换到“placement”的数据库中
MariaDB [(none)]> use placement;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
查看当前数据库中的所有表
MariaDB [placement]> show tables;
+------------------------------+
| Tables_in_placement          |
+------------------------------+
| alembic_version              |
| allocations                  |
| consumers                    |
| inventories                  |
| placement_aggregates         |
| projects                     |
| resource_classes             |
| resource_provider_aggregates |
| resource_provider_traits     |
| resource_providers           |
| traits                       |
| users                        |
+------------------------------+
12 rows in set (0.000 sec)

MariaDB [placement]> quit
Bye
```

导入环境变量模拟登陆（可以用source或者.）

```jsx
source admin-login
```

在“default”域中创建用户placement

```jsx
openstack user create --domain default --password 123456 placement

+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 20f8cd5b95dc44cf98b43348b7f6124c |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

为“placement”用户分配“admin”角色

```jsx
openstack role add --project project --user placement admin
```

创建一个名为“placement”类型为“placement”的服务

```jsx
openstack service create --name placement placement

+---------+----------------------------------+
| Field   | Value                            |
+---------+----------------------------------+
| enabled | True                             |
| id      | f517d649ec4b4f58996d800679698090 |
| name    | placement                        |
| type    | placement                        |
+---------+----------------------------------+
```

创建公众用户访问的服务端点

```jsx
openstack endpoint create --region RegionOne placement public http://controller:8778

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 41d7666d90de4d0fafaadf0422dc3432 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f517d649ec4b4f58996d800679698090 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

创建内部组件访问的服务端点

```jsx
openstack endpoint create --region RegionOne placement internal http://controller:8778

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 7a5b6a7987884264bde8dcbbc746db9b |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f517d649ec4b4f58996d800679698090 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

创建admin用户访问的服务端点

```jsx
openstack endpoint create --region RegionOne placement admin http://controller:8778

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 5eaacaa4019e49db865a97e9a728e133 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f517d649ec4b4f58996d800679698090 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

启动并设置为开机自启动placement服务(需要借助Apache的web服务实现功能)

```jsx
systemctl restart httpd && systemctl enable httpd
```

查看端口的占用情况

```jsx
netstat -tnlup | grep 8778

tcp6       0      0 :::8778                 :::*                    LISTEN      2267/httpd        
```

检验服务的端点

```jsx
curl http://controller:8778

{"versions": [{"status": "CURRENT", "min_version": "1.0", "max_version": "1.36", "id": "v1.0", "links": [{"href": "", "rel": "self"}]}]}
```

自检

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 17
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

查看远程‘keystone’用户是否授予对‘keystone’数据库的完全控制权限
MariaDB [(none)]> show grants for 'placement'@'%';
+----------------------------------------------------------------------------------------------------------+
| Grants for placement@%                                                                                   |
+----------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'placement'@'%' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9' |
| GRANT ALL PRIVILEGES ON `placement`.* TO 'placement'@'%'                                                 |
+----------------------------------------------------------------------------------------------------------+
2 rows in set (0.000 sec)

查看本地‘keystone’用户是否授予对‘keystone’数据库的完全控制权限
MariaDB [(none)]> show grants for 'placement'@'localhost';
+------------------------------------------------------------------------------------------------------------------+
| Grants for placement@localhost                                                                                   |
+------------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'placement'@'localhost' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9' |
| GRANT ALL PRIVILEGES ON `placement`.* TO 'placement'@'localhost'                                                 |
+------------------------------------------------------------------------------------------------------------------+
2 rows in set (0.000 sec)

MariaDB [(none)]> quit
Bye

查看用户列表是否存在placement 
[root@controller ~]# openstack user list

+----------------------------------+-----------+
| ID                               | Name      |
+----------------------------------+-----------+
| fcc523d60cc74e19ad13d4f7b029df70 | admin     |
| 7c4087ac47184343997d023de6d357d4 | glance    |
| 20f8cd5b95dc44cf98b43348b7f6124c | placement |
+----------------------------------+-----------+

查看角色ID
[root@controller ~]# openstack role list

+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| 4fb78ac4adff43ef8127afdf81f84694 | user   |
| 57adb6fe8ac04387853003b18c9c7240 | member |
| a794d5d1b9014757b8e61e7c97f0ff27 | admin  |
| dd525862a0ba4b3090b00e18436abdfd | reader |
+----------------------------------+--------+

查看placement用户是否和admin角色绑定
[root@controller ~]# openstack role assignment list

+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+
| Role                             | User                             | Group | Project                          | Domain | System | Inherited |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+
| a794d5d1b9014757b8e61e7c97f0ff27 | 20f8cd5b95dc44cf98b43348b7f6124c |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | 7c4087ac47184343997d023de6d357d4 |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | fcc523d60cc74e19ad13d4f7b029df70 |       | a9001f880f424f79929111ef200030fc |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | fcc523d60cc74e19ad13d4f7b029df70 |       |                                  |        | all    | False     |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+

查看服务列表，是否创建了该服务
[root@controller ~]# openstack service list

+----------------------------------+-----------+-----------+
| ID                               | Name      | Type      |
+----------------------------------+-----------+-----------+
| 4a1a975ce099473a9f8a0b56ba24bbcc | keystone  | identity  |
| b645abce902b46f9b1b4f39a80601cb1 | glance    | image     |
| f517d649ec4b4f58996d800679698090 | placement | placement |
+----------------------------------+-----------+-----------+

查看此服务的3个服务端点是否建立
[root@controller ~]# openstack endpoint list

+----------------------------------+-----------+--------------+--------------+---------+-----------+---------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                       |
+----------------------------------+-----------+--------------+--------------+---------+-----------+---------------------------+
| 2330a6b94c92439b89cd64e40b4fce61 | RegionOne | keystone     | identity     | True    | admin     | http://contrpller:5000/v3 |
| 41d7666d90de4d0fafaadf0422dc3432 | RegionOne | placement    | placement    | True    | public    | http://controller:8778    |
| 484f573d093545048cfe6d7de14c7609 | RegionOne | glance       | image        | True    | public    | http://controller:9292    |
| 5eaacaa4019e49db865a97e9a728e133 | RegionOne | placement    | placement    | True    | admin     | http://controller:8778    |
| 6770c81a2b4e4de98ce89ff16ff69bd1 | RegionOne | glance       | image        | True    | internal  | http://controller:9292    |
| 7a5b6a7987884264bde8dcbbc746db9b | RegionOne | placement    | placement    | True    | internal  | http://controller:8778    |
| 91b53fd5293b44889afc309a7c676e41 | RegionOne | keystone     | identity     | True    | internal  | http://controller:5000/v3 |
| 925e27d07cb248489a0938626bad1d0a | RegionOne | keystone     | identity     | True    | public    | http://controller:5000/v3 |
| fc5b67bed2304918bdf51630d6fc09f5 | RegionOne | glance       | image        | True    | admin     | http://controller:9292    |
+----------------------------------+-----------+--------------+--------------+---------+-----------+---------------------------+

```

### nova服务

### **一、安装配置controller节点上的nova服务**

安装Nova服务

```jsx
yum install -y openstack-nova-api openstack-nova-conductor openstack-nova-scheduler openstack-nova-novncproxy
```

查看用户的信息

```jsx
[root@controller ~]# cat /etc/passwd |grep nova

nova:x:162:162:OpenStack Nova Daemons:/var/lib/nova:/sbin/nologin
```

查看用户组的信息

```jsx
[root@controller ~]# cat /etc/group |grep nova

nobody:x:99:nova
nova:x:162:nova
```

进入数据库

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 23
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

新建“nova_api”“nova_cell0”“nova”数据库
MariaDB [(none)]> create database nova_api;
Query OK, 1 row affected (0.001 sec)

MariaDB [(none)]> create database nova_cell0;
Query OK, 1 row affected (0.001 sec)

MariaDB [(none)]> create database nova;
Query OK, 1 row affected (0.000 sec)

为数据库进行授权（授权“nova”用户从本地和远程管理“nova_api”“nova_cell0”“nova”数据库的权限）
MariaDB [(none)]>  grant all privileges on nova_api.* to 'nova'@'localhost' identified by '123456';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]>  grant all privileges on nova_api.* to 'nova'@'%' identified by '123456';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]>  grant all privileges on nova_cell0.* to 'nova'@'localhost' identified by '123456';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]>  grant all privileges on nova_cell0.* to 'nova'@'%' identified by '123456';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]>  grant all privileges on nova.* to 'nova'@'localhost' identified by '123456';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]>  grant all privileges on nova.* to 'nova'@'%' identified by '123456';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> quit
Bye
```

备份配置文件

```jsx
cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
```

去掉配置文件中的注释行和空行，并生成新的配置文件

```jsx
grep -Ev '^$|#' /etc/nova/nova.conf.bak > /etc/nova/nova.conf
```

> E表示采用正则表达式进行读取；^$|#（^$ ^表示一行的开头，$表示一行的结尾，整体表示为匹配空行；| 这里表示或者； #表示注释行）v表示反向匹配

修改配置文件

```jsx
vi /etc/nova/nova.conf

[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://rabbitmq:123456@controller:5672
my_ip =192.168.20.100
use_neutron = true
enabled_apis：指定启用的 API 服务，这里是计算 API (osapi_compute) 和元数据 API (metadata)。
transport_url：RabbitMQ 的连接 URL，用于消息队列。
my_ip：当前主机的 IP 地址。
use_neutron：是否使用 Neutron 作为网络服务。
[api]
auth_strategy = keystone
auth_strategy：认证策略，这里使用 Keystone 认证。
[api_database]
connection = mysql+pymysql://nova:123456@controller/nova_api
connection：API 数据库的连接字符串。
[barbican]
[cache]
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[database]
connection = mysql+pymysql://nova:123456@controller/nova
connection：主数据库的连接字符串。
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://controller:9292
api_servers：Glance 图像服务的 API 服务器地址。
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = project
username = nova
password = 123456
auth_url：Keystone 认证服务的 URL。
memcached_servers：Memcached 服务器地址，用于缓存。
auth_type：认证类型，这里使用密码认证。
project_domain_name：项目域名，默认 Default。
user_domain_name：用户域名，默认 Default。
project_name：项目名称，这里是 project。
username：Nova 服务的用户名。
password：Nova 服务的密码。
[libvirt]
[metrics]
[mks]
[neutron]
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
lock_path：锁文件路径，用于并发控制。
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = project
username = placement
password = 123456
region_name = RegionOne
auth_url：Keystone 认证服务的 URL。
auth_type：认证类型，这里使用密码认证。
project_domain_name：项目域名，默认 Default。
user_domain_name：用户域名，默认 Default。
project_name：项目名称，这里是 project。
username：Placement 服务的用户名。
password：Placement 服务的密码。
region_name：区域名称，这里是 RegionOne。
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
enabled：是否启用 VNC 。
server_listen：VNC 服务器监听的地址，这里使用 $my_ip 表示当前主机的 IP 地址。
server_proxyclient_address：VNC 代理客户端地址，这里也使用 $my_ip 表示当前主机的 IP 地址。
[workarounds]
[wsgi]
[xenserver]
[xvp]
[zvm]
```

初始化“nova\_api”数据库

```jsx
su nova -s /bin/sh -c "nova-manage api_db sync"
```

创建“cell1”单元，该单元使用“nova”数据库

```jsx
su nova -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1"
```

映射“nova”到“cell0”数据库，使“cell0”的表结构和“nova”的表结构一致

```jsx
su nova -s /bin/sh -c "nova-manage cell_v2 map_cell0"
```

初始化“nova”数据库，由于映射存在，“cell0”中同时创建相同的数据表

```jsx
su nova -s /bin/sh -c "nova-manage db sync"
```

进行验证：获得已注册的单元列表

```jsx
nova-manage cell_v2 list_cells

+-------+--------------------------------------+----------------------------------------+-------------------------------------------------+----------+
|  Name |                 UUID                 |             Transport URL              |               Database Connection               | Disabled |
+-------+--------------------------------------+----------------------------------------+-------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                 none:/                 | mysql+pymysql://nova:****@controller/nova_cell0 |  False   |
| cell1 | 23a019a1-3e8f-4073-8b24-660ebf13fad6 | rabbit://rabbitmq:****@controller:5672 |    mysql+pymysql://nova:****@controller/nova    |  False   |
+-------+--------------------------------------+----------------------------------------+-------------------------------------------------+----------+
```

导入环境变量模拟登陆（可以用source或者.）

```jsx
source admin-login
```

在“default”域中创建用户nova

```jsx
openstack user create --domain default --password 123456 nova

+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 7e956474824a416ca7d36b84fa0f66bd |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

为用户“nova”分配“admin”角色

```jsx
openstack role add --project project --user nova admin
```

创建一个名为“nova”，类型为“compute”的服务

```jsx
openstack service create --name nova compute

+---------+----------------------------------+
| Field   | Value                            |
+---------+----------------------------------+
| enabled | True                             |
| id      | cbe846595ca74ef98ced87c9e0f33324 |
| name    | nova                             |
| type    | compute                          |
+---------+----------------------------------+
```

创建公众用户访问的端点

```jsx
openstack endpoint create --region RegionOne nova public http://controller:8774/v2.1

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 00905e83131944549c31c2706a6fcbc0 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | cbe846595ca74ef98ced87c9e0f33324 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

创建内部组件访问的端点

```jsx
openstack endpoint create --region RegionOne nova internal http://controller:8774/v2.1

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 26aeaad364c8463b8635636c6e91a676 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | cbe846595ca74ef98ced87c9e0f33324 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

创建“admin”用户访问的端点

```jsx
openstack endpoint create --region RegionOne nova admin http://controller:8774/v2.1

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 85ba3807db7448c3b36cc38272afad81 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | cbe846595ca74ef98ced87c9e0f33324 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

启动“openstack-nova-api openstack-nova-conductor openstack-nova-scheduler openstack-nova-novncproxy”四大模块，并设置为开机自启动

```jsx
systemctl start openstack-nova-api openstack-nova-conductor openstack-nova-scheduler openstack-nova-novncproxy && systemctl enable openstack-nova-api openstack-nova-conductor openstack-nova-scheduler openstack-nova-novncproxy
```

查看“nova”服务占用的端口情况

```jsx
netstat -nutpl|grep 877

tcp        0      0 0.0.0.0:8774            0.0.0.0:*               LISTEN      4843/python2        
tcp        0      0 0.0.0.0:8775            0.0.0.0:*               LISTEN      4843/python2        
tcp6       0      0 :::8778                 :::*                    LISTEN      2267/httpd   
```

查看计算服务中各模块的服务状态

```jsx
 openstack compute service list
 
+----+----------------+------------+----------+---------+-------+----------------------------+
| ID | Binary         | Host       | Zone     | Status  | State | Updated At                 |
+----+----------------+------------+----------+---------+-------+----------------------------+
|  1 | nova-conductor | controller | internal | enabled | up    | 2024-05-26T03:36:08.000000 |
|  5 | nova-scheduler | controller | internal | enabled | up    | 2024-05-26T03:36:13.000000 |
+----+----------------+------------+----------+---------+-------+----------------------------+
```

验证单元是否已正确配置

```jsx
	nova-manage cell_v2 list_cells
	
+-------+--------------------------------------+----------------------------------------+-------------------------------------------------+----------+
|  Name |                 UUID                 |             Transport URL              |               Database Connection               | Disabled |
+-------+--------------------------------------+----------------------------------------+-------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                 none:/                 | mysql+pymysql://nova:****@controller/nova_cell0 |  False   |
| cell1 | 23a019a1-3e8f-4073-8b24-660ebf13fad6 | rabbit://rabbitmq:****@controller:5672 |    mysql+pymysql://nova:****@controller/nova    |  False   |
+-------+--------------------------------------+----------------------------------------+-------------------------------------------------+----------+
```

在“default”域中创建用户nova密码为‘123456’

```jsx
openstack user create --domain default --password 123456 nova
```

为“nova”用户分配“admin”角色

```jsx
openstack role add --project project --user nova admin
```

创建一个名为“nova”类型为“compute”的服务

```jsx
openstack service create --name nova compute

+---------+----------------------------------+
| Field   | Value                            |
+---------+----------------------------------+
| enabled | True                             |
| id      | 29646558831f477fa7e0ded095a77ca6 |
| name    | nova                             |
| type    | compute                          |
+---------+----------------------------------+
```

创建公众用户访问的服务端点

```jsx
openstack endpoint create --region RegionOne nova public http://controller:8774/v2.1

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 4b6e9b411f5c41cd9f21dffecd996d0e |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 29646558831f477fa7e0ded095a77ca6 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

创建内部组件访问的服务端点

```jsx
openstack endpoint create --region RegionOne nova internal http://controller:8774/v2.1

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a5369014eb7f4ae0a15dfd6abcdc3dcd |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 29646558831f477fa7e0ded095a77ca6 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

创建admin用户访问的服务端点

```jsx
openstack endpoint create --region RegionOne nova admin http://controller:8774/v2.1

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a6c0bf48c73e4a588fb9507754e0a7e1 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 29646558831f477fa7e0ded095a77ca6 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

### **二、安装配置计算节点上的nova服务**

安装Nova服务

```jsx
yum install -y openstack-nova-compute
```

查看用户的信息

```jsx
[root@compute ~]# cat /etc/passwd | grep nova

nova:x:162:162:OpenStack Nova Daemons:/var/lib/nova:/sbin/nologin
```

查看用户组的信息

```jsx
[root@compute ~]# cat /etc/group | grep nova

nobody:x:99:nova
qemu:x:107:nova
libvirt:x:990:nova
nova:x:162:nova
```

备份配置文件

```jsx
cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
```

去掉配置文件中的注释行和空行，并生成新的配置文件

```jsx
grep -Ev '^$|#' /etc/nova/nova.conf.bak > /etc/nova/nova.conf
```

> E表示采用正则表达式进行读取；^$|#（^$ ^表示一行的开头，$表示一行的结尾，整体表示为匹配空行；| 这里表示或者； #表示注释行）v表示反向匹配

修改配置文件

```jsx
vi /etc/nova/nova.conf

[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://rabbitmq:123456@controller:5672
my_ip =192.168.20.200
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api]
auth_strategy = keystone
[api_database]
[barbican]
[cache]
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[database]
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://controller:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = project
username = nova
password = 123456
[libvirt]
virt_type = qemu
[metrics]
[mks]
[neutron]
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = project
username = placement
password = 123456
region_name = RegionOne
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
novncproxy_base_url = http://192.168.20.100:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
[zvm]
```

```jsx
[DEFAULT] 部分
enabled_apis: 指定启用的API，这里启用了osapi_compute和metadata API。
transport_url: RabbitMQ的消息传输URL，包含用户名、密码、主机名和端口。
my_ip: Nova服务主机的IP地址。
use_neutron: 指示是否使用Neutron进行网络管理，这里设置为true。
firewall_driver: 指定使用的防火墙驱动，这里设置为nova.virt.firewall.NoopFirewallDriver，表示不进行防火墙管理。
[api] 部分
auth_strategy: 指定认证策略，这里设置为keystone，即使用OpenStack的身份认证服务。
[glance] 部分
api_servers: Glance镜像服务的URL。
[keystone_authtoken] 部分
auth_url: Keystone身份认证服务的URL。
memcached_servers: 用于令牌缓存的Memcached服务器地址。
auth_type: 认证类型，这里设置为password。
project_domain_name: 项目域名。
user_domain_name: 用户域名。
project_name: 用于服务认证的项目名称。
username: 用于认证的用户名。
password: 用于认证的密码。
[libvirt] 部分
virt_type: 指定使用的虚拟化类型，这里设置为qemu。
[oslo_concurrency] 部分
lock_path: 存储锁文件的目录。
[placement] 部分
auth_url: Keystone身份认证服务的URL。
auth_type: 认证类型，这里设置为password。
project_domain_name: 项目域名。
user_domain_name: 用户域名。
project_name: 用于服务认证的项目名称。
username: 用于认证的用户名。
password: 用于认证的密码。
region_name: 放置服务所在的区域名称。
[vnc] 部分
enabled: 启用VNC服务。
server_listen: VNC服务器监听的IP地址，0.0.0.0表示监听所有接口。
server_proxyclient_address: 代理客户端使用的地址。
novncproxy_base_url: noVNC代理服务的基本URL。
```

启动nova服务，并设置为开机自启动

```jsx
systemctl start libvirtd openstack-nova-compute && systemctl enable libvirtd openstack-nova-compute
```

### **三、发现计算节点并启动服务**

导入环境变量模拟登陆

```jsx
source admin-login
```

发现新的计算节点（计算节点将自动与cell1单元形成关联，即可通过cell1对该计算节点进行管理）

```jsx
su nova -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose"
```

修改配置文件（设置每隔60s自动执行一次发现命令）

```jsx
vi /etc/nova/nova.conf

[scheduler]
discover_hosts_in_cells_interval = 60
```

重启nova服务并设置为开机自启动

```jsx
systemctl restart openstack-nova-api && systemctl enable openstack-nova-api
```

查看计算服务列表

```jsx
openstack compute service list

+----+----------------+------------+----------+---------+-------+----------------------------+
| ID | Binary         | Host       | Zone     | Status  | State | Updated At                 |
+----+----------------+------------+----------+---------+-------+----------------------------+
|  1 | nova-conductor | controller | internal | enabled | up    | 2024-05-26T05:49:25.000000 |
|  5 | nova-scheduler | controller | internal | enabled | up    | 2024-05-26T05:49:21.000000 |
|  7 | nova-compute   | compute    | nova     | enabled | up    | 2024-05-26T05:49:22.000000 |
+----+----------------+------------+----------+---------+-------+----------------------------+
```

查看所有OpenStack服务及端点列表

```jsx
openstack catalog list

+-----------+-----------+-----------------------------------------+
| Name      | Type      | Endpoints                               |
+-----------+-----------+-----------------------------------------+
| nova      | compute   | RegionOne                               |
|           |           |   public: http://controller:8774/v2.1   |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:8774/v2.1 |
|           |           | RegionOne                               |
|           |           |   admin: http://controller:8774/v2.1    |
|           |           |                                         |
| keystone  | identity  | RegionOne                               |
|           |           |   admin: http://controller:5000/v3      |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:5000/v3   |
|           |           | RegionOne                               |
|           |           |   public: http://controller:5000/v3     |
|           |           |                                         |
| glance    | image     | RegionOne                               |
|           |           |   public: http://controller:9292        |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:9292      |
|           |           | RegionOne                               |
|           |           |   admin: http://controller:9292         |
|           |           |                                         |
| placement | placement | RegionOne                               |
|           |           |   public: http://controller:8778        |
|           |           | RegionOne                               |
|           |           |   admin: http://controller:8778         |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:8778      |
|           |           |                                         |
+-----------+-----------+-----------------------------------------+
```

使用Nova状态检测工具进行检查

```jsx
nova-status upgrade check

+--------------------------------+
| Upgrade Check Results          |
+--------------------------------+
| Check: Cells v2                |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Placement API           |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Ironic Flavor Migration |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Cinder API              |
| Result: Success                |
| Details: None                  |
+--------------------------------+
```

自检

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 52
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

查看本地‘keystone’用户是否授予对‘keystone’数据库的完全控制权限
MariaDB [(none)]> show grants  for 'nova'@'localhost';
+-------------------------------------------------------------------------------------------------------------+
| Grants for nova@localhost                                                                                   |
+-------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'nova'@'localhost' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9' |
| GRANT ALL PRIVILEGES ON `nova`.* TO 'nova'@'localhost'                                                      |
| GRANT ALL PRIVILEGES ON `nova_cell0`.* TO 'nova'@'localhost'                                                |
| GRANT ALL PRIVILEGES ON `nova_api`.* TO 'nova'@'localhost'                                                  |
+-------------------------------------------------------------------------------------------------------------+
4 rows in set (0.006 sec)

查看远程‘keystone’用户是否授予对‘keystone’数据库的完全控制权限
MariaDB [(none)]> show grants  for 'nova'@'%';
+-----------------------------------------------------------------------------------------------------+
| Grants for nova@%                                                                                   |
+-----------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'nova'@'%' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9' |
| GRANT ALL PRIVILEGES ON `nova`.* TO 'nova'@'%'                                                      |
| GRANT ALL PRIVILEGES ON `nova_api`.* TO 'nova'@'%'                                                  |
| GRANT ALL PRIVILEGES ON `nova_cell0`.* TO 'nova'@'%'                                                |
+-----------------------------------------------------------------------------------------------------+
4 rows in set (0.000 sec)

MariaDB [(none)]> quit
Bye

查看用户列表是否存在nova 
[root@controller ~]# openstack user list

+----------------------------------+-----------+
| ID                               | Name      |
+----------------------------------+-----------+
| fcc523d60cc74e19ad13d4f7b029df70 | admin     |
| 7c4087ac47184343997d023de6d357d4 | glance    |
| 20f8cd5b95dc44cf98b43348b7f6124c | placement |
| 7e956474824a416ca7d36b84fa0f66bd | nova      |
+----------------------------------+-----------+

查看角色ID
[root@controller ~]# openstack role list

+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| 4fb78ac4adff43ef8127afdf81f84694 | user   |
| 57adb6fe8ac04387853003b18c9c7240 | member |
| a794d5d1b9014757b8e61e7c97f0ff27 | admin  |
| dd525862a0ba4b3090b00e18436abdfd | reader |
+----------------------------------+--------+

查看nova用户是否和admin角色绑定
[root@controller ~]# openstack role assignment list

+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+
| Role                             | User                             | Group | Project                          | Domain | System | Inherited |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+
| a794d5d1b9014757b8e61e7c97f0ff27 | 20f8cd5b95dc44cf98b43348b7f6124c |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | 7c4087ac47184343997d023de6d357d4 |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | 7e956474824a416ca7d36b84fa0f66bd |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | fcc523d60cc74e19ad13d4f7b029df70 |       | a9001f880f424f79929111ef200030fc |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | fcc523d60cc74e19ad13d4f7b029df70 |       |                                  |        | all    | False     |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+

查看服务列表，是否创建了该服务
[root@controller ~]# openstack service list

+----------------------------------+-----------+-----------+
| ID                               | Name      | Type      |
+----------------------------------+-----------+-----------+
| 29646558831f477fa7e0ded095a77ca6 | nova      | compute   |
| 4a1a975ce099473a9f8a0b56ba24bbcc | keystone  | identity  |
| b645abce902b46f9b1b4f39a80601cb1 | glance    | image     |
| f517d649ec4b4f58996d800679698090 | placement | placement |
+----------------------------------+-----------+-----------+

查看此服务的3个服务端点是否建立
[root@controller ~]# openstack endpoint list

+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                         |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------+
| 2330a6b94c92439b89cd64e40b4fce61 | RegionOne | keystone     | identity     | True    | admin     | http://controller:5000/v3   |
| 41d7666d90de4d0fafaadf0422dc3432 | RegionOne | placement    | placement    | True    | public    | http://controller:8778      |
| 484f573d093545048cfe6d7de14c7609 | RegionOne | glance       | image        | True    | public    | http://controller:9292      |
| 4b6e9b411f5c41cd9f21dffecd996d0e | RegionOne | nova         | compute      | True    | public    | http://controller:8774/v2.1 |
| 5eaacaa4019e49db865a97e9a728e133 | RegionOne | placement    | placement    | True    | admin     | http://controller:8778      |
| 6770c81a2b4e4de98ce89ff16ff69bd1 | RegionOne | glance       | image        | True    | internal  | http://controller:9292      |
| 7a5b6a7987884264bde8dcbbc746db9b | RegionOne | placement    | placement    | True    | internal  | http://controller:8778      |
| 91b53fd5293b44889afc309a7c676e41 | RegionOne | keystone     | identity     | True    | internal  | http://controller:5000/v3   |
| 925e27d07cb248489a0938626bad1d0a | RegionOne | keystone     | identity     | True    | public    | http://controller:5000/v3   |
| a5369014eb7f4ae0a15dfd6abcdc3dcd | RegionOne | nova         | compute      | True    | internal  | http://controller:8774/v2.1 |
| a6c0bf48c73e4a588fb9507754e0a7e1 | RegionOne | nova         | compute      | True    | admin     | http://controller:8774/v2.1 |
| fc5b67bed2304918bdf51630d6fc09f5 | RegionOne | glance       | image        | True    | admin     | http://controller:9292      |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------+
```

### Neutron服务

准备工作

### **1、将外网网卡设置为混杂模式**

```jsx
[root@controller ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ef:23:ac brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.100/24 brd 192.168.100.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ced6:fa2:c4e:ce43/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::e6b7:e547:4777:89d9/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ef:23:b6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.20.100/24 brd 192.168.20.255 scope global noprefixroute ens34
       valid_lft forever preferred_lft forever
    inet6 fe80::8382:5a58:74c3:bd21/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
    inet6 fe80::708e:880f:93e:9c63/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

将外网网卡“ens33”设置为混杂模式

```jsx
ifconfig ens33 promisc
```

查看是否设置成功

```jsx
[root@controller ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ef:23:ac brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.100/24 brd 192.168.100.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ced6:fa2:c4e:ce43/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::e6b7:e547:4777:89d9/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ef:23:b6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.20.100/24 brd 192.168.20.255 scope global noprefixroute ens34
       valid_lft forever preferred_lft forever
    inet6 fe80::8382:5a58:74c3:bd21/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
    inet6 fe80::708e:880f:93e:9c63/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

设置开机后混杂模式自动生效（编辑配置文件）

```jsx
vi /etc/profile
添加下面命令
ifconfig ens33 promisc
```

### **2、加载桥接模式防火墙模块**

编辑“/etc/sysctl.conf”系统参数配置文件

```jsx
vi /etc/sysctl.conf

net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

加载“br\_netfilter”模块

```jsx
modprobe br_netfilter
```

检查模块加载情况

```jsx
sysctl -p

net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

> 网络过滤器是linux内核的一个软件框架，用于管理网络数据包，不仅具有网络地址转换功能，还有数据包内容修改以及包过滤等防火墙功能

> 与Linux网桥功能联动实现桥接模式防火墙功能。centos默认没有加载桥接模式防火墙模块，需要手动添加

### **3、计算节点同样操作**

### **1、安装controller节点Neutron服务：**

安装Neutron服务

```jsx
yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge
```

查看用户的信息

```jsx
[root@controller ~]# cat /etc/passwd | grep neutron
neutron:x:990:987:OpenStack Neutron Daemons:/var/lib/neutron:/sbin/nologin
```

查看用户组的信息

```jsx
[root@controller ~]# cat /etc/group | grep neutron
neutron:x:987:
```

**创建Neutron的数据库并授权**

进入数据库

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 53
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

新建“neutron”数据库
MariaDB [(none)]> CREATE DATABASE neutron;
Query OK, 1 row affected (0.002 sec)

为数据库进行授权（授权“neutron”用户从本地和远程管理“neutron”数据库的权限）
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.012 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> quit
Bye
```

### **修改Neutron服务相关配置文件**

**1.配置neutron组件信息**

备份配置文件

```jsx
cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
```

去掉配置文件中的注释行和空行，并生成新的配置文件

```jsx
grep -Ev '^$|#' /etc/neutron/neutron.conf.bak > /etc/neutron/neutron.conf
```

> E表示采用正则表达式进行读取；^$|#（^$ ^表示一行的开头，$表示一行的结尾，整体表示为匹配空行；| 这里表示或者； #表示注释行）v表示反向匹配

修改配置文件

```jsx
vi /etc/neutron/neutron.conf

[DEFAULT]
core_plugin = ml2
service_plugins =
transport_url = rabbit://rabbitmq:123456@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
[cors]
[database]
connection = mysql+pymysql://neutron:123456@controller/neutron
[keystone_authtoken]
auth_url = http://controller:5000
memcached_servers = controller:11211
project_domain_name = Default
user_domain_name = Default
project_name = project
username = neutron
password = 123456
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[privsep]
[ssl]
[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = project
username = nova
password = 123456
region_name = RegionOne
server_proxyclient_address = 192.168.20.100
```

```jsx
[DEFAULT] 部分
core_plugin: 指定使用的核心插件，这里设置为ml2（Modular Layer 2）。
service_plugins: 指定使用的服务插件，这里为空，表示不使用额外的服务插件。
transport_url: RabbitMQ的消息传输URL，包含用户名、密码和主机名。
auth_strategy: 指定认证策略，这里设置为keystone，即使用OpenStack的身份认证服务。
notify_nova_on_port_status_changes: 在端口状态变化时通知Nova，这里设置为true。
notify_nova_on_port_data_changes: 在端口数据变化时通知Nova，这里设置为true。
[database] 部分
connection: 连接Neutron数据库的URL，这里使用MySQL，通过pymysql驱动连接。
[keystone_authtoken] 部分
auth_url: Keystone身份认证服务的URL。
memcached_servers: 用于令牌缓存的Memcached服务器地址。
project_domain_name: 项目域名。
user_domain_name: 用户域名。
project_name: 用于服务认证的项目名称。
username: 用于认证的用户名。
password: 用于认证的密码。
[oslo_concurrency] 部分
lock_path: 存储锁文件的目录。
[nova] 部分
auth_url: Keystone身份认证服务的URL。
auth_type: 认证类型，这里设置为password。
project_domain_name: 项目域名。
user_domain_name: 用户域名。
project_name: 用于服务认证的项目名称。
username: 用于认证的用户名。
password: 用于认证的密码。
region_name: 放置服务所在的区域名称。
server_proxyclient_address: 代理客户端使用的地址，这里设置为192.168.20.100。
```

**2.配置二层模块插件（ML2 Plugin）组件信息**

备份配置文件

```jsx
cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bak
```

去掉配置文件中的注释行和空行，并生成新的配置文件

```jsx
grep -Ev '^$|#' /etc/neutron/plugins/ml2/ml2_conf.ini.bak > /etc/neutron/plugins/ml2/ml2_conf.ini
```

> E表示采用正则表达式进行读取；^$|#（^$ ^表示一行的开头，$表示一行的结尾，整体表示为匹配空行；| 这里表示或者； #表示注释行）v表示反向匹配

修改配置文件

```jsx
vi /etc/neutron/plugins/ml2/ml2_conf.ini

[DEFAULT]
[ml2]
type_drivers = flat
tenant_network_types =
mechanism_drivers = linuxbridge
extension_drivers = port_security
[ml2_type_flat]
flat_networks = provider
[securitygroup]
enable_ipset = true
```

```jsx
[DEFAULT] 部分
通常用来定义全局配置参数。

[ml2] 部分
type_drivers: 指定网络类型驱动，这里设置为flat，即使用平面网络。
tenant_network_types: 指定租户网络类型，这里为空，表示不创建租户网络类型。
mechanism_drivers: 指定机制驱动，这里设置为linuxbridge，即使用Linux桥接。
extension_drivers: 指定扩展驱动，这里设置为port_security，启用端口安全扩展。
[ml2_type_flat] 部分
flat_networks: 指定平面网络的名称，这里设置为provider，即提供者网络。
[securitygroup] 部分
enable_ipset: 启用ipset以提高安全组规则应用的性能，这里设置为true。
```

启动ML2插件

```jsx
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

> 只有在"/etc/neutron/"下的插件才可以生效，所以将“ml2\_conf.ini”映射为“/etc/neutron/plugins.ini”文件

**3.修改网桥代理（linuxbridge-agent）的配置文件：**

备份配置文件

```jsx
cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
```

去掉配置文件中的注释行和空行，并生成新的配置文件

```jsx
grep -Ev '^$|#' /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak > /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```

> E表示采用正则表达式进行读取；^$|#（^$ ^表示一行的开头，$表示一行的结尾，整体表示为匹配空行；| 这里表示或者； #表示注释行）v表示反向匹配

修改配置文件

```jsx
vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[DEFAULT]
[linux_bridge]
physical_interface_mappings = provider:ens33
[vxlan]
enable_vxlan = false
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

```jsx
[DEFAULT] 部分
通常用于定义全局配置参数。

[linux_bridge] 部分
physical_interface_mappings: 定义物理网络接口的映射。这里设置为provider:ens33，表示将名为provider的网络映射到物理接口ens33。
[vxlan] 部分
enable_vxlan: 指定是否启用VXLAN（虚拟扩展局域网）。这里设置为false，表示不启用VXLAN。
[securitygroup] 部分
enable_security_group: 指定是否启用安全组。这里设置为true，表示启用安全组。
firewall_driver: 指定防火墙驱动程序。这里设置为neutron.agent.linux.iptables_firewall.IptablesFirewallDriver，即使用iptables作为防火墙驱动。
```

**4.修改DHCP代理（dhcp-agent）的配置文件：**

备份配置文件

```jsx
cp /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.bak
```

去掉配置文件中的注释行和空行，并生成新的配置文件

```jsx
grep -Ev '^$|#' /etc/neutron/dhcp_agent.ini.bak > /etc/neutron/dhcp_agent.ini
```

> E表示采用正则表达式进行读取；^$|#（^$ ^表示一行的开头，$表示一行的结尾，整体表示为匹配空行；| 这里表示或者； #表示注释行）v表示反向匹配

修改配置文件

```jsx
vi /etc/neutron/dhcp_agent.ini

[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

```jsx
[DEFAULT] 部分
interface_driver: 指定接口驱动，这里设置为linuxbridge，表示使用Linux桥接。Linux桥接驱动负责在虚拟网络设备和物理网络设备之间建立桥接，以实现网络通信。
dhcp_driver: 指定DHCP驱动，这里设置为neutron.agent.linux.dhcp.Dnsmasq，表示使用Dnsmasq作为DHCP服务。Dnsmasq是一个轻量级的、易于配置的DHCP和DNS服务器，适合用于小型网络环境。
enable_isolated_metadata: 启用独立的元数据服务，这里设置为true，表示为每个租户网络提供独立的元数据代理。元数据代理提供实例在启动时所需的配置信息，例如SSH密钥和用户数据。
```

**5.修改元数据代理（neutron-metadata-agent）的配置文件**

```jsx
vi /etc/neutron/metadata_agent.ini

[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = METADATA_SECRET
```

```jsx
[DEFAULT] 部分
nova_metadata_host: 指定Nova元数据服务的主机名或IP地址。这里设置为controller，表示元数据服务运行在名为controller的主机上。元数据代理会将请求转发到这个主机以获取实例的元数据。在这里，controller是一个主机名，通常是运行控制节点的主机。
metadata_proxy_shared_secret: 指定元数据代理和Nova元数据服务之间共享的密钥。这里设置为METADATA_SECRET，这是一个共享的秘密，用于元数据代理和Nova元数据服务之间的认证。这个密钥用于认证，以确保只有授权的元数据代理可以与Nova元数据服务通信。
```

**6.修改Nova的配置文件**

```jsx
vi /etc/nova/nova.conf
 
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = project
username = neutron
password = 123456
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET
```

```jsx
[neutron] 部分
auth_url: 这个配置项指定了Keystone服务的URL，Neutron将通过这个URL进行身份认证。
auth_url: 这个配置项指定了Keystone服务的URL，Neutron将通过这个URL进行身份认证。
auth_type: 认证类型为password，表示使用用户名和密码进行认证。
project_domain_name: 指定项目所在的域名。
user_domain_name: 指定用户所在的域名。
region_name: 指定区域名称，以支持多区域部署。
project_name: 指定用于认证的项目名称。
username: 指定用于认证的用户名，这里是Neutron服务用户。
password: 指定用于认证的密码。
service_metadata_proxy: 启用元数据代理服务，确保实例可以访问元数据。
metadata_proxy_shared_secret: 这个共享密钥用于元数据代理和Nova元数据服务之间的认证，确保只有授权的元数据代理可以与Nova元数据服务通信。
```

**初始化数据库**

```jsx
su neutron -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head"
```

进入数据库

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 66
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

使用neutron数据库
MariaDB [(none)]> use neutron;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed

查看neutron数据库的表
MariaDB [neutron]> show tables;
+-----------------------------------------+
| Tables_in_neutron                       |
+-----------------------------------------+
| address_scopes                          |
| agents                                  |
| alembic_version                         |
| allowedaddresspairs                     |
| arista_provisioned_nets                 |
| arista_provisioned_tenants              |
| arista_provisioned_vms                  |
| auto_allocated_topologies               |
| bgp_peers                               |
| bgp_speaker_dragent_bindings            |
| bgp_speaker_network_bindings            |
| bgp_speaker_peer_bindings               |
| bgp_speakers                            |
| brocadenetworks                         |
| brocadeports                            |
| cisco_csr_identifier_map                |
| cisco_hosting_devices                   |
| cisco_ml2_apic_contracts                |
| cisco_ml2_apic_host_links               |
| cisco_ml2_apic_names                    |
| cisco_ml2_n1kv_network_bindings         |
| cisco_ml2_n1kv_network_profiles         |
| cisco_ml2_n1kv_policy_profiles          |
| cisco_ml2_n1kv_port_bindings            |
| cisco_ml2_n1kv_profile_bindings         |
| cisco_ml2_n1kv_vlan_allocations         |
| cisco_ml2_n1kv_vxlan_allocations        |
| cisco_ml2_nexus_nve                     |
| cisco_ml2_nexusport_bindings            |
| cisco_port_mappings                     |
| cisco_router_mappings                   |
| conntrack_helpers                       |
| consistencyhashes                       |
| default_security_group                  |
| dnsnameservers                          |
| dvr_host_macs                           |
| externalnetworks                        |
| extradhcpopts                           |
| firewall_policies                       |
| firewall_rules                          |
| firewalls                               |
| flavors                                 |
| flavorserviceprofilebindings            |
| floatingipdnses                         |
| floatingips                             |
| ha_router_agent_port_bindings           |
| ha_router_networks                      |
| ha_router_vrid_allocations              |
| healthmonitors                          |
| ikepolicies                             |
| ipallocationpools                       |
| ipallocations                           |
| ipamallocationpools                     |
| ipamallocations                         |
| ipamsubnets                             |
| ipsec_site_connections                  |
| ipsecpeercidrs                          |
| ipsecpolicies                           |
| logs                                    |
| lsn                                     |
| lsn_port                                |
| maclearningstates                       |
| members                                 |
| meteringlabelrules                      |
| meteringlabels                          |
| ml2_brocadenetworks                     |
| ml2_brocadeports                        |
| ml2_distributed_port_bindings           |
| ml2_flat_allocations                    |
| ml2_geneve_allocations                  |
| ml2_geneve_endpoints                    |
| ml2_gre_allocations                     |
| ml2_gre_endpoints                       |
| ml2_nexus_vxlan_allocations             |
| ml2_nexus_vxlan_mcast_groups            |
| ml2_port_binding_levels                 |
| ml2_port_bindings                       |
| ml2_ucsm_port_profiles                  |
| ml2_vlan_allocations                    |
| ml2_vxlan_allocations                   |
| ml2_vxlan_endpoints                     |
| multi_provider_networks                 |
| network_segment_ranges                  |
| networkconnections                      |
| networkdhcpagentbindings                |
| networkdnsdomains                       |
| networkgatewaydevicereferences          |
| networkgatewaydevices                   |
| networkgateways                         |
| networkqueuemappings                    |
| networkrbacs                            |
| networks                                |
| networksecuritybindings                 |
| networksegments                         |
| neutron_nsx_network_mappings            |
| neutron_nsx_port_mappings               |
| neutron_nsx_router_mappings             |
| neutron_nsx_security_group_mappings     |
| nexthops                                |
| nsxv_edge_dhcp_static_bindings          |
| nsxv_edge_vnic_bindings                 |
| nsxv_firewall_rule_bindings             |
| nsxv_internal_edges                     |
| nsxv_internal_networks                  |
| nsxv_port_index_mappings                |
| nsxv_port_vnic_mappings                 |
| nsxv_router_bindings                    |
| nsxv_router_ext_attributes              |
| nsxv_rule_mappings                      |
| nsxv_security_group_section_mappings    |
| nsxv_spoofguard_policy_network_mappings |
| nsxv_tz_network_bindings                |
| nsxv_vdr_dhcp_bindings                  |
| nuage_net_partition_router_mapping      |
| nuage_net_partitions                    |
| nuage_provider_net_bindings             |
| nuage_subnet_l2dom_mapping              |
| poolloadbalanceragentbindings           |
| poolmonitorassociations                 |
| pools                                   |
| poolstatisticss                         |
| portbindingports                        |
| portdataplanestatuses                   |
| portdnses                               |
| portforwardings                         |
| portqueuemappings                       |
| ports                                   |
| portsecuritybindings                    |
| portuplinkstatuspropagation             |
| providerresourceassociations            |
| provisioningblocks                      |
| qos_bandwidth_limit_rules               |
| qos_dscp_marking_rules                  |
| qos_fip_policy_bindings                 |
| qos_minimum_bandwidth_rules             |
| qos_network_policy_bindings             |
| qos_policies                            |
| qos_policies_default                    |
| qos_port_policy_bindings                |
| qos_router_gw_policy_bindings           |
| qospolicyrbacs                          |
| qosqueues                               |
| quotas                                  |
| quotausages                             |
| reservations                            |
| resourcedeltas                          |
| router_extra_attributes                 |
| routerl3agentbindings                   |
| routerports                             |
| routerroutes                            |
| routerrules                             |
| routers                                 |
| securitygroupportbindings               |
| securitygrouprbacs                      |
| securitygrouprules                      |
| securitygroups                          |
| segmenthostmappings                     |
| serviceprofiles                         |
| sessionpersistences                     |
| standardattributes                      |
| subnet_service_types                    |
| subnetpoolprefixes                      |
| subnetpools                             |
| subnetroutes                            |
| subnets                                 |
| subports                                |
| tags                                    |
| trunks                                  |
| tz_network_bindings                     |
| vcns_router_bindings                    |
| vips                                    |
| vpnservices                             |
+-----------------------------------------+
172 rows in set (0.002 sec)

MariaDB [neutron]> quit
Bye
```

### **创建Neutron用户并分配角色**

导入环境变量模拟登陆（可以用source或者.）

```jsx
source admin-login
```

在“default”域中创建一个名为“neutron”密码为“123456”的用户

```jsx
openstack user create --domain default --password 123456 neutron

+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 0153be7dae864b54bae093671978ac1d |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

为用户“neutron”分配“admin”角色（赋予“neutron”用户操作“project”项目时的“admin”权限）

```jsx
openstack role add --project project --user neutron admin
```

创建一个名为“neutron”，类型为“network”的服务

```jsx
openstack service create --name neutron network

+---------+----------------------------------+
| Field   | Value                            |
+---------+----------------------------------+
| enabled | True                             |
| id      | 3768ce391cd34c268e6e30cf55786597 |
| name    | neutron                          |
| type    | network                          |
+---------+----------------------------------+
```

创建公众用户访问的端点

```jsx
openstack endpoint create --region RegionOne neutron public http://controller:9696

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 87079e18f15f47009a4f50b911fe9734 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 3768ce391cd34c268e6e30cf55786597 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

创建内部组件访问的端点

```jsx
openstack endpoint create --region RegionOne neutron internal http://controller:9696

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ead423861be84565a9c9806ee7742ae1 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 3768ce391cd34c268e6e30cf55786597 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

创建“admin”用户访问的端点

```jsx
openstack endpoint create --region RegionOne neutron admin http://controller:9696

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 525e44cf4a7647f68317ecc8b983a820 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 3768ce391cd34c268e6e30cf55786597 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

重启Nova服务并设置为开机自启动

```jsx
systemctl restart openstack-nova-api &&  systemctl enable openstack-nova-api
```

启动Neutron服务组件、网桥代理、DHCP代理、元数据代理并设置为开机自启动

```jsx
systemctl start neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent && systemctl enable neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent
```

查看端口占用情况

```jsx
netstat -tulup|grep 9696

tcp        0      0 0.0.0.0:9696            0.0.0.0:*               LISTEN      16974/server.log 
```

检验服务端点

```jsx
curl http://controller:9696

{"versions": [{"status": "CURRENT", "id": "v2.0", "links": [{"href": "http://controller:9696/v2.0/", "rel": "self"}]}]}
```

查看服务运行情况

```jsx
[root@controller ~]# systemctl status neutron-server
● neutron-server.service - OpenStack Neutron Server
   Loaded: loaded (/usr/lib/systemd/system/neutron-server.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2024-05-26 00:14:01 PDT; 8min ago
 Main PID: 16974 (/usr/bin/python)
   CGroup: /system.slice/neutron-server.service
           ├─16974 /usr/bin/python2 /usr/bin/neutron-server --config-file /usr/share/neutron/neutron-dist.conf --config-dir /usr/share/neutron/se...
           ├─17011 /usr/bin/python2 /usr/bin/neutron-server --config-file /usr/share/neutron/neutron-dist.conf --config-dir /usr/share/neutron/se...
           ├─17012 neutron-server: rpc worker (/usr/bin/python2 /usr/bin/neutron-server --config-file /usr/share/neutron/neutron-dist.conf --conf...
           ├─17013 neutron-server: rpc worker (/usr/bin/python2 /usr/bin/neutron-server --config-file /usr/share/neutron/neutron-dist.conf --conf...
           └─17014 neutron-server: periodic worker (/usr/bin/python2 /usr/bin/neutron-server --config-file /usr/share/neutron/neutron-dist.conf -...

May 26 00:13:57 controller systemd[1]: Stopped OpenStack Neutron Server.
May 26 00:13:57 controller systemd[1]: Starting OpenStack Neutron Server...
May 26 00:13:59 controller neutron-server[16974]: /usr/lib/python2.7/site-packages/paste/deploy/loadwsgi.py:22: PkgResourcesDeprecationWar...rately.
May 26 00:13:59 controller neutron-server[16974]: return pkg_resources.EntryPoint.parse("x=" + s).load(False)
May 26 00:14:01 controller systemd[1]: Started OpenStack Neutron Server.
Hint: Some lines were ellipsized, use -l to show in full.
```

### **2、安装compute节点Neutron服务：**

安装Neutron服务

```jsx
yum install -y openstack-neutron-linuxbridge
```

查看用户的信息

```jsx
[root@compute ~]# cat /etc/passwd | grep neutron
neutron:x:993:990:OpenStack Neutron Daemons:/var/lib/neutron:/sbin/nologin
```

查看用户组的信息

```jsx
[root@compute ~]# cat /etc/group | grep neutron
neutron:x:990:
```

### **修改Neutron服务相关配置文件**

**1.配置neutron组件信息**

备份配置文件

```jsx
cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
```

去掉配置文件中的注释行和空行，并生成新的配置文件

```jsx
grep -Ev '^$|#' /etc/neutron/neutron.conf.bak > /etc/neutron/neutron.conf
```

> E表示采用正则表达式进行读取；^$|#（^$ ^表示一行的开头，$表示一行的结尾，整体表示为匹配空行；| 这里表示或者； #表示注释行）v表示反向匹配

修改配置文件

```jsx
vi /etc/neutron/neutron.conf

[DEFAULT]
transport_url = rabbit://rabbitmq:123456@controller
auth_strategy = keystone
[cors]
[database]
[keystone_authtoken]
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = project
username = neutron
password = 123456
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[privsep]
[ssl]
```

```jsx
[DEFAULT] 部分
transport_url: 该配置项指定了Neutron使用RabbitMQ作为消息传输的中间件，并提供连接RabbitMQ的URL，包括用户名、密码和主机名。
auth_strategy: 认证策略。这里设置为keystone，表示使用OpenStack Keystone服务进行认证。
[cors] 部分
通常用于配置跨域资源共享（CORS）相关设置。

[database] 部分
通常用于配置Neutron数据库的连接信息。

[keystone_authtoken] 部分
auth_url: Keystone身份认证服务的URL。这里设置为http://controller:5000。
memcached_servers: Memcached服务器的地址，用于令牌缓存。这里设置为controller:11211。
auth_type: 认证类型。这里设置为password，表示使用密码认证。
project_domain_name: 项目域名。这里设置为default。
user_domain_name: 用户域名。这里设置为default。
project_name: 用于服务认证的项目名称。这里设置为project。
username: 用于认证的用户名。这里设置为neutron。
password: 用于认证的密码。这里设置为123456。
[oslo_concurrency] 部分
lock_path: 指定用于存储并发锁文件的路径，确保在高并发环境下Neutron的操作是线程安全的。
```

vif\_plugging\_is\_fatal = false vif\_plugging\_timeout = 0

**2.修改网桥代理（linuxbridge-agent）的配置文件：**

备份配置文件

```jsx
cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
```

去掉配置文件中的注释行和空行，并生成新的配置文件

```jsx
grep -Ev '^$|#' /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak > /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```

> E表示采用正则表达式进行读取；^$|#（^$ ^表示一行的开头，$表示一行的结尾，整体表示为匹配空行；| 这里表示或者； #表示注释行）v表示反向匹配

修改配置文件

```jsx
vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[DEFAULT]
[linux_bridge]
physical_interface_mappings = provider:ens33
[vxlan]
enable_vxlan = false
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.lptablesFirewallDriver
```

```jsx
[DEFAULT] 部分
通常用于定义全局配置参数。

[linux_bridge] 部分
physical_interface_mappings: 定义物理网络接口的映射。这里设置为provider:ens33，表示将名为provider的网络映射到物理接口ens33。
[vxlan] 部分
enable_vxlan: 指定是否启用VXLAN（虚拟扩展局域网）。这里设置为false，表示不启用VXLAN。表明你不需要在多租户环境中使用覆盖网络（overlay network）。
[securitygroup] 部分
enable_security_group: 指定是否启用安全组。这里设置为true，表示启用安全组，以控制流量的进出。
firewall_driver: 指定防火墙驱动程序。这里有一个小错误，应为neutron.agent.linux.iptables_firewall.IptablesFirewallDriver而非neutron.agent.linux.iptables_firewall.lptablesFirewallDriver。
```

**3.修改Nova的配置文件**

```jsx
vi /etc/nova/nova.conf

[DEFAULT]
vif_plugging_is_fatal = false
vif_plugging_timeout = 0

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = project
username = neutron
password = 123456
```

```jsx
[DEFAULT] 部分
vif_plugging_is_fatal: 设置虚拟接口插入失败是否视为致命错误。这里设置为false，表示如果虚拟接口插入失败，不会导致实例启动失败。
vif_plugging_timeout: 设置虚拟接口插入的超时时间。这里设置为0，表示没有超时限制。
[neutron] 部分
auth_url: Keystone身份认证服务的URL。这里设置为http://controller:5000。
auth_type: 认证类型。这里设置为password，表示使用密码认证。
project_domain_name: 项目域名。这里设置为default。
user_domain_name: 用户域名。这里设置为default。
region_name: 区域名称。这里设置为RegionOne。
project_name: 用于服务认证的项目名称。这里设置为project。
username: 用于认证的用户名。这里设置为neutron。
password: 用于认证的密码。这里设置为123456。
```

重启Nova服务并设置为开机自启动

```jsx
systemctl restart openstack-nova-compute && systemctl enable openstack-nova-compute
```

重启Neutron网桥代理服务并设置为开机自启动

```jsx
systemctl restart neutron-linuxbridge-agent && systemctl enable  neutron-linuxbridge-agent
```

导入环境变量模拟登陆（可以用source或者.）

```jsx
source admin-login
```

查看代理服务列表

```jsx
openstack network agent list

+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| b24af582-baf8-423c-a384-7b903a708d9b | DHCP agent         | controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
| c89c869d-213d-47de-9316-1c3b7908068f | Linux bridge agent | compute    | None              | :-)   | UP    | neutron-linuxbridge-agent |
| d5c5e1c3-8f3c-4716-a3d4-91811c157bc7 | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
| ee8c5443-a443-412c-94a7-9a7d4240f7cc | Linux bridge agent | controller | None              | :-)   | UP    | neutron-linuxbridge-agent |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
```

自检

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 35
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

查看本地和远程‘neutron’用户是否授予对‘keystone’数据库的完全控制权限
MariaDB [(none)]> show grants for neutron;
+--------------------------------------------------------------------------------------------------------+
| Grants for neutron@%                                                                                   |
+--------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'neutron'@'%' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9' |
| GRANT ALL PRIVILEGES ON `neutron`.* TO 'neutron'@'%'                                                   |
+--------------------------------------------------------------------------------------------------------+
2 rows in set (0.002 sec)

MariaDB [(none)]> quit
Bye

查看用户列表是否存在neutron
[root@controller ~]# openstack user list 

+----------------------------------+-----------+
| ID                               | Name      |
+----------------------------------+-----------+
| fcc523d60cc74e19ad13d4f7b029df70 | admin     |
| 7c4087ac47184343997d023de6d357d4 | glance    |
| 20f8cd5b95dc44cf98b43348b7f6124c | placement |
| 987f57db50b44896acd1c8fc337bf664 | nova      |
| 0cac9e15c6b145adb2e4f2117f4c5cbb | neutron   |
+----------------------------------+-----------+

查看角色ID
[root@controller ~]# openstack role list

+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| 4fb78ac4adff43ef8127afdf81f84694 | user   |
| 57adb6fe8ac04387853003b18c9c7240 | member |
| a794d5d1b9014757b8e61e7c97f0ff27 | admin  |
| dd525862a0ba4b3090b00e18436abdfd | reader |
+----------------------------------+--------+

查看neutron用户是否和admin角色绑定
[root@controller ~]# openstack role assignment list 

+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+
| Role                             | User                             | Group | Project                          | Domain | System | Inherited |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+
| a794d5d1b9014757b8e61e7c97f0ff27 | 0cac9e15c6b145adb2e4f2117f4c5cbb |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | 20f8cd5b95dc44cf98b43348b7f6124c |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | 7c4087ac47184343997d023de6d357d4 |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | 987f57db50b44896acd1c8fc337bf664 |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | fcc523d60cc74e19ad13d4f7b029df70 |       | a9001f880f424f79929111ef200030fc |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | fcc523d60cc74e19ad13d4f7b029df70 |       |                                  |        | all    | False     |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+

查看服务列表，是否创建了该服务
[root@controller ~]# openstack service list 

+----------------------------------+-----------+-----------+
| ID                               | Name      | Type      |
+----------------------------------+-----------+-----------+
| 223a2a4e7b034bcfb477526dad6eea36 | nova      | compute   |
| 4a1a975ce099473a9f8a0b56ba24bbcc | keystone  | identity  |
| b645abce902b46f9b1b4f39a80601cb1 | glance    | image     |
| e390256a7275423489bac1ed58e98ed5 | neutron   | network   |
| f517d649ec4b4f58996d800679698090 | placement | placement |
+----------------------------------+-----------+-----------+

查看此服务的3个服务端点是否建立
[root@controller ~]# openstack endpoint list

+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                         |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------+
| 1a2ed1898d1e4b5893e770ca0e0abd3d | RegionOne | nova         | compute      | True    | admin     | http://controller:8774/v2.1 |
| 2330a6b94c92439b89cd64e40b4fce61 | RegionOne | keystone     | identity     | True    | admin     | http://controller:5000/v3   |
| 41d7666d90de4d0fafaadf0422dc3432 | RegionOne | placement    | placement    | True    | public    | http://controller:8778      |
| 484f573d093545048cfe6d7de14c7609 | RegionOne | glance       | image        | True    | public    | http://controller:9292      |
| 5773cb1dd1e54ec59baf2c52a9afa629 | RegionOne | neutron      | network      | True    | public    | http://controller:9696      |
| 5eaacaa4019e49db865a97e9a728e133 | RegionOne | placement    | placement    | True    | admin     | http://controller:8778      |
| 6770c81a2b4e4de98ce89ff16ff69bd1 | RegionOne | glance       | image        | True    | internal  | http://controller:9292      |
| 7a5b6a7987884264bde8dcbbc746db9b | RegionOne | placement    | placement    | True    | internal  | http://controller:8778      |
| 91b53fd5293b44889afc309a7c676e41 | RegionOne | keystone     | identity     | True    | internal  | http://controller:5000/v3   |
| 925e27d07cb248489a0938626bad1d0a | RegionOne | keystone     | identity     | True    | public    | http://controller:5000/v3   |
| 9d9f077399454f599e58ccc0babbb8c8 | RegionOne | nova         | compute      | True    | public    | http://controller:8774/v2.1 |
| 9e8843cc9e1a4243889f163d3ee2bab5 | RegionOne | nova         | compute      | True    | internal  | http://controller:8774/v2.1 |
| b10654894de940278a7913e39a72c735 | RegionOne | neutron      | network      | True    | internal  | http://controller:9696      |
| cdeb0344b88845ceb5a5ee5b91427e79 | RegionOne | neutron      | network      | True    | admin     | http://controller:9696      |
| efc2299731994e5cbcafc7264c8d23c8 | RegionOne | neutron      | network      | True    | admin     | http://controller:9696      |
| fc5b67bed2304918bdf51630d6fc09f5 | RegionOne | glance       | image        | True    | admin     | http://controller:9292      |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------+
```

### Dashboard服务

安装dashboard服务

```jsx
yum install -y openstack-dashboard
```

配置dahboard配置文件

```jsx
vi /etc/openstack-dashboard/local_settings 

ALLOWED_HOSTS = ['*']
OPENSTACK_HOST = "controller"
TIME_ZONE = "Asia/Shanghai"

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
 'default':{
     'BACKEND':'django.core.cache.backends.memcached.MemcachedCache',
     'LOCATION':'controller:11211',
}
}

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
"identity":3,
"image":2,
"volume":3,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

OPENSTACK_NEUTRON_NETWORK = {
    'enable_auto_allocated_network': False,
    'enable_distributed_router': False,
    'enable_fip_topology_check': False,
    'enable_ha_router': False,
    'enable_ipv6': False,
    # TODO(amotoki): Drop OPENSTACK_NEUTRON_NETWORK completely from here.
    # enable_quotas has the different default value here.
    'enable_quotas': False,
    'enable_rbac_policy': False,
    'enable_router': False,

    'default_dns_nameservers': [],
    'supported_provider_types': ['*'],
    'segmentation_id_range': {},
    'extra_provider_types': {},
    'supported_vnic_types': ['*'],
    'physical_networks': [],

}
```

```jsx
ALLOWED_HOSTS: 允许的主机列表。设置为 ['*'] 表示允许所有主机访问，这在开发环境中是可以的，但在生产环境中应限制为特定的主机名或IP地址以提高安全性。
OPENSTACK_HOST: OpenStack控制节点的主机名。这里设置为 controller，表示Horizon将与主机名为 controller 的节点进行交互。
TIME_ZONE: 时区设置。这里设置为 Asia/Shanghai，表示使用上海时间。

SESSION_ENGINE: Django会话引擎。这里设置为 django.contrib.sessions.backends.cache，表示使用缓存来存储会话数据。
CACHES: Django缓存设置。
default: 缓存的默认配置。
BACKEND: 缓存后端，这里设置为 django.core.cache.backends.memcached.MemcachedCache，表示使用Memcached作为缓存后端。
LOCATION: Memcached服务器的位置，这里设置为 controller:11211，表示Memcached服务器运行在 controller 主机上的端口 11211。

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT: 是否支持多域。设置为 True，表示启用多域支持。

OPENSTACK_API_VERSIONS: 指定各个OpenStack服务的API版本。
identity: 身份认证服务（Keystone）的API版本，设置为 3。
image: 镜像服务（Glance）的API版本，设置为 2。
volume: 卷服务（Cinder）的API版本，设置为 3。

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN: Keystone的默认域名。这里设置为 Default。
OPENSTACK_KEYSTONE_DEFAULT_ROLE: Keystone的默认角色。这里设置为 user。

OPENSTACK_NEUTRON_NETWORK: Neutron网络服务的相关配置。
enable_auto_allocated_network: 是否启用自动分配网络。设置为 False。
enable_distributed_router: 是否启用分布式路由。设置为 False。
enable_fip_topology_check: 是否启用浮动IP拓扑检查。设置为 False。
enable_ha_router: 是否启用高可用路由器。设置为 False。
enable_ipv6: 是否启用IPv6。设置为 False。
enable_quotas: 是否启用配额。设置为 False。
enable_rbac_policy: 是否启用基于角色的访问控制策略。设置为 False。
enable_router: 是否启用路由器功能。设置为 False。
default_dns_nameservers: 默认的DNS服务器列表。这里设置为空列表。
supported_provider_types: 支持的提供者网络类型。这里设置为 ['*']，表示支持所有类型。
```

### 发布dashboard服务

重建dashboard的web应用的配置文件

进入dashboard网站目录

```jsx
cd /usr/share/openstack-dashboard/
```

编译生成dashboard的web服务配置文件

```jsx
python [manage.py](http://manage.py/) make_web_conf --apache > /etc/httpd/conf.d/openstack-dashboard.conf
```

查看生成的配置文件（dashboard需要的各种参数）

```jsx
cat /etc/httpd/conf.d/openstack-dashboard.conf

<VirtualHost *:80>

ServerAdmin webmaster@openstack.org
ServerName  openstack_dashboard

DocumentRoot /usr/share/openstack-dashboard/

LogLevel warn
ErrorLog /var/log/httpd/openstack_dashboard-error.log
CustomLog /var/log/httpd/openstack_dashboard-access.log combined

WSGIScriptReloading On
WSGIDaemonProcess openstack_dashboard_website processes=3
WSGIProcessGroup openstack_dashboard_website
WSGIApplicationGroup %{GLOBAL}
WSGIPassAuthorization On

WSGIScriptAlias / /usr/share/openstack-dashboard/openstack_dashboard/wsgi.py

<Location "/">
    Require all granted
</Location>

Alias /static /usr/share/openstack-dashboard/static
<Location "/static">
    SetHandler None
</Location>

</Virtualhost>
```

建立策略文件的软链接

查看该目录下的策略文件

```jsx
ls /etc/openstack-dashboard

cinder_policy.json  keystone_policy.json  neutron_policy.json  nova_policy.json
glance_policy.json  local_settings        nova_policy.d
```

为使其生效将其放入dashboard项目中（软链接）

```jsx
ln -s /etc/openstack-dashboard/ /usr/share/openstack-dashboard/openstack_dashboard/conf
```

查看网站目录

```jsx
ll /usr/share/openstack-dashboard/openstack_dashboard/

lrwxrwxrwx.  1 root root    25 May 29 04:01 conf -> /etc/openstack-dashboard/
```

启动apache服务器并设置为开机自启动

```jsx
systemctl restart httpd && systemctl enable httpd
```

检测dashboard服务

![Untitled](../../.gitbook/assets/Untitled.png)

### **Cinder服务**

### controller

安装**Cinder**服务

```jsx
yum install -y openstack-cinder
```

查看用户的信息

```jsx
[root@controller ~]# cat /etc/passwd | grep cinder

cinder:x:165:165:OpenStack Cinder Daemons:/var/lib/cinder:/sbin/nologin
```

查看用户组的信息

```jsx
[root@controller ~]# cat /etc/group | grep cinder

nobody:x:99:nova,cinder
cinder:x:165:cinder
```

进入数据库

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 46
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

新建“cinder”数据库
MariaDB [(none)]> CREATE DATABASE cinder;
Query OK, 1 row affected (0.001 sec)

为数据库进行授权（授权“cinder”用户从本地和远程管理“cinder”数据库的权限）
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO "cinder"@"localhost" IDENTIFIED BY "123456";
Query OK, 0 rows affected (0.020 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO "cinder"@"%" IDENTIFIED BY "123456";
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> quit
Bye
```

备份配置文件

```jsx
cp /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
```

去掉配置文件中的注释行和空行，并生成新的配置文件

```jsx
grep -Ev '^$|#' /etc/cinder/cinder.conf.bak > /etc/cinder/cinder.conf
```

> E表示采用正则表达式进行读取；^$|#（^$ ^表示一行的开头，$表示一行的结尾，整体表示为匹配空行；| 这里表示或者； #表示注释行）v表示反向匹配

修改配置文件

```jsx
vi /etc/cinder/cinder.conf

[DEFAULT]
auth_strategy = keystone
transport_url = rabbit://rabbitmq:123456@controller:5672
[backend]
[backend_defaults]
[barbican]
[brcd_fabric_example]
[cisco_fabric_example]
[coordination]
[cors]
[database]
connection = mysql+pymysql://cinder:123456@controller/cinder 
[fc-zone-manager]
[healthcheck]
[key_manager]
[keystone_authtoken]
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_pyte = password
project_domain_name = Default
user_domain_name = Default
project_name = project
username = cinder
password = 123456
[nova]
[oslo_concurrency]
look_path = /var/lib/cinder/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[oslo_versionedobjects]
[privsep]
[profiler]
[sample_castellan_source]
[sample_remote_file_source]
[service_user]
[ssl]
[vault]
```

```jsx
[DEFAULT] 部分
auth_strategy = keystone：指定使用 Keystone 进行认证。
transport_url = rabbit://rabbitmq:123456@controller:5672：定义了与 RabbitMQ 消息代理的连接 URL，包括用户名 rabbitmq、密码 123456 和 RabbitMQ 服务器 controller 的端口 5672。注意：密码以明文形式存在，存在安全风险。
[backend] 部分
配置后端存储的相关设置。
[backend_defaults] 部分
用于定义后端存储的默认设置。
[barbican] 部分
配置 Barbican（OpenStack 的密钥管理服务）的相关设置。
[brcd_fabric_example] 和 [cisco_fabric_example] 部分
用来配置与 Brocade 和 Cisco 相关的 Fabric 设置。
[coordination] 部分
用于配置协调服务。
[cors] 部分
用于配置跨域资源共享（CORS）设置。
[database] 部分
connection = mysql+pymysql://cinder:123456@controller/cinder：定义了与 MySQL 数据库的连接字符串，包括用户名 cinder、密码 123456 和数据库服务器 controller。数据库名为 cinder。同样，密码以明文形式存在，应该加密处理以确保安全。
[fc-zone-manager] 部分
用于配置 FC Zone Manager。
[healthcheck] 部分
用于配置健康检查。
[key_manager] 部分
用于配置密钥管理。
[keystone_authtoken] 部分
auth_url = http://controller:5000：Keystone 认证服务的 URL。
memcached_servers = controller:11211：Memcached 服务器地址。
auth_pyte = password：这里有拼写错误，应该是 auth_type = password。
project_domain_name = Default：项目域名，默认为 Default。
user_domain_name = Default：用户域名，默认为 Default。
project_name = project：项目名称，建议使用更具体的名称。
username = cinder：用于认证的用户名。
password = 123456：用于认证的密码
[nova] 部分
用于配置与 Nova 的集成。
[oslo_concurrency] 部分
lock_path = /var/lib/cinder/tmp：锁文件路径，用于进程间同步。
[oslo_messaging_amqp], [oslo_messaging_kafka], [oslo_messaging_notifications], [oslo_messaging_rabbit] 部分
用于配置不同的消息传递后端（如 AMQP、Kafka、通知和 RabbitMQ）。
[oslo_middleware], [oslo_policy], [oslo_reports], [oslo_versionedobjects] 部分
用于配置中间件、策略、报告和版本化对象。
[privsep] 部分
用于配置特权分离。
[profiler] 部分
用于配置性能分析。
[sample_castellan_source] 和 [sample_remote_file_source] 部分
通常用作示例配置。
[service_user] 部分
用于配置服务用户。
[ssl] 部分
用于配置 SSL 相关设置。
[vault] 部分
用于配置与 Vault 集成。
```

添加nova配置文件实现cinder与nova的交互

```jsx
vi /etc/nova/nova.conf

[cinder]
os_region_name = RegionOne
```

初始化数据库

```jsx
su cinder -s /bin/sh -c "cinder-manage db sync"

Deprecated: Option "logdir" from group "DEFAULT" is deprecated. Use option "log-dir" from group "DEFAULT".
```

检查数据库

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 63
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

使用cinder数据库
MariaDB [(none)]> use cinder
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed

查看cinder数据库的表
MariaDB [cinder]> show tables;
+----------------------------+
| Tables_in_cinder           |
+----------------------------+
| attachment_specs           |
| backup_metadata            |
| backups                    |
| cgsnapshots                |
| clusters                   |
| consistencygroups          |
| driver_initiator_data      |
| encryption                 |
| group_snapshots            |
| group_type_projects        |
| group_type_specs           |
| group_types                |
| group_volume_type_mapping  |
| groups                     |
| image_volume_cache_entries |
| messages                   |
| migrate_version            |
| quality_of_service_specs   |
| quota_classes              |
| quota_usages               |
| quotas                     |
| reservations               |
| services                   |
| snapshot_metadata          |
| snapshots                  |
| transfers                  |
| volume_admin_metadata      |
| volume_attachment          |
| volume_glance_metadata     |
| volume_metadata            |
| volume_type_extra_specs    |
| volume_type_projects       |
| volume_types               |
| volumes                    |
| workers                    |
+----------------------------+
35 rows in set (0.001 sec)

MariaDB [cinder]> quit
Bye
```

导入环境变量模拟登录

```jsx
source admin-login
```

在“default”域中创建一个名为“cinder“，密码为”123456“的用户

```jsx
openstack user create --domain default --password 123456 cinder

+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 3afe6e7f77474542acb2798a208ed2e6 |
| name                | cinder                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

授予”cinder“用户操作”project“项目时的”admin“的权限

```jsx
openstack role add --project project --user cinder admin
```

创建一个名为”cinderv3“，类型为”volumev3“的服务

```jsx
openstack service create --name cinderv3 volumev3

+---------+----------------------------------+
| Field   | Value                            |
+---------+----------------------------------+
| enabled | True                             |
| id      | 2c1305308dfe44ce9d8dd8b1c6d3b393 |
| name    | cinderv3                         |
| type    | volumev3                         |
+---------+----------------------------------+
```

创建公众用户访问的端点

```jsx
openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s

+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | c4d59156b51b4c1eb4c113720f820068         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 2c1305308dfe44ce9d8dd8b1c6d3b393         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+
```

创建内部组件访问的端点

```jsx
openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s

+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | b53397d529a3475dadd30584c799cb4b         |
| interface    | internal                                 |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 2c1305308dfe44ce9d8dd8b1c6d3b393         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+
```

创建“admin”用户访问的端点

```jsx
openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s

+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 08629fb0d892462eb2523daf490f44c9         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 2c1305308dfe44ce9d8dd8b1c6d3b393         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+
```

重启Nova服务并设置为开机自启动

```jsx
systemctl restart openstack-nova-api &&  systemctl enable openstack-nova-api
```

启动cinder-api和cinder-scheduler并设置为开机自启动

```jsx
systemctl restart openstack-cinder-api && systemctl enable openstack-cinder-api && systemctl restart openstack-cinder-scheduler && systemctl enable openstack-cinder-scheduler
```

查看端口占用情况

```jsx
netstat -nutpl | grep 8776

tcp        0      0 0.0.0.0:8776            0.0.0.0:*               LISTEN      7077/python2 
```

查看存储服务列表

```jsx
openstack volume service list

+------------------+------------+------+---------+-------+----------------------------+
| Binary           | Host       | Zone | Status  | State | Updated At                 |
+------------------+------------+------+---------+-------+----------------------------+
| cinder-scheduler | controller | nova | enabled | up    | 2024-05-29T12:11:53.000000 |
+------------------+------------+------+---------+-------+----------------------------+
```

### compute

为计算节点添加一块硬盘（sdb）

创建卷组

查看系统中所有硬盘信息

```jsx
[root@compute ~]# lsblk

NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk 
├─sda1   8:1    0  300M  0 part /boot
├─sda2   8:2    0  3.5G  0 part [SWAP]
└─sda3   8:3    0 36.2G  0 part /
sdb      8:16   0   20G  0 disk 
```

创建LVM物理卷组

将物理硬盘初始化为物理卷

```jsx
pvcreate /dev/sdb
```

将物理卷归并为卷组

```jsx
vgcreate cinder-volumes /dev/sdb
```

配置LVM卷组扫描的设备（直接指定仅对哪些磁盘设备进行扫描，提高效率）

```jsx
vi /etc/lvm/lvm.conf 

filter = [ "a/sdb/", "r/.*/" ]
```

> 接受/dev/sdb磁盘并拒绝其他设备的过滤器；“a”表示接受，“r”表示拒绝

启动LVM元数据服务并设置为开机自启动

```jsx
systemctl restart lvm2-lvmetad && systemctl enable lvm2-lvmetad
```

安装cinder服务

```jsx
yum install -y openstack-cinder targetcli python-keystone
```

备份配置文件

```jsx
cp /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
```

去掉配置文件中的注释行和空行，并生成新的配置文件

```jsx
grep -Ev '^$|#' /etc/cinder/cinder.conf.bak > /etc/cinder/cinder.conf
```

> E表示采用正则表达式进行读取；^$|#（^$ ^表示一行的开头，$表示一行的结尾，整体表示为匹配空行；| 这里表示或者； #表示注释行）v表示反向匹配

修改配置文件

```jsx
vi /etc/cinder/cinder.conf

[DEFAULT]
auth_strategy = keystone
transport_url = rabbit://rabbitmq:123456@controller:5672
glance_api_servers = http://controller:9292
enabled_backends = lvm
[backend]
[backend_defaults]
[barbican]
[brcd_fabric_example]
[cisco_fabric_example]
[coordination]
[cors]
[database]
connection = mysql+pymysql://cinder:123456@controller/cinder
[fc-zone-manager]
[healthcheck]
[key_manager]
[keystone_authtoken]
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = project
username = cinder
password = 123456
[nova]
[oslo_concurrency]
lock_path  = /var/lib/cinder/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[oslo_versionedobjects]
[privsep]
[profiler]
[sample_castellan_source]
[sample_remote_file_source]
[service_user]
[ssl]
[vault]
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = lioadm
```

```jsx
[DEFAULT] 部分
auth_strategy = keystone：指定使用 Keystone 进行认证。
transport_url = rabbit://rabbitmq:123456@controller:5672：定义了 RabbitMQ 消息代理的连接 URL，包括用户名 rabbitmq、密码 123456 和服务器 controller 的端口 5672。注意：密码以明文形式存在，存在安全风险。
glance_api_servers = http://controller:9292：指定 Glance 服务的 API 服务器地址。
enabled_backends = lvm：启用的后端存储类型，这里指定为 lvm。
[backend] 部分
用于配置后端存储的具体设置。
[backend_defaults] 部分
用于定义后端存储的默认设置。
[barbican] 部分
用于配置 Barbican（OpenStack 的密钥管理服务）的相关设置。
[brcd_fabric_example] 和 [cisco_fabric_example] 部分
用于配置与 Brocade 和 Cisco 相关的 Fabric 设置。
[coordination] 部分
用于配置协调服务。
[cors] 部分
用于配置跨域资源共享（CORS）设置。
[database] 部分
connection = mysql+pymysql://cinder:123456@controller/cinder：定义了与 MySQL 数据库的连接字符串，包括用户名 cinder、密码 123456 和数据库服务器 controller。数据库名为 cinder。
[fc-zone-manager] 部分
用于配置 FC Zone Manager。
[healthcheck] 部分
用于配置健康检查。
[key_manager] 部分
用于配置密钥管理。
[keystone_authtoken] 部分
auth_url = http://controller:5000：Keystone 认证服务的 URL。
memcached_servers = controller:11211：Memcached 服务器地址。
auth_type = password：认证类型，使用密码认证。
project_domain_name = Default：项目域名，默认为 Default。
user_domain_name = Default：用户域名，默认为 Default。
project_name = project：项目名称，
username = cinder：用于认证的用户名。
password = 123456：用于认证的密码，
[nova] 部分
用于配置与 Nova 的集成。
[oslo_concurrency] 部分
lock_path = /var/lib/cinder/tmp：锁文件路径，用于进程间同步。
[oslo_messaging_amqp], [oslo_messaging_kafka], [oslo_messaging_notifications], [oslo_messaging_rabbit] 部分
用于配置不同的消息传递后端（如 AMQP、Kafka、通知和 RabbitMQ）。
[oslo_middleware], [oslo_policy], [oslo_reports], [oslo_versionedobjects] 部分
用于配置中间件、策略、报告和版本化对象。
[privsep] 部分
用于配置特权分离。
[profiler] 部分
用于配置性能分析。
[sample_castellan_source] 和 [sample_remote_file_source] 部分
通常用作示例配置。
[service_user] 部分
用于配置服务用户。
[ssl] 部分
用于配置 SSL 相关设置。
[vault] 部分
用于配置与 Vault 集成。
[lvm] 部分
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver：指定使用 LVM 作为存储后端的驱动。
volume_group = cinder-volumes：LVM 的卷组名为 cinder-volumes。
target_protocol = iscsi：指定使用 iSCSI 作为目标协议。
target_helper = lioadm：指定使用 lioadm 作为目标辅助工具。
```

启动cinder-api和cinder-scheduler并设置为开机自启动

```jsx
systemctl restart openstack-cinder-volume target && systemctl enable openstack-cinder-volume target
```

controller

查看存储服务列表

```jsx
openstack volume service list

+------------------+-------------+------+---------+-------+----------------------------+
| Binary           | Host        | Zone | Status  | State | Updated At                 |
+------------------+-------------+------+---------+-------+----------------------------+
| cinder-scheduler | controller  | nova | enabled | up    | 2024-05-30T06:33:51.000000 |
| cinder-volume    | compute@lvm | nova | enabled | up    | 2024-05-30T06:33:49.000000 |
+------------------+-------------+------+---------+-------+----------------------------+
```

通过dashboard查看卷概况

用cinder创建卷

1、使用命令创建（创建一个8GD的卷，名称为volume1）

```jsx
openstack volume create --size 8 volume1
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2024-05-30T06:36:37.000000           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | c1c7545b-2210-4a08-bb16-ba3ad64c9bea |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | volume1                              |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 8                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | __DEFAULT__                          |
| updated_at          | None                                 |
| user_id             | fcc523d60cc74e19ad13d4f7b029df70     |
+---------------------+--------------------------------------+
```

查看卷列表

```jsx
openstack volume list

+--------------------------------------+---------+-----------+------+-------------+
| ID                                   | Name    | Status    | Size | Attached to |
+--------------------------------------+---------+-----------+------+-------------+
| c1c7545b-2210-4a08-bb16-ba3ad64c9bea | volume1 | available |    8 |             |
+--------------------------------------+---------+-----------+------+-------------+
```

2、使用dashboard创建卷

自检

```jsx
[root@controller ~]# mysql -uroot -p123456
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 62
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

查看用户“cinder”是否授予了远程和本地对“cinder”数据库的完全控制权限
MariaDB [(none)]> show grants for cinder;
+-------------------------------------------------------------------------------------------------------+
| Grants for cinder@%                                                                                   |
+-------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'cinder'@'%' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9' |
| GRANT ALL PRIVILEGES ON `cinder`.* TO 'cinder'@'%'                                                    |
+-------------------------------------------------------------------------------------------------------+
2 rows in set (0.002 sec)

MariaDB [(none)]> quit
Bye

查看“cinder”用户是否存在
[root@controller ~]# openstack user list

+----------------------------------+-----------+
| ID                               | Name      |
+----------------------------------+-----------+
| fcc523d60cc74e19ad13d4f7b029df70 | admin     |
| 7c4087ac47184343997d023de6d357d4 | glance    |
| 20f8cd5b95dc44cf98b43348b7f6124c | placement |
| 987f57db50b44896acd1c8fc337bf664 | nova      |
| 0cac9e15c6b145adb2e4f2117f4c5cbb | neutron   |
| 3afe6e7f77474542acb2798a208ed2e6 | cinder    |
+----------------------------------+-----------+

查看角色ID
[root@controller ~]# openstack role list

+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| 4fb78ac4adff43ef8127afdf81f84694 | user   |
| 57adb6fe8ac04387853003b18c9c7240 | member |
| a794d5d1b9014757b8e61e7c97f0ff27 | admin  |
| dd525862a0ba4b3090b00e18436abdfd | reader |
+----------------------------------+--------+

查看“cinder”用户是否和“admin”角色绑定
[root@controller ~]# openstack role assignment list

+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+
| Role                             | User                             | Group | Project                          | Domain | System | Inherited |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+
| a794d5d1b9014757b8e61e7c97f0ff27 | 0cac9e15c6b145adb2e4f2117f4c5cbb |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | 20f8cd5b95dc44cf98b43348b7f6124c |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | 3afe6e7f77474542acb2798a208ed2e6 |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | 7c4087ac47184343997d023de6d357d4 |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | 987f57db50b44896acd1c8fc337bf664 |       | b8b677d25ac04cdd950c0782050aed60 |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | fcc523d60cc74e19ad13d4f7b029df70 |       | a9001f880f424f79929111ef200030fc |        |        | False     |
| a794d5d1b9014757b8e61e7c97f0ff27 | fcc523d60cc74e19ad13d4f7b029df70 |       |                                  |        | all    | False     |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+--------+-----------+

查看服务列表
[root@controller ~]# openstack service list

+----------------------------------+-----------+-----------+
| ID                               | Name      | Type      |
+----------------------------------+-----------+-----------+
| 223a2a4e7b034bcfb477526dad6eea36 | nova      | compute   |
| 2c1305308dfe44ce9d8dd8b1c6d3b393 | cinderv3  | volumev3  |
| 4a1a975ce099473a9f8a0b56ba24bbcc | keystone  | identity  |
| b645abce902b46f9b1b4f39a80601cb1 | glance    | image     |
| e390256a7275423489bac1ed58e98ed5 | neutron   | network   |
| f517d649ec4b4f58996d800679698090 | placement | placement |
+----------------------------------+-----------+-----------+

查看“cinder”的服务端点是否建立
[root@controller ~]# openstack catalog list

+-----------+-----------+------------------------------------------------------------------------+
| Name      | Type      | Endpoints                                                              |
+-----------+-----------+------------------------------------------------------------------------+
| nova      | compute   | RegionOne                                                              |
|           |           |   admin: http://controller:8774/v2.1                                   |
|           |           | RegionOne                                                              |
|           |           |   public: http://controller:8774/v2.1                                  |
|           |           | RegionOne                                                              |
|           |           |   internal: http://controller:8774/v2.1                                |
|           |           |                                                                        |
| cinderv3  | volumev3  | RegionOne                                                              |
|           |           |   admin: http://controller:8776/v3/a9001f880f424f79929111ef200030fc    |
|           |           | RegionOne                                                              |
|           |           |   internal: http://controller:8776/v3/a9001f880f424f79929111ef200030fc |
|           |           | RegionOne                                                              |
|           |           |   public: http://controller:8776/v3/a9001f880f424f79929111ef200030fc   |
|           |           |                                                                        |
| keystone  | identity  | RegionOne                                                              |
|           |           |   admin: http://controller:5000/v3                                     |
|           |           | RegionOne                                                              |
|           |           |   internal: http://controller:5000/v3                                  |
|           |           | RegionOne                                                              |
|           |           |   public: http://controller:5000/v3                                    |
|           |           |                                                                        |
| glance    | image     | RegionOne                                                              |
|           |           |   public: http://controller:9292                                       |
|           |           | RegionOne                                                              |
|           |           |   internal: http://controller:9292                                     |
|           |           | RegionOne                                                              |
|           |           |   admin: http://controller:9292                                        |
|           |           |                                                                        |
| neutron   | network   | RegionOne                                                              |
|           |           |   public: http://controller:9696                                       |
|           |           | RegionOne                                                              |
|           |           |   internal: http://controller:9696                                     |
|           |           | RegionOne                                                              |
|           |           |   admin: http://controller:9696                                        |
|           |           | RegionOne                                                              |
|           |           |   admin: http://controller:9696                                        |
|           |           |                                                                        |
| placement | placement | RegionOne                                                              |
|           |           |   public: http://controller:8778                                       |
|           |           | RegionOne                                                              |
|           |           |   admin: http://controller:8778                                        |
|           |           | RegionOne                                                              |
|           |           |   internal: http://controller:8778                                     |
|           |           |                                                                        |
+-----------+-----------+------------------------------------------------------------------------+

查看“cinder”服务是否正常运行
[root@controller ~]# cinder service-list

+------------------+-------------+------+---------+-------+----------------------------+---------+-----------------+---------------+
| Binary           | Host        | Zone | Status  | State | Updated_at                 | Cluster | Disabled Reason | Backend State |
+------------------+-------------+------+---------+-------+----------------------------+---------+-----------------+---------------+
| cinder-scheduler | controller  | nova | enabled | up    | 2024-05-30T07:01:41.000000 | -       | -               |               |
| cinder-volume    | compute@lvm | nova | enabled | up    | 2024-05-30T07:01:43.000000 | -       | -               | up            |
+------------------+-------------+------+---------+-------+----------------------------+---------+-----------------+---------------+
```
