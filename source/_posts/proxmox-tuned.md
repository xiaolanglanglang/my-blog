---
title: 优化 Proxmox 待机功耗
date: 2023-03-03 14:41:17
tags: 
  - 个人备忘
  - Proxmox
  - Linux
  - KVM
---

近日在折腾了一圈 DPDK 发现在家用环境中没多少用回退到原来的网络配置，发现待机功耗相比于之前有了接近 10W 左右的提升，于是研究了一下如何降低 Linux 待机功耗。

<!-- more -->

# 调整 CPU 运行模式

在搜索资料时，从[这个帖子](https://forum.proxmox.com/threads/fix-always-high-cpu-frequency-in-proxmox-host.84270/)中发现 CPU 当前处于 ```performance``` 模式，而不是 ```powersave``` 模式，将其调整为 ```powersave``` 模式后，待机功耗明显降低。

```bash
echo "powersave" | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

# 降低待机 CPU 使用率

其实上一步做完，待机功耗就已经降低到之前的正常水平了，但是偶然间发现了其他人分享的 PVE 虚拟机的设置，可以进一步降低 CPU 的系统占用率。对于无 GUI 的虚拟机来说，可以将设置里的```启用平板指针```选项卡关闭，可以降低 CPU 的占用率。变相的也可以降低待机功耗。
从[这个帖子](https://forum.proxmox.com/threads/use-tablet-for-pointer-option-causing-cpu-usage-on-linux.54084/)来看，这个问题不久之后就会被修复，届时应该不会再是问题。

# 使用 Tuned 模块调整系统参数

在 RHEL 中内置了一个 tuned 工具，但 Ubuntu 和 Proxmox 默认并没有安装，需要手动安装。

```bash
apt install tuned
```

对于 Linux 的虚拟机来说，可以使用  ```virtual-guest``` 的配置文件来优化虚拟机的性能。

```bash
tuned-adm profile virtual-guest
```

而对于 Proxmox 的宿主机来说，因为我的目标是节约待机功耗，所以这里使用 ```server-powersave``` 的配置文件。

```bash
tuned-adm profile server-powersave
```

# 总结

在使用了上述方法后，在没有造成体感上的性能损失的情况下，待机功耗甚至低于了之前的正常水平。

{% asset_img grafana.png %}

~~其实并没有节省多少电费，毕竟降低后依然有接近 100W 的功耗~~