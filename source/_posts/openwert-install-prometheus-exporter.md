---
title: Prometheus 添加 OpenWRT 监控
tags:
  - prometheus
  - openwrt
date: 2020-09-05 17:05:15
---


# 配置 OpenWRT

因为我的 OpenWRT 路由并没有 Wi-Fi, 所以只需要安装下面三个组件, 可以在后台使用 GUI 安装也可以使用 opkg 安装.

```bash
opkg install prometheus-node-exporter-lua
opkg install prometheus-node-exporter-lua-netstat
opkg install prometheus-node-exporter-lua-nat_traffic
```

<!-- more -->

OpenWRT 的 exporter 默认只监听在 loopback 里, 所以我们需要调整配置文件它监听在内网里.

修改 ```/etc/config/prometheus-node-exporter-lua``` 文件, 将 ```listen_interface``` 的值修改为 ```lan```:

```
config prometheus-node-exporter-lua 'main'
        option listen_interface 'lan'
        option listen_ipv6 '0'
        option listen_port '9100'
```

然后在启动项里重启 ```prometheus-node-exporter-lua``` 服务即可.

# 配置 Prometheus

因为我的 Prometheus 是使用 ```kube-prometheus``` 装在 Kubernetes 里的(见 {%post_link install-prometheus%}), 所以这里使用 Kubernetes 的 CRD 来定义监控点:


因为 OpenWRT 是一个外部应用, 所以我们需要定义一个外部端点来指向 OpenWRT 的端点:

openwrt-ep.yaml:

```yaml
apiVersion: v1
kind: Endpoints
metadata:
    name: openwrt
    labels:
        system: openwrt
subsets:
  - addresses:
    - ip: 192.168.2.1
    ports:
    - name: metrics
      port: 9100
      protocol: TCP
```

接着配置一个 service 来指向它的端点:

openwrt-svc.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
    name: openwrt
    labels:
        system: openwrt
spec:
    ports:
    - name: metrics
      port: 9100
      protocol: TCP
      targetPort: 9100
```

最后定义一下对应的监控点, 配置好监控频率即可:

openwrt-service-monitor.yaml:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
    name: openwrt-metrics
    labels:
        app: openwrt-metrics
        prometheus: kube-prometheus
spec:
    selector:
        matchLabels:
            system: openwrt
    namespaceSelector:
        matchNames:
        - default
    endpoints:
    - port: metrics
      interval: 10s
      honorLabels: true
```

在 Prometheus 的 target 中可以看到端点已经 UP 了.

{% asset_img prometheus-target.png %}

# 配置 Grafana

导入 [https://grafana.com/grafana/dashboards/11147](https://grafana.com/grafana/dashboards/11147) 即可.

最终效果:

{% asset_img openwrt-grafana.png %}

~~然而并没有什么卵用.~~