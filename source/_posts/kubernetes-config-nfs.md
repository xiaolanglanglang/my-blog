---
title: Kubernetes 配置 NFS 服务
tags:
  - ubuntu
  - nfs
  - linux
  - Kubernetes
  - 个人备忘
date: 2020-08-11 13:31:50
---

Kubernetes 要使用 NFS 协议提供卷储存的功能, 首先得有一个 NFS 服务提供者, 如果还没有的话, 需要先配置一个 NFS 服务.

# 安装配置 NFS 服务

## 安装依赖

首先安装 NFS 服务器需要的软件包

```bash
sudo apt update
sudo apt install nfs-kernel-server
```

在使用端安装 NFS 客户端需要的软件包

```bash
sudo apt update
sudo apt install nfs-common
```

<!-- more -->

## 设置导出目录

在服务端配置要暴露出的文件夹, 编辑 ```/etc/exports``` 文件, 添加下行的内容:

```
/storage 192.168.2.0/24(rw,sync,no_subtree_check,anonuid=33,anongid=33)
```

 其中 /storage 是要暴露的目录, 192.168.2.0/24 则是允许访问的客户端 IP 信息. anonuid=33 和 anongid=33 限制了 NFS 用户的权限与服务端 uid 为 33 的用户, gid 为 33 的组的权限相同.
 
 注意, 与 Samba 不同, NFS 默认不提供任何验证用户的方法, 客户端访问限制是通过 IP 地址和/或 hostname 实现的.

然后执行重新导出命令:

```bash
exportfs -arv
```

当想查看已经共享的详细信息时, 可以使用:

```bash
exportfs -v
```

# 配置 Kubernetes

## 定义储存文件

首先我们定义一个储存用的 PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  capacity:
    storage: 1024Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.2.120
    path: "/storage"
```

其中 ```192.168.2.120``` 即为 NFS 服务器所在的 IP 地址.

## 声明分配需求

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 1024Gi
```

## 验证结果

完成后, 我们可以启动两个测试容器写入一个文件检测一下是否工作正常

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/staging/volumes/nfs/nfs-busybox-rc.yaml
```

查看 NFS 服务器上导出的目录, 可以发现已经正确的写入了对应的文件.

# 参考资料

- [https://wiki.archlinux.org/index.php/NFS_(简体中文)](https://wiki.archlinux.org/index.php/NFS_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
- [https://github.com/kubernetes/examples/tree/master/staging/volumes/nfs](https://github.com/kubernetes/examples/tree/master/staging/volumes/nfs)