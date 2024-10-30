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

# statefulset 一直重启



## 1、查看statefulset及pod的状态，可以发现一直在重启

```
kubectl get all

pod/es-cluster-0                       0/1     CrashLoopBackOff   9 (109s ago)   66m
pod/es-cluster-1                       0/1     CrashLoopBackOff   8 (100s ago)   28m
pod/es-cluster-2                       0/1     CrashLoopBackOff   8 (73s ago)    22m

statefulset.apps/es-cluster   0/3     66m
```

## 2、查看pod的日志发现elasticsearch 容器一直重启失败

```
kubectl describe pod/es-cluster-0   # 查看pod的日志
  
Warning  BackOff           3m2s (x95 over 27m)  kubelet            Back-off restarting failed container elasticsearch in pod es-cluster-0_default(f9523b61-bc45-495c-9399-490ae9a9cea9)
```

## 3、查看容器elasticsearch的日志

```
 kubectl logs -f es-cluster-2 -c elasticsearch
```

```
{"type": "server", "timestamp": "2024-10-29T08:28:22,633+0000", "level": "INFO", "component": "o.e.x.s.a.s.FileRolesStore", "cluster.name": "docker-cluster", "node.name": "es-cluster-2",  "message": "parsed [0] roles from file [/usr/share/elasticsearch/config/roles.yml]"  }
{"type": "server", "timestamp": "2024-10-29T08:28:24,967+0000", "level": "INFO", "component": "o.e.x.m.p.l.CppLogMessageHandler", "cluster.name": "docker-cluster", "node.name": "es-cluster-2",  "message": "[controller/87] [Main.cc@110] controller (64 bit): Version 7.2.0 (Build 65aefcbfce449b) Copyright (c) 2019 Elasticsearch BV"  }
{"type": "server", "timestamp": "2024-10-29T08:28:26,318+0000", "level": "DEBUG", "component": "o.e.a.ActionModule", "cluster.name": "docker-cluster", "node.name": "es-cluster-2",  "message": "Using REST wrapper from plugin org.elasticsearch.xpack.security.Security"  }
{"type": "server", "timestamp": "2024-10-29T08:28:27,192+0000", "level": "INFO", "component": "o.e.d.DiscoveryModule", "cluster.name": "docker-cluster", "node.name": "es-cluster-2",  "message": "using discovery type [zen] and seed hosts providers [settings]"  }
{"type": "server", "timestamp": "2024-10-29T08:28:28,937+0000", "level": "INFO", "component": "o.e.n.Node", "cluster.name": "docker-cluster", "node.name": "es-cluster-2",  "message": "initialized"  }
{"type": "server", "timestamp": "2024-10-29T08:28:28,938+0000", "level": "INFO", "component": "o.e.n.Node", "cluster.name": "docker-cluster", "node.name": "es-cluster-2",  "message": "starting ..."  }
{"type": "server", "timestamp": "2024-10-29T08:28:29,213+0000", "level": "INFO", "component": "o.e.t.TransportService", "cluster.name": "docker-cluster", "node.name": "es-cluster-2",  "message": "publish_address {10.244.104.54:9300}, bound_addresses {[::]:9300}"  }
{"type": "server", "timestamp": "2024-10-29T08:28:29,234+0000", "level": "INFO", "component": "o.e.b.BootstrapChecks", "cluster.name": "docker-cluster", "node.name": "es-cluster-2",  "message": "bound or publishing to a non-loopback address, enforcing bootstrap checks"  }
ERROR: [2] bootstrap checks failed
[1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[2]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
{"type": "server", "timestamp": "2024-10-29T08:28:29,348+0000", "level": "INFO", "component": "o.e.n.Node", "cluster.name": "docker-cluster", "node.name": "es-cluster-2",  "message": "stopping ..."  }
{"type": "server", "timestamp": "2024-10-29T08:28:29,402+0000", "level": "INFO", "component": "o.e.n.Node", "cluster.name": "docker-cluster", "node.name": "es-cluster-2",  "message": "stopped"  }
{"type": "server", "timestamp": "2024-10-29T08:28:29,403+0000", "level": "INFO", "component": "o.e.n.Node", "cluster.name": "docker-cluster", "node.name": "es-cluster-2",  "message": "closing ..."  }
{"type": "server", "timestamp": "2024-10-29T08:28:29,435+0000", "level": "INFO", "component": "o.e.n.Node", "cluster.name": "docker-cluster", "node.name": "es-cluster-2",  "message": "closed"  }
```

## 4、根据日志可以发现Elasticsearch 启动失败的原因主要有两个：

1. **虚拟内存设置过低**：（每个节点都要运行）
   *   错误信息指出 `vm.max_map_count` 的值太低，建议至少设置为 `262144`。你可以通过以下命令临时更改这个值：

       ```bash
       sudo sysctl -w vm.max_map_count=262144
       ```
   *   如果想要永久生效，可以将该设置添加到 `/etc/sysctl.conf` 文件中：

       ```bash
       vm.max_map_count=262144
       ```

       然后运行以下命令使配置生效：

       ```bash
       sudo sysctl -p
       ```
2. **集群发现配置不适合生产使用**：

*   你需要为 Elasticsearch 配置至少一个以下选项：`discovery.seed_hosts`、`discovery.seed_providers` 或 `cluster.initial_master_nodes`。对于 StatefulSet，通常使用 `discovery.seed_hosts` 配置集群中的节点地址。例如，你可以在 `elasticsearch` 容器的环境变量中添加如下配置：（直接修改上面创建的 StatefulSet文件即可）

    ```yaml
    env:
      - name: discovery.seed_hosts
        value: "es-cluster-0,es-cluster-1,es-cluster-2"  # 节点的名称或 IP 地址
      - name: cluster.initial_master_nodes
        value: "es-cluster-0,es-cluster-1,es-cluster-2"
    ```

在更新完这些配置后，重新启动你的 StatefulSet

```bash
kubectl delete pod -l app=elasticsearch
```

## 5、查看statefulset及pod的状态

```
pod/es-cluster-0                      1/1     Running            0             2m20s
pod/es-cluster-1                      1/1     Running            0             2m13s
pod/es-cluster-2                      1/1     Running            0             2m2s

statefulset.apps/es-cluster   3/3     2m52s
```
