---
title: 使用 Helm 部署 Gitlab 家庭实例
tags:
  - Kubernetes
  - k3s
  - helm
  - gitlab
  - 个人备忘
date: 2022-07-03 19:30:00
---

使用 Helm 安装 Gitlab 时, 默认的配置会安装所有的依赖, 但是对于已有数据库等功能的场景来说, 这样做并不是很合适, 所以需要调整 values.yaml 文件让 Gitlab 使用已有的外部依赖.

<!-- more -->

# 准备工作

在这里我们假定已经提前安装好了 PostgreSQL 、 Redis 和 MinIO.

## 添加 helm chart

```bash
helm repo add gitlab https://charts.gitlab.io/
helm repo update
```

## 导出默认配置到文件

```bash
helm show values gitlab/gitlab > values.yaml
```

# 修改配置

使用自己熟悉的编辑器编辑这个文件.

## 修改主机信息

参考 [配置主机设置](https://docs.gitlab.com/charts/charts/globals#configure-host-settings) 将相关的配置修改为自己的主机信息.

```yaml
  hosts:
    domain: xiaolanglang.net
    hostSuffix:
    https: true
    externalIP:
    ssh: ~
    gitlab: {}
    minio: {}
    registry: {}
    tls: {}
    smartcard: {}
    kas: {}
    pages: {}
```

## 配置 Ingress 信息

计划使用 istio 的 IngressGateway 来暴露 Gitlab, 所以这里关闭 Gitlab 默认的 Ingress 配置.

```yaml
  ingress:
    apiVersion: ""
    configureCertmanager: false
    provider: nginx
    # class:
    annotations: {}
    enabled: false
    tls:
     #  enabled: true
      secretName: xiaolanglang-net-wildcard-certificate
    path: /
    pathType: Prefix
```

## 配置 PostgreSQL 信息

我的 PostgreSQL 实例是单独部署的, 所以这里直接使用部署的 PostgreSQL 的信息.
配置文件参考 [配置postgresql设置](https://docs.gitlab.com/charts/charts/globals#configure-postgresql-settings)

```yaml
  psql:
    connectTimeout:
    keepalives:
    keepalivesIdle:
    keepalivesInterval:
    keepalivesCount:
    tcpUserTimeout:
    password:
      useSecret: true
      secret: postgresql-gitlab
      key: postgres-password
      file:
    host: postgresql.default.svc.cluster.local
    port: 5432
    username: gitlab
    database: gitlabhq_production
    # applicationName:
    # preparedStatements: false
    # databaseTasks: true
```

## 配置 Redis 信息

参考 [配置redis设置](https://docs.gitlab.com/charts/charts/globals#configure-redis-settings) 对 Redis 进行配置

```yaml
  redis:
    password:
      enabled: true
      secret: redis-gitlab
      key: redis-password
    host: redis-master.default.svc.cluster.local
    port: 6379
    # sentinels:
    #   - host:
    #     port:
```

## 配置时区

修改时区到东八区

```yaml
  timezone: Asia/Shanghai
```

## 配置外部对象储存

参考[外部储存对象](https://docs.gitlab.com/charts/advanced/external-object-storage/index.html)配置对象储存

```yaml
    ## https://docs.gitlab.com/charts/charts/globals#lfs-artifacts-uploads-packages-external-mr-diffs-and-dependency-proxy
    object_store:
      enabled: true
      proxy_download: true
      storage_options: {}
        # server_side_encryption:
        # server_side_encryption_kms_key_id
      connection:
        secret: gitlab-rails-storage
        key: connection
```

我使用的是 MinIO, 所以 Secret 参考 S3 进行配置, Secret 的 key 对应一个 yaml 文件, 文件参考为:

```yaml
# Example configuration of `connection` secret for Rails
# Example for Minio storage
#   See https://gitlab.com/gitlab-org/charts/gitlab/blob/master/doc/charts/globals.md#connection
#   See https://gitlab.com/gitlab-org/charts/gitlab/blob/master/doc/advanced/external-object-storage
provider: AWS
# Specify the region
region: us-east-1
# Specify access/secret keys
aws_access_key_id: BOGUS_ACCESS_KEY
aws_secret_access_key: BOGUS_SECRET_KEY
# The below settings are for S3 compatible endpoints
#   See https://docs.gitlab.com/ee/administration/job_artifacts.html#s3-compatible-connection-settings
aws_signature_version: 4
host: storage.example.com
endpoint: "https://storage.example.com:9000"
path_style: true
```

## 关闭依赖的安装

### 关闭证书管理

因为已经配置过证书管理工具, 此处不需要安装证书管理工具.

```yaml
certmanager:
  installCRDs: false
  nameOverride: certmanager
  # Install cert-manager chart. Set to false if you already have cert-manager
  # installed or if you are not using cert-manager.
  install: false
  # Other cert-manager configurations from upstream
  # See https://github.com/jetstack/cert-manager/blob/master/deploy/charts/cert-manager/README#configuration
  rbac:
    create: false
```

### 关闭 Ingress 安装

```yaml
nginx-ingress:
  enabled: false
```

### 关闭 Prometheus 安装

```yaml
prometheus:
  install: false
```

### 关闭 Redis 安装

```yaml
redis:
  install: false
```

### 关闭 Postgresql 安装

```yaml
postgresql:
  install: false
```

# 安装 Gitlab

执行以下命令安装 Gitlab

```sh
helm install gitlab gitlab/gitlab -f values.yaml
```

# 配置 IngressGateway

因为我使用的是 Istio 的 IngressGateway, 所以无法直接使用 Helm 的 Ingress 生成.
参考配置如下:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: gitlab-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http-gitlab
      protocol: HTTP
    hosts:
      - gitlab.xiaolanglang.net
  - port:
      number: 443
      name: https-gitlab
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: xiaolanglang-net-wildcard-certificate # This should match the Certificate secretName
    hosts:
      - gitlab.xiaolanglang.net
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: gitlab-vs
spec:
  hosts:
    - gitlab.xiaolanglang.net
    - gitlab-webservice-default.gitlab.svc.cluster.local
  gateways:
  - gitlab-gateway
  - mesh
  http:
  - route:
    - destination:
        port:
          number: 8181
        host: gitlab-webservice-default.gitlab.svc.cluster.local
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: gitlab-dr
spec:
  host: gitlab-webservice-default.gitlab.svc.cluster.local

```