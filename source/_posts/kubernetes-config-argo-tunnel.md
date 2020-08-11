---
title: Kubernetes 集群配置使用 Argo Tunnel
date: 2020-08-10 17:54:05
tags:
  - Kubernetes
  - argo tunnel
  - cloudflare
  - sidecar
  - 个人备忘
---

按照 Cloudflare 文章的说法, Argo Tunnel 的 Ingress Controller 在 2019 年年底已经停止维护, 建议用户使用 Sidecar 的方式使用 Argo Tunnel, 所以这里使用 Sidecar 的方式进行部署.

<!-- more -->

# 配置 Cert 文件

初次使用时, 首先需要将 Cloudflare 的证书文件转换为 Kubernetes Secret.

## 生成 Cert 文件

在[下载页面](https://developers.cloudflare.com/argo-tunnel/downloads)下载 ```cloudflared``` 文件, 或在 Linux 系统中使用命令行直接安装

```bash
curl https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-darwin-amd64.tgz | tar xzC /usr/local/bin
```

Mac 系统也可使用 brew 安装

```bash
brew install cloudflare/cloudflare/cloudflared
```

接着使用登录指令绑定账号

```bash
cloudflared tunnel login
```

按照提示进行, 完成后证书将会被保存在 ```~/.cloudflared/cert.pem``` 文件中.

## 转换 Cert 文件为 Secret

例如我们将绑定域名为 ```xiaolanglang.net```, 则命令为:

```bash
kubectl create secret generic xiaolanglang.net --from-file="$HOME/.cloudflared/cert.pem"
```

# 配置 Deployment 文件

我们将之前的配置文件修改如下:

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
      - name: tunnel
        image: docker.io/cloudflare/cloudflared:2020.8.0
        imagePullPolicy: Always
        command: ["cloudflared", "tunnel"]
        args:
        - --url=http://127.0.0.1:8080
        - --hostname=echo.xiaolanglang.net
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
      nodeSelector:
        network: proxy
      terminationGracePeriodSeconds: 60 
```

# 验证结果

当节点启动后, 使用浏览器访问 [echo.xiaolanglang.net](https://echo.xiaolanglang.net) 即可获得响应信息.