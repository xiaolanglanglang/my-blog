---
title: K3S 集群的 Ingress 配置
tags:
  - Kubernetes
  - ingress
  - 个人备忘
date: 2020-08-10 16:45:50
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
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: echo
spec:
  backend:
    serviceName: echo
    servicePort: http
```

> 这里没有指定 hostname , 所以它可以处理所有的入站请求.

一个最基本的 Ingress 服务就配置好了. 使用浏览器打开任一 k3s 节点的 IP 地址即可查看 echo server 的响应信息.