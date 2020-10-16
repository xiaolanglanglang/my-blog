---
title: Kubernetes 安装 Nextcloud
tags:
  - Kubernetes
  - Nextcloud
  - 个人备忘
date: 2020-08-28 12:12:44
updated: 2020-10-16 14:13:00
---


Nextcloud 是一个很热门的可以个人部署的网盘服务器, 它的客户端支持了我使用的所有平台方便我同步文件, 还可以让我远程维护下载文件夹. 下面记录一下我将 Nextcloud 安装到 Kubernetes 的过程.

<!-- more -->

# 部署前的工作

首先当然是创建文件的存储目录了:

```bash
mkdir -p /zfs-pool/nextcloud
```

接着就是创建对应的 PV 和 PVC 文件, 配置如下:

nextcloud-pv.yaml:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nextcloud-pv
  labels:
    app: nextcloud
spec:
  capacity:
    storage: 500Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /zfs-pool/nextcloud
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - master
```

nextcloud-pvc.yaml:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: "local-storage"
  resources:
    requests:
      storage: 500Gi
  # 使用选择器选择特定的标签
  selector:
    matchLabels:
      app: nextcloud
```

> 如果要挂载其他的目录, 也是一样的操作

接着我们需要给 Nextcloud 创建一个数据库. 登录数据库后, 首先创建一个仓库:

```sql
create database nextcloud;
```

然后创建一个 nextcloud 使用的 mysql 用户:

```sql
CREATE USER 'nextcloud'@'%' IDENTIFIED BY 'PASSWORD';
```

并赋予它使用 nextcloud 数据库的权限:

```sql
grant ALL on nextcloud.* to 'nextcloud'@'%';
```

# 部署 Nextcloud

我使用 Nextcloud 官方打包的 docker 镜像 ([地址](https://hub.docker.com/_/nextcloud)) 进行封装, 配置文件如下:

nextcloud-deploy.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nextcloud
  name: nextcloud
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nextcloud
  template:
    metadata:
      labels:
        app: nextcloud
    spec:
      containers:
      - name: nextcloud
        image: nextcloud:20
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nextcloud-pv
          mountPath: /var/www/html/
      - name: nextcloud-cron
        image: nextcloud:20
        args:
        - /cron.sh
        volumeMounts:
        - name: nextcloud-pv
          mountPath: /var/www/html/
      - name: tunnel
        image: docker.io/cloudflare/cloudflared:2020.10.0
        command: ["cloudflared", "tunnel"]
        args:
        - --url=http://127.0.0.1:80
        - --hostname=drive.xiaolanglang.net
        - --origincert=/etc/cloudflared/cert.pem
        - --no-autoupdate
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
        volumeMounts:
        - mountPath: /etc/cloudflared
          name: tunnel-secret
          readOnly: true
      volumes:
      - name: tunnel-secret
        secret:
          secretName: xiaolanglang.net
      - name: nextcloud-pv
        persistentVolumeClaim:
          claimName: nextcloud-pvc
      terminationGracePeriodSeconds: 60
```

配置文件中使用了三个容器, 一个是 Nextcloud ,一个是 Nextcloud 的定时任务, 最后一个是 cloudflare 的镜像, 用来提供 argo tunnel 的支撑, 详情见文章 {%post_link kubernetes-config-argo-tunnel%}.

nextcloud-svc.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nextcloud
  name: nextcloud
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nextcloud
```

nextcloud-ingress.yaml:

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nextcloud
spec:
  rules:
  - host: drive.xiaolanglang.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          serviceName: nextcloud
          servicePort: http
  tls:
  - hosts:
    - drive.xiaolanglang.net
    secretName: xiaolanglang-net-wildcard-certificate
```

其实如果使用了 argo tunnel, 那么 service 和 ingress 都不需要再配置, 这里配置它们是方便之后在内网将域名直接解析到内网 IP 上, 避免内网访问还要过一遍 CDN.

# 部署后的工作

因为我们的 SSL 相关配置是配置到容器之外的, 并且 https 链接在容器外就已卸载, 所以需要额外的配置来告知容器我们使用了 https 链接.

进入我们映射的目录中我们可以看到不少容器生成的文件.

首先我们打开 config.php 文件, 找到并调整如下字段:

```
# 将 http 修改为 https
'overwrite.cli.url' => 'https://drive.xiaolanglang.net',
# 新增这一行, 强制使用 https 协议
'overwriteprotocol' => 'https',
```

同时我们需要告知容器外部的反向代理是可以信任的, 在 config.php 新增如下信息:

```
  'trusted_proxies' =>
  array (
    # 这是 Kubernetes 的 IP 段, 可能可以更精确的配置……
    0 => '10.0.0.0/8',
  ),
```

在 Nextcloud 的安全检查时汇报了关于 HSTS 的问题, 编辑 .htaccess 文件, 在 ```<IfModule mod_env.c>``` 里, 添加以下两行:

```apache
    Header onsuccess unset Strict-Transport-Security
    Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
```

修复提示未正确解析 carddav 和 caldav 的错误, 编辑 .htaccess 文件, 将原有的

```apache
  RewriteRule ^\.well-known/carddav /remote.php/dav/ [R=301,L]
  RewriteRule ^\.well-known/caldav /remote.php/dav/ [R=301,L]
```

调整为:

```apache
  RewriteRule ^\.well-known/carddav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
  RewriteRule ^\.well-known/caldav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
```

接着重启一下容器, Nextcloud 就完成了所有的部署了.