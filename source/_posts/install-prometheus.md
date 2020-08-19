---
title: K3S 集群安装 Prometheus 监控
tags:
  - Kubernetes
  - k3s
  - prometheus
  - monitor
  - 个人备忘
date: 2020-08-09 18:20:12
updated: 2020-08-19 13:34:10
---

# 安装 Kube-Prometheus

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

<!-- more -->

# 验证安装结果

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

-------

2020-08-19 更新:

经过一段时间的使用, 我发现每次都这样访问的话, 会非常麻烦, 所以我决定将 grafana 暴露到公网中.

# 申请证书

按照 {%post_link traefik-support-https%} 里的方法申请一个证书, 配置文件如下:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: grafana-xiaolanglang-net-certificate
  namespace: monitoring
spec:
  secretName: grafana-xiaolanglang-net-certificate
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
  commonName: grafana.xiaolanglang.net
  dnsNames:
  - grafana.xiaolanglang.net
```

> 注意: 因为 grafana 在 monitoring 的命名空间中, 所以证书也应当存储在 monitoring 里.

# 配置 Ingress

依旧是按照上述的文章操作即可, 这里只记录一下配置文件:

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
spec:
  rules:
  - host: grafana.xiaolanglang.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          serviceName: grafana
          servicePort: 3000
  tls:
  - hosts:
    - grafana.xiaolanglang.net
    secretName: grafana-xiaolanglang-net-certificate
```

> 注意: 同样需要保证 Ingress 也在 monitoring 下

# 使用域名访问

添加好对应的 DNS 信息后, 直接使用 [grafana.xiaolanglang.net](https://grafana.xiaolanglang.net) 就可以访问到 grafana 界面了, 无需再设置端口转发. 使用起来方便多了.

-------

2020-08-19 再次更新:

经过查阅后发现, 实际上并不需要每次都手动申请证书, 只需要在 Ingress 的 annotations 里声明 issuer 就可以自动签发证书. 调整后的 yaml 如下:

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    # 添加关于 issuer 的信息
    cert-manager.io/cluster-issuer: "letsencrypt"
spec:
  rules:
  - host: grafana.xiaolanglang.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          serviceName: grafana
          servicePort: 3000
  tls:
  - hosts:
    - grafana.xiaolanglang.net
    secretName: grafana-xiaolanglang-net-certificate
```
