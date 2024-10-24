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

# 创建node1节点时，出现以下问题

```
[root@ceph-master ceph]# ceph-deploy new ceph-master ceph-node1 ceph-node2
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.25): /usr/bin/ceph-deploy new ceph-master ceph-node1 ceph-node2
[ceph_deploy.new][DEBUG ] Creating new cluster named ceph
[ceph_deploy.new][INFO  ] making sure passwordless SSH succeeds
[ceph_deploy][ERROR ] Traceback (most recent call last):
[ceph_deploy][ERROR ]   File "/usr/lib/python2.7/site-packages/ceph_deploy/util/decorators.py", line 69, in newfunc
[ceph_deploy][ERROR ]     return f(*a, **kw)
[ceph_deploy][ERROR ]   File "/usr/lib/python2.7/site-packages/ceph_deploy/cli.py", line 162, in _main
[ceph_deploy][ERROR ]     return args.func(args)
[ceph_deploy][ERROR ]   File "/usr/lib/python2.7/site-packages/ceph_deploy/new.py", line 141, in new
[ceph_deploy][ERROR ]     ssh_copy_keys(host, args.username)
[ceph_deploy][ERROR ]   File "/usr/lib/python2.7/site-packages/ceph_deploy/new.py", line 35, in ssh_copy_keys
[ceph_deploy][ERROR ]     if ssh.can_connect_passwordless(hostname):
[ceph_deploy][ERROR ]   File "/usr/lib/python2.7/site-packages/ceph_deploy/util/ssh.py", line 15, in can_connect_passwordless
[ceph_deploy][ERROR ]     if not remoto.connection.needs_ssh(hostname):
[ceph_deploy][ERROR ] AttributeError: 'module' object has no attribute 'needs_ssh'
[ceph_deploy][ERROR ] 
```

由于缺少依赖环境问题，重装ceph-deploy

```
sudo pip3 uninstall ceph-deploy
sudo pip3 install ceph-deploy     # 默认安装到/usr/local/bin目录下，没有环境变量
export PATH=$PATH:/usr/local/bin   # 临时添加
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc    # 永久添加
source ~/.bashrc
```

之后出现（错误原因是由于ceph-node1和ceph-node2没有python环境）

```
[ceph-node1][ERROR ] Can't communicate with remote host, possibly because python3 is not installed there
[ceph_deploy][ERROR ] RuntimeError: connecting to host: ceph-node1 resulted in errors: OSError cannot send (already closed?)

[ceph-node2][ERROR ] Can't communicate with remote host, possibly because python3 is not installed there
[ceph_deploy][ERROR ] RuntimeError: connecting to host: ceph-node2 resulted in errors: OSError cannot send (already closed?)
```

在ceph-node1和ceph-node2安装python环境

```
yum install python3 -y
```
