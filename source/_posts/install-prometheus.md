---
title: K3S 集群安装 Prometheus 监控
tags:
  - Kubernetes
  - k3s
  - prometheus
  - monitor
  - 个人备忘
date: 2020-08-09 18:20:12
---


安装 prometheus 的方法有很多, 这里选择直接安装 [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus), 先根据兼容性列表确认可以安装的 kube-prometheus 版本, 然后下载兼容的版本.

```bash
wget https://github.com/prometheus-operator/kube-prometheus/archive/v0.6.0.tar.gz
```

然后解压进入目录

```bash
tar zxvf v0.6.0.tar.gz
cd kube-prometheus-0.6.0
```

首先安装 setup 目录内的配置.

```bash
kubectl apply -f manifests/setup/
```

安装好后再应用外层目录下的配置.

```bash
kubectl apply -f manifests/
```

使用命令 

```bash
kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090
``` 

后访问 [http://localhost:9090](http://localhost:9090) 即可打开 prometheus 的页面.

使用命令 

```bash
kubectl --namespace monitoring port-forward svc/grafana 3000
``` 

后访问 [http://localhost:3000](http://localhost:3000) 即可打开 grafana 的页面

> 注: grafana 默认的用户名和密码均为 ```admin```, 首次进入会要求修改默认密码.

