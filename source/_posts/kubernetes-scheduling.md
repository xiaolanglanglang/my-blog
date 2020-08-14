---
title: Kubernetes 的标签和容忍度
tags:
  - Kubernetes
  - 标签选择器
  - 容忍度
  - Taint
  - 个人备忘
date: 2020-08-14 10:09:27
---


按照我之前的规划, 我的两个节点的 K3S 集群中, 应当有且仅有与下载相关的容器被分配到 node 节点上. 为了实现这一目标, 我需要给节点添加标签和污点(Taint).

<!-- more -->

# 容忍度

污点有两种效果(effect), 当值为 ```NoSchedule``` 时, 不能容忍该污点的容器就不会被分配到这个节点上, 当值为 ```PreferNoSchedule``` 时, 不能容忍该污点的容器会尽可能的不被分配到这个节点上.

因为我只希望下载相关的容器被分配到 node 节点, 所以这里使用 ```NoSchedule``` 效果.

```bash
kubectl taint nodes node network=direct:NoSchedule
```

这样, 只要我们不给非下载的容器添加对应的容忍度声明, 它们就不会被路由到 node 节点上. 而对于下载相关的容器来说, 只需要添加如下所示的容忍度信息, 即可被分配到 node 节点上.

```yaml
tolerations:
- key: "network"
  operator: "Equal"
  value: "direct"
  effect: "NoSchedule"
```


# 标签选择器

使用了污点后, Kubernetes 保证了仅有下载相关的容器可以被分配到 node 节点上, 但此时下载容器还有可能被分配到其他的节点中, 所以还需要使用标签选择器来确保下载相关的容器只会被分配到 node 节点上.

首先需要使用 label 指令给 node 节点添加标签

```yaml
kubectl label nodes node network=direct
```

> 这一步其实在{% post_link install-and-config-k3s %}时已经执行过.

然后对需要被路由到添加标签选择器(nodeSelector)即可分配到上述节点:

```yaml
nodeSelector:
  network: direct
```

完整的 yaml 文件可在 {% post_link kubernetes-config-argo-tunnel %} 中看到, 只不过它是分配到标记了另一个标签的节点上.