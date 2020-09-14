---
title: Kubernetes 安装 Plex Server
tags:
  - Kubernetes
  - Plex
  - 个人备忘
date: 2020-09-14 22:02:17
---


Plex 是一款集储存、索引、转码和在线播放的媒体库软件, 使用它可以在家庭内部搭建自己的媒体库. 官方有提供 plex server 的 docker 镜像, 这里可以直接使用它将 plex 安装到我的 Kubernetes 中.

<!-- more -->

# 安装 Plex
## 定义储存目录

首先定义一个储存媒体的地址, 这里储存在 master 的 ```/zfs-pool/media``` 目录下:

media-pv.yaml:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: media-pv
  labels:
    external: media
spec:
  capacity:
    storage: 500Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany # 此处使用 ReadWriteMany 是为了可以与 nextcloud 联动
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /zfs-pool/media
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - master
```

media-pvc.yaml:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: media-pvc
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
      external: media
```

> 将此 pvc 挂载到 nextcloud 中即可使用 nextcloud 管理 plex 的媒体库文件.

接着给 plex 自身定义储存空间 (转码目录、配置目录等), 这里配置到 ```/zfs-pool/plex``` 下:

plex-pv.yaml:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: plex-pv
  labels:
    app: plex
spec:
  capacity:
    storage: 500Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /zfs-pool/plex
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - master
```

plex-pvc.yaml:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: plex-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: "local-storage"
  resources:
    requests:
      storage: 500Gi
  # 使用选择器选择特定的标签
  selector:
    matchLabels:
      app: plex
```

## 定义 Deployment

只是有点长, 但是也没有很难:

plex-deploy.yaml:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app
  name: plex
spec:
  replicas: 1
  selector:
    matchLabels:
      app: plex
  template:
    metadata:
      labels:
        app: plex
    spec:
      containers:
      - name: plex
        image: plexinc/pms-docker:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 32400
        - containerPort: 3005
        - containerPort: 8324
        - containerPort: 32469
        - containerPort: 1900
        - containerPort: 32410
        - containerPort: 32412
        - containerPort: 32413
        - containerPort: 32414
        volumeMounts:
        - name: plex-pv
          mountPath: /config
          subPath: config
        - name: plex-pv
          mountPath: /transcode
          subPath: temp
        - name: media-pv
          mountPath: /mnt/media/
      - name: tunnel
        image: docker.io/cloudflare/cloudflared:2020.8.0
        imagePullPolicy: Always
        command: ["cloudflared", "tunnel"]
        args:
        - --url=http://127.0.0.1:32400
        - --hostname=media.xiaolanglang.net
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
      terminationGracePeriodSeconds: 60
      volumes:
      - name: tunnel-secret
        secret:
          secretName: xiaolanglang.net
      - name: plex-pv
        persistentVolumeClaim:
          claimName: plex-pvc
      - name: media-pv
        persistentVolumeClaim:
          claimName: media-pvc
      terminationGracePeriodSeconds: 60
```

## 定义 Service

然后就是将它暴露出服务了, 虽然上面定义了很多的端口, 但是根据[这篇文章](https://support.plex.tv/articles/201543147-what-network-ports-do-i-need-to-allow-through-my-firewall/)里的说明, 以我的需求来说只需要暴露 32400 端口即可.

plex-svc.yaml:
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: plex
  name: plex
spec:
  ports:
  - name: http
    port: 32400
    protocol: TCP
    targetPort: 32400
  selector:
    app: plex
```

## 定义 Ingress

将上面的 service 暴露出去就好啦!

plex-ingress.yaml:
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress
spec:
  rules:
  - host: media.xiaolanglang.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          serviceName: plex
          servicePort: http
  tls:
  - hosts:
    - media.xiaolanglang.net
    secretName: xiaolanglang-net-wildcard-certificate
```

# 配置 Plex

## 安装插件

因为我们使用的刮削器(Scraper)会使用 XBMC 的格式储存元信息, 所以要使用插件 [XBMCnfoMoviesImporter](https://github.com/gboudreau/XBMCnfoMoviesImporter.bundle) 让 plex 支持 XBMC 格式的元信息.

从 [GitHub](https://github.com/gboudreau/XBMCnfoMoviesImporter.bundle/archive/master.zip) 上下载 zip 格式的压缩包, 解压, 并重命名为 ```XBMCnfoMoviesImporter.bundle```, 然后将其拷贝到 ```Plug-ins``` 文件夹里即可.

> 此处的完整目录为 /zfs-pool/plex/config/Library/Application Support/Plex Media Server/Plug-ins

## 初始化 Plex

第一次初始化 Plex 服务器只能通过 localhost 进行, 这里使用端口转发来进行, 使用命令:

```bash
kubectl port-forward svc/plex 32400
```

然后打开浏览器, 访问 [http://localhost:32400](http://localhost:32400) 即可进行配置.

## 配置网络

在 settings - Network - Enable Advanced Settings - Custom server access URLs 里添加上述的地址, 方便其他设备找到我们的服务器.