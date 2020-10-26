---
title: K3S 集群的 Ingress 配置
tags:
  - Kubernetes
  - ingress
  - 个人备忘
date: 2020-08-10 16:45:50
updated: 2020-10-26 17:21:46
---


我使用的是 K3S 搭建的 Kubernetes 集群, 它默认安装了一个 Ingress 工具 treafik , 这里直接使用 treafik 并使用 echo server 来举例说明 Ingress 的配置.

首先定义 echo server 的 deployment 信息.

<!-- more -->

echo-deploy.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: echo
  name: echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: k8s.gcr.io/echoserver:1.10
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
      nodeSelector:
        network: proxy
      terminationGracePeriodSeconds: 60
```

> 这里使用了亲和性要求该 pod 部署在使用特定的网络环境的节点中.

接着配置 echo server 的 service 信息, 因为我们要使用 ingress 提供流量入口, 所以 service 可以直接声明为使用 ClusterIP, 即默认类型.

echo-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: echo
  name: echo
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: echo
```

最后声明 Ingress 的配置.

echo-ingress.yml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
spec:
  defaultBackend:
    service:
      name: echo
      port:
        name: http
```

> 这里没有指定 hostname , 所以它可以处理所有的入站请求.

一个最基本的 Ingress 服务就配置好了. 使用浏览器打开任一 k3s 节点的 IP 地址即可查看 echo server 的响应信息.

有时我们需要获取客户的真实 IP 信息, 但此时我们会发现 echo 服务的真实 IP 为内部的 IP, 我们可以这样调整它:

编辑 ```/var/lib/rancher/k3s/server/manifests/traefik.yaml``` 文件, 在 ```valuesContent``` 中添加 ```externalTrafficPolicy: Local``` 即可.

例如:

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: traefik
  namespace: kube-system
spec:
  chart: https://%{KUBERNETES_API}%/static/charts/traefik-1.81.0.tgz
  valuesContent: |-
    rbac:
      enabled: true
    ssl:
      enabled: true
    metrics:
      prometheus:
        enabled: true
    kubernetes:
      ingressEndpoint:
        useDefaultPublishedService: true
    image: "rancher/library-traefik"
    tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
    externalTrafficPolicy: Local
```

externalTrafficPolicy 中 Local 和 Cluster (默认) 的区别:
Local 只会把请求转发到本机的 Pod 中, 所以能保留真实的源 IP, 当本机没有对应的 Pod 时会直接抛弃连接.
Cluster 可以在转发到其他的节点上运行的 Pod 中, 没有保留真实的源 IP, 当本机没有对应的 Pod 时会将连接转发到其他的节点中.