---
title: 家庭部署 Miniflux 和 RSSHub
tags:
  - Kubernetes
  - k3s
  - PostgreSQL
  - Miniflux
  - RSSHub
  - rss
  - 个人备忘
date: 2022-12-13 19:30:00
---


Miniflux 是一个自建的 RSS 阅读器, RSSHub 是一个开源、简单易用、易于扩展的 RSS 生成器，可以给任何奇奇怪怪的内容生成 RSS 订阅源。

<!-- more -->

# 安装配置数据库

miniflux 依赖 PostgreSQL 数据库, 所以需要先安装 PostgreSQL 数据库. 这里使用 helm chart 安装.

## 添加 helm chart

这里使用的是 bitnami 提供的 PostgreSQL chart.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

## 安装 PostgreSQL

将 bitnami 里的 PostgreSQL chart 配置文件下载到本地, 修改配置后安装.

```bash
helm show values bitnami/postgresql > values.yaml
helm install postgresql bitnami/postgresql -f values.yaml
```

## 配置数据库

我们需要给 Miniflux 创建一个数据库, 并为这个数据库创建一个用户, 授予这个用户对这个数据库的所有权限.

> 使用 helm 安装的 PostgresSQL 在执行命令前需要先执行 `. /opt/bitnami/scripts/libpostgresql.sh && postgresql_enable_nss_wrapper` 命令.

完成的命令如下:

```bash
. /opt/bitnami/scripts/libpostgresql.sh && postgresql_enable_nss_wrapper

# Create a database user for Miniflux
$ createuser -P miniflux
Enter password for new role: ******
Enter it again: ******

# Create a database for miniflux that belongs to our user
$ createdb -O miniflux miniflux

# Create the extension hstore as superuser
$ psql miniflux -c 'create extension hstore'
CREATE EXTENSION
```

此处可以参考 [Miniflux 官方文档](https://miniflux.app/docs/installation.html#database).

# 安装 Miniflux

Miniflux 没有提供 helm chart, 这里使用配置文件安装.

deploy.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: miniflux
  labels:
    app: miniflux
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: miniflux
  template:
    metadata:
      labels:
        app: miniflux
        version: v1
    spec:
      containers:
        - name: miniflux
          image: "miniflux/miniflux:2.0.41"
          env:
            - name: "DATABASE_URL"
              value: ""
            - name: "RUN_MIGRATIONS"
              value: "1"
            - name: "TZ"
              value: "Asia/Shanghai"
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
      nodeSelector:
        network: proxy          
      terminationGracePeriodSeconds: 60 
```

service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: miniflux
  labels:
    app: miniflux
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: http
    protocol: TCP
    name: http-miniflux
  selector:
    app: miniflux
```

ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: miniflux
spec:
  rules:
  - host: MINIFLUX_HOST
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: miniflux
            port:
              number: 8080
  tls:
  - hosts:
    - MINIFLUX_HOST
    secretName: MINIFLUX_HOST_SECRET
```

# 安装 RSSHub

RSSHub 同样没有提供 helm chart 但是提供了 docker 镜像, 这里使用配置文件安装.

deploy.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: rsshub
    version: v1
  name: rsshub
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rsshub
  template:
    metadata:
      labels:
        app: rsshub
        version: v1
    spec:
      containers:
      - name: rsshub
        image: diygod/rsshub:chromium-bundled-2022-10-31
        ports:
        - containerPort: 1200
        envFrom:
        - configMapRef:
            name: rsshub-config
      nodeSelector:
        network: proxy          
      terminationGracePeriodSeconds: 60 
```

service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: rsshub
  name: rsshub
spec:
  ports:
  - name: http-rsshub
    port: 1200
    protocol: TCP
    targetPort: 1200
  selector:
    app: rsshub
```

ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rsshub
spec:
  rules:
  - host: RSSHUB_HOST
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: rsshub
            port:
              number: 1200
  tls:
  - hosts:
    - RSSHUB_HOST
    secretName: RSSHUB_HOST_SECRET
```

RSSHub 需要在抓取某些网站时需要配置一些环境变量, 这里使用 configmap 来配置.

configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rsshub-config
  namespace: default
data:
  XXX: XXXX
```

至此, RSSHub 和 Miniflux 都已经安装完成, 可以配置订阅源开始阅读了.