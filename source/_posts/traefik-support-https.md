---
title: Traefik 添加 HTTPS 支持
tags: [k3s, Kubernetes, traefik, https, cert-manager, cloudflare, 个人备忘]
date: 2020-08-17 11:22:26
updated: 2020-10-26 17:23:26
---

在之前的文章中({% post_link ingress-config %}), 我们配置了 echo 服务的 Ingress 入口, 当我们使用 http 进行访问时, 一切正常, 但当使用 https 访问时, 我们会得到一个证书错误的提示. 这篇文章记录了如何使用 cert-manager 来解决 https 提示证书错误的问题.

<!-- more -->

# Cloudflare

我选择了 [Let's Encrypt](https://letsencrypt.org/) 作为我的证书签发机构. 它的验证方式有两种, 一种是使用 http 请求去验证, 另一种是使用 DNS 记录的方式去验证. 因为我的网络环境不允许公网访问我的 80/443 端口, 所以这里只能使用 DNS 的方式进行验证. 我们首先需要生成一组 Token 来方便 cert-manager 来操作 DNS 记录.

## 生成 Token

注意生成 Token 时的配置如下即可:

权限:

- 区域 - DNS -编辑
- 区域 - 区域 - 读取

区域资源:

- 包括 - 所有区域

## 导入 Token

将 CloudFlare 生成的 token 使用 secret 的方式导入到 Kubernetes 中, 配置文件如下:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: <API Token>
```

# cert-manager

按照 [Github](https://github.com/jetstack/cert-manager) 上的描述, cert-manager 是一个 Kubernetes 插件, 用于自动管理和发放来自不同发证源的TLS证书. 它将确保证书的有效性和定期更新，并尝试在到期前的适当时间更新证书.

## 安装

在 Kubernetes 里安装 cert-manager 很简单, 只需要一行命令即可:

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager.yaml
```

## 配置

我们需要将 cert-manager 使用的签发机构的相关信息配置进 Kubernetes, 配置文件如下:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <EMAIL>
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - dns01:
        cloudflare:
          email: <EMAIL>
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
```

## 签发

现在我们来签发一张证书吧! 使用如下的文件来进行证书的签发:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: echo-xiaolanglang-net-certificate
  namespace: default
spec:
  secretName: echo-xiaolanglang-net-certificate
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
  commonName: echo.xiaolanglang.net
  dnsNames:
  - echo.xiaolanglang.net
```

除了签发单独的证书外,我们还可以签发一张 wildcard 类型的证书, 避免每次添加一个二级域名都需要签发一张证书的痛苦:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: xiaolanglang-net-wildcard-certificate
  namespace: default
spec:
  secretName: xiaolanglang-net-wildcard-certificate
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
  dnsNames:
  - xiaolanglang.net
  - '*.xiaolanglang.net'
```

然后, 我们可以使用命令:

```bash
kubectl get certificates
```

来查看证书的状态, 当其 Ready 字段为 True 时, 意味着证书已经签发成功.


# Traefik

## 使用已有的证书

最后, 我们修改 echo 的 Ingress 配置文件, 增加关于 tls 的字段, 完整的配置文件如下:

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
  tls:
  - hosts:
    - echo.xiaolanglang.net
    secretName: echo-xiaolanglang-net-certificate
```

使用 Chrome 打开 [https://echo.xiaolanglang.net](https://echo.xiaolanglang.net) 网页, 可以发现已经是小绿锁了.

> 当然了, 因为我的网络环境没有 80/443 端口, 所以实际上在公网中访问的是套了一层 Argo Tunnel 的(文章见 {%post_link kubernetes-config-argo-tunnel%}), 这里给 ingress 配置主要是方便做 DNS 劫持, 毕竟内网流量也要走一遍 CDN 也太蛋疼了.

------

2020-08-19 更新:

## 自动签发证书

除了使用已有的证书外, 我们还可以在 Ingress 里指定 issuer 来自动签发证书, 完整的配置文件如下:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt"
spec:
  defaultBackend:
    service:
      name: echo
      port:
        name: http
  tls:
  - hosts:
    - echo.xiaolanglang.net
    secretName: echo-xiaolanglang-net-certificate
```