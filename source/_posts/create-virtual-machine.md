---
title: 创建 Ubuntu 虚拟机
tags: 个人备忘
date: 2020-08-03 10:24:30
---


创建一个 Ubuntu 虚拟机, 使用 openvswitch 的方式使虚拟机拥有与宿主机同级的 IP 地址, 方便后续在路由器上针对此虚拟机做路由策略. 

配置暂定为 1c2g, 安装盘镜像为 ```/zfs-pool/kvm/image/ubuntu-20.04-live-server-amd64.iso```, 系统盘存放在 ```/mnt/sdf1/kvm/disk/``` 目录下, sdf1 为一块 SSD 硬盘, 使用 openvswitch 管理网络.

首先创建一个系统盘, 并调整文件权限, 系统盘容量暂定为 40G

```bash
qemu-img create -f qcow2 ubuntu.qcow2 40G
chown libvirt-qemu:kvm ubuntu.qcow2
```

然后使用命令行创建一个虚拟机

```
virt-install --name ubuntu \
--vcpus=1 \
--ram 2048 \
--cdrom=/zfs-pool/kvm/image/ubuntu-20.04-live-server-amd64.iso \
--disk=/mnt/sdf1/kvm/disk/ubuntu.qcow2 \
--os-variant=ubuntu20.04 \
--boot machine=q35 \
--graphics vnc,listen=0.0.0.0
```
然后使用 vnc 链接进虚拟机, 一路 Enter 安装完成. 接着使用命令 ```virsh edit ubuntu``` 进入配置文件调整配置.

```xml
<!-- 调整网络配置 -->
<!-- 从 network 调整为 bridge -->
<interface type='bridge'>
  <!-- 从 network 调整为 bridge, 从 default 调整为 osvbr0 -->
  <source bridge='ovsbr0'/>
  <!-- 新增一行 -->
  <virtualport type='openvswitch'/>
</interface>
```

虚拟机就创建好了.