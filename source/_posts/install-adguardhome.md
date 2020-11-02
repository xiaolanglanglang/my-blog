---
title: 安装并配置 AdGuardHome
tags:
  - AdGuardHome
  - Kubernetes
  - DNS
  - 个人备忘
date: 2020-11-02 17:49:10
---


很多人推荐使用 AdGuardHome 来实现全网的广告拦截和反跟踪, 那么我也试试!

<!-- more -->

# 准备工作

老规矩先给它分配一个储存空间, 用来保存它的配置和数据信息.

对应的 PV 文件:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: adguardhome-pv
  labels:
    app: adguardhome
spec:
  capacity:
    storage: 500Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /zfs-pool/adguardhome
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - master
```

以及 PVC 文件内容:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: adguardhome-pvc
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
      app: adguardhome
```

# 部署 AdGuardHome

AdGuardHome 官方提供了 docker 镜像, 这事儿就简单多了. 直接使用官方镜像即可.

deployment 文件:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: adguardhome
  name: adguardhome
spec:
  replicas: 1
  selector:
    matchLabels:
      app: adguardhome
  template:
    metadata:
      labels:
        app: adguardhome
    spec:
      containers:
      - name: adguardhome
        image: adguard/adguardhome:v0.103.3
        ports:
        - containerPort: 53
        - containerPort: 67
        - containerPort: 68
        - containerPort: 80
        - containerPort: 443
        - containerPort: 853
        - containerPort: 3000
        volumeMounts:
        - name: adguardhome-pv
          mountPath: /opt/adguardhome/work
          subPath: work
        - name: adguardhome-pv
          mountPath: /opt/adguardhome/conf
          subPath: conf
      - name: tunnel
        image: docker.io/cloudflare/cloudflared:2020.10.0
        imagePullPolicy: Always
        command: ["cloudflared", "tunnel"]
        args:
        - --url=http://127.0.0.1:3000
        - --hostname=adguardhome.xiaolanglang.net
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
      - name: adguardhome-pv
        persistentVolumeClaim:
          claimName: adguardhome-pvc
      terminationGracePeriodSeconds: 60
```

AdGuardHome 就部署好了. 接下来把它暴露出来.

# 声明服务

AdGuardHome 的默认管理端端口为 3000, 我们维持默认端口即可.

adguardhome-svc.yaml:
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: adguardhome
  name: adguardhome
spec:
  ports:
  - name: http
    port: 3000
    protocol: TCP
    targetPort: 3000
  selector:
    app: adguardhome
```

因为 AdGuardHome 使用 DNS 来提供广告过滤的功能, 而 DNS 的默认端口为 53, 所以这里我们需要用 LoadBalancer 类型的服务来暴露它.

adguardhome-svc-udp.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: adguardhome-udp
  name: adguardhome-udp
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  ports:
  - name: dns-udp
    port: 53
    protocol: UDP
    targetPort: 53
  selector:
    app: adguardhome
```

# 配置 Ingress 入口

接下来, 使用 Ingress 来将控制台服务暴露出来即可.

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: adguardhome
spec:
  rules:
  - host: adguardhome.xiaolanglang.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          serviceName: adguardhome
          servicePort: http
  tls:
  - hosts:
    - adguardhome.xiaolanglang.net
    secretName: xiaolanglang-net-wildcard-certificate
```

# 提供 DoH 服务

AdGuardHome 实际上还可以提供 DOH 服务, 因为我们在 Ingress 层做了 SSL 卸载, 所以只需要在 AdGuardHome 的 http 端口提供服务即可.

AdGuardHome 目前并没有办法在控制台上将 DNS 查询服务暴露在 http 端口上, 我们需要修改它的配置文件.

在第一步配置的 conf 目录中打开 AdGuardHome.yaml 文件, 在 ```tls``` 里找到

```yaml
allow_unencrypted_doh: false
```

这一项, 将其改为

```yaml
allow_unencrypted_doh: true
```

即可.

# 提供服务

对于家庭网络来说, 直接将 DNS 服务器地址指向 LoadBalancer 服务的外部 IP 即可.
对于 DoH 服务来说, 地址为 Ingress 域名下的 dns-query 目录.