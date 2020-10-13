---
title: 升级 K3S 集群版本
tags:
  - 个人备忘
  - K3S
  - 升级
date: 2020-10-13 12:03:34
---


升级 K3S 集群的方式有很多, 这里选择了使用升级套件进行升级.

<!-- more -->

根据官方的文档, 使用升级套件升级 K3S 的版本非常简单, 首先安装升级用的套件:

```bash
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/download/v0.6.4/system-upgrade-controller.yaml
```

接着定义好 Plan 文件, Plan 文件指示了系统要升级到的版本.

plans.yaml:

```yaml
# Server plan
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: In
      values:
      - "true"
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.19.2+k3s1
---
# Agent plan
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: agent-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  tolerations:
  - key: "network"
    operator: "Equal"
    value: "direct"
    effect: "NoSchedule"
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: DoesNotExist
  prepare:
    args:
    - prepare
    - server-plan
    image: rancher/k3s-upgrade:v1.19.2-k3s1
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.19.2+k3s1
```

> 因为我的节点中有容忍度的配置, 所以对应的 agent 也要添加相关的容忍度设置.

然后应用它:

```bash
kubectl apply -f plans.yml
```

> 升级完成后可能会出现 kubeconfig 被修改的情况, 从 master 的 ```/etc/rancher/k3s/k3s.yaml``` 重新复制到本地目录即可.