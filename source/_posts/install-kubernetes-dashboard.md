---
title: K3S 集群安装 Kubernetes Dashboard
tags:
  - Kubernetes
  - k3s
  - monitor
  - Kubernetes Dashboard
  - 个人备忘
date: 2020-08-06 18:44:08
---


### Kubernetes Dashboard

Kubernetes Dashboard 是 Kubernetes 团队开源的一个平台, 可以使用它查询最基本的 Kubernetes 运行信息.

#### 安装 Kubernetes Dashboard

这一步很简单, 在 Github 上的 dashboard 项目 [release](https://github.com/kubernetes/dashboard/releases) 上选择兼容的版本, 然后执行对应的安装命令即可, 例如:

```bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml
```

#### 创建人员权限

创建一个配置文件 ```dashboard-adminuser.yaml``` 来记录以下的代码段

首先创建一个账号

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

接着绑定对应的权限

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

应用上述的两个代码段

```bash
kubectl apply -f dashboard-adminuser.yaml
```

#### 使用 Kubernetes Dashboard

相关的账号就创建好了, 接着可以使用命令获取到对应的 Token 信息

```bash
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

使用命令 ```kubectl proxy``` 可以在本地启动对应的转发, 接着用浏览器打开链接 

[http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/) 

并输入上一步获取的 Token 信息即可访问 Kubernetes 的运行信息.