---
title: 安装并配置 K3S 集群
tags:
  - Kubernetes
  - k3s
  - 个人备忘
date: 2020-08-05 15:16:11
---


k3s 默认使用 containerd 作为容器运行时, 因为我更熟悉 docker, 所以打算在 k3s 中使用 docker 代替 containerd.

先升级一下系统

```bash
apt update
apt upgrade -y
```

在系统中安装 docker, 直接使用 apt 安装

```bash
apt install docker.io
```

然后安装 k3s 的 master 节点

```bash
curl -sfL https://get.k3s.io | sh -s - --docker
```

安装完成后, 从 master 节点上的 ```/var/lib/rancher/k3s/server/node-token``` 获取到 token 值. 然后用 token 值安装 node 节点

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://server:6443 K3S_TOKEN=token sh -s - --docker
```

给 node 节点添加 role.

```bash
kubectl label node node node-role.kubernetes.io/worker=worker
```

计划中 master 和 node 将使用两个网络出口, 给两种服务提供支撑, 所以我们需要给节点打标记来让应用可以正确的部署到对应的节点中.

```bash
kubectl label nodes master network=proxy
kubectl label nodes node network=direct
```