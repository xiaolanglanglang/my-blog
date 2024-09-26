---
title: 重新配置 HomeLab
date: 2024-09-26 11:19:39
tags:
  - HomeLab
  - 个人备忘
  - Kubernetes
  - k3s
---

家里的这台服务器已经运行了一段时间，在使用的过程中暴露出了不少的问题，所以我决定重新配置一下这台设备，以便更好地使用它。

<!-- more -->

# 备份数据

第一步备份数据，保证后续重装后业务可以正常运行。
将 TrueNas 里的阵列离线，在页面中操作将其 export，后续将在宿主机上直接挂载。这样阵列里的数据就可以继续使用。
为了防止后续无法挂载阵列，建议将阵列中的重要数据备份到其他地方。

# 硬件变化

之前使用 Proxmox VE 作为宿主机的操作系统，又虚拟化了一个 TrueNas 作为 NAS 服务器，为了让 TrueNas 能够直接管理硬盘，我使用了一张 HBA 卡，将硬盘接入 HBA 卡后直通给 TrueNas 管理，这样带来了两个问题

1. 风扇在宿主机管理，而硬盘在 TrueNas 管理，导致无法根据硬盘温度调节风扇转速。
2. HBA 卡会额外消耗一定的功耗。

新架构将硬盘直接接入主板的 SATA 接口，移除了 HBA 卡，降低了大约 20W 的功耗。

# 宿主机系统

宿主机我选择了 Ubuntu 24.04 LTS。使用安装工具安装即可。

## 挂载硬盘

安装 Ubuntu 后，需要将 ZFS 阵列挂载到系统中，首先安装 ZFS:

```bash
sudo apt install zfsutils-linux
```

然后将阵列挂载到系统中:

首先搜索一下可以导入的阵列:

```bash
sudo zpool import
```

此时应当可以看到之前的阵列，然后导入:

```bash
sudo zpool import -f <pool_name>
```

## 配置网络

因为之后的虚拟机需要一个 bridge 网络，所以需要先配置一下网络，对于 Ubuntu 来说，这个工作是 Netplan 完成的。
首先需要安装 bridge-utils:

```bash
sudo apt install bridge-utils
```

然后编辑配置文件:

```yaml
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
  ethernets:
    enp4s0:
      dhcp4: false
  bridges:
    br0:
      interfaces: [enp4s0]
      dhcp4: true
      macaddress: "XX:XX:XX:XX:XX:XX"
  version: 2
```

> 这里的 `macaddress` 是我之前使用的 MAC 地址，为了保持一致，所以指定了 MAC 地址。

最后应用一下配置:

```bash
sudo netplan apply
```

# 容器平台

容器平台我选择了 K3S 作为 Kubernetes 集群管理工具，相比于 K8S，K3S 更加轻量，更适合家庭使用<del>(Kubernetes 真的适合家用吗？)</del> 。

## 安装 K3S

K3S 的安装跟之前的差不太多，某些组件的选择与上次有所不同。安装命令:

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -s - server --disable=traefik --flannel-backend=none --disable-network-policy --cluster-cidr=10.0.0.0/16,fd42::/64 --service-cidr=10.1.0.0/16,fd42:00:01::/112 --default-local-storage-path=/mnt/sda1/volumes --kube-controller-manager-arg=bind-address=0.0.0.0 --kube-proxy-arg=metrics-bind-address=0.0.0.0 --kube-scheduler-arg=bind-address=0.0.0.0
```

因为计划使用 calico 和 ingress-nginx，所以关闭了 traefik 和 flannel，另外，因为默认监听 127.0.0.1 会导致 prometheus 无法采集到数据，所以将一些组件的监听地址改为 0.0.0.0.

## 安装 Calico

在安装时最新的 Calico 版本是 3.28.2，安装命令如下:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml
```

安装好后，需要配置 `custom-resources.yaml`。

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
      - blockSize: 26
        cidr: 10.0.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
      - blockSize: 122
        cidr: fd42::/64
        encapsulation: None
        natOutgoing: Enabled
        nodeSelector: all()
```

编辑好后应用一下即可:

```bash
kubectl apply -f custom-resources.yaml
```

## 安装 Multus

在之前的架构中，我选择了使用两个虚拟机来分别运行一般服务和下载服务，并根据两个虚拟机的不同 IP 来做流量控制，在新的架构中，我选择使用 Multus 来实现下载服务分配独占 IP 来进行流量控制。

```bash
helm repo add rke2-charts https://rke2-charts.rancher.io
helm repo update
helm install multus rke2-charts/rke2-multus -n kube-system --values multus-values.yaml
```

其中 `multus-values.yaml` 的内容如下:

```yaml
manifests:
  dhcpDaemonSet: true
```

## 安装 Ingress-Nginx

Ingress-Nginx 用于将外部流量导入到集群内部的服务中，安装命令如下:

```bash
helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --create-namespace --namespace ingress-nginx -f values.yaml
```

其中 `values.yaml` 的内容如下:

```yaml
controller:
  # NextCloud 需要使用
  allowSnippetAnnotations: true
  ingressClassResource:
    default: true
  service:
    # 为了保留原始 IP
    externalTrafficPolicy: "Local"
    # 为了支持 IPv6
    ipFamilyPolicy: RequireDualStack
    ipFamilies:
      - IPv4
      - IPv6
```

# 虚拟化平台

这次的虚拟化平台我选择了 KubeVirt 来实现，KubeVirt 是一个 Kubernetes 上的虚拟化平台，可以将虚拟机作为一个 Pod 运行在 Kubernetes 集群中。

## 安装 KubeVirt

### 安装 containerized-data-importer

```bash
export VERSION=$(basename $(curl -s -w %{redirect_url} https://github.com/kubevirt/containerized-data-importer/releases/latest))
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-operator.yaml
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-cr.yaml
```

### 安装 KubeVirt

```bash
export VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml
kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/v1.3.1/kubevirt-cr.yaml
```

### 安装 virtctl

```bash
VERSION=$(kubectl get kubevirt.kubevirt.io/kubevirt -n kubevirt -o=jsonpath="{.status.observedKubeVirtVersion}")
ARCH=$(uname -s | tr A-Z a-z)-$(uname -m | sed 's/x86_64/amd64/') || windows-amd64.exe
curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-${ARCH}
chmod +x virtctl
sudo install virtctl /usr/local/bin
```

# 调整内核参数

目前调整的内核参数如下:

```conf
fs.inotify.max_user_instances=256
net.ipv6.conf.all.use_tempaddr=0
net.ipv6.conf.default.use_tempaddr=0
```

至此，安装工作基本完成，接下来就是将之前的服务迁移到新的环境中。
