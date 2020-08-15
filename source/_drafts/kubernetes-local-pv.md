---
title: Kubernetes 部署 MySQL 应用
tags: [Kubernetes, MySQL, 个人备忘]
---

我计划将数据库(MySQL)也使用 K8S 来进行部署~~(别问为什么, 可能就是闲的蛋疼吧)~~, 如果使用之前文章中介绍的 NFS 方式({% post_link kubernetes-config-nfs %})来提供持久化卷的话, 可能会有性能问题, 因为我不需要数据库有漂移到其他节点的能力, 所以这里选择使用 ```local``` 的方式.

# 创建 local 卷

首先创建好给数据库数据的文件夹.

```bash
mkdir -p /zfs-pool/mysql/data
```

接着使用 local 模式将这个硬盘挂入 kubernetes.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    app: mysql
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    # 数据库数据挂载的宿主目录.
    path: /zfs-pool/mysql/data
  # 使用亲和性指定 MySQL 的数据文件夹所属的节点.
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - master
```

> 当 PV 决定好在哪个节点时, pods 会根据亲和性同样被绑定在这个节点上.

# 创建 PersistentVolumeClaim

使用如下的配置文件创建一个 PersistentVolumeClaim:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: "local-storage"
  resources:
    requests:
      storage: 100Gi
  # 使用选择器选择特定的标签
  selector:
    matchLabels:
      app: mysql
```

使用命令可以确认两者已经被正确的绑定在一起:

```
kubectl get pv
kubectl get pvc
```

# 创建 MySQL 密码

首先我们创建一个包含有密码的文件:

```bash
echo -n MYSQL_ROOT_PASSWORD > mysql-root-password.txt
```

> 注意使用 -n 参数避免生成多余的换行.
> 默认键名就是文件名

然后使用命令将密码导入到 Kubernetes 中:

```bash
kubectl create secret generic mysql-root-password --from-file=password=./mysql-root-password.txt
```

> 这里使用```[--from-file=[key=]source]```参数重新设置了键名.

# 创建配置文件

我们可以使用 Kubernetes 提供的 ConfigMaps 功能来管理 MySQL 的配置文件.

首先创建一个 MySQL 的配置文件:

```
[mysqld]
skip_name_resolve
character-set-server=utf8mb4
collation-server=utf8mb4_general_ci

[client]
default-character-set=utf8mb4
```

然后使用指令将其导入到 Kubernetes 中:

```bash
kubectl create configmap mysql-config --from-file=/zfs-pool/mysql/mysql.cnf
```

# 创建 StatefulSet

因为 MySQL 是有状态的容器, 所以这里使用 StatefulSet 来创建, 而不是 Deployment.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0.21
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-root-password
                  key: password
          ports:
            - containerPort: 3306
          livenessProbe:
            exec:
              command: ["mysqladmin", "ping"]
          volumeMounts:
            - name: mysql-pv
              mountPath: /var/lib/mysql
            - name: mysql-conf
              mountPath: /etc/mysql/conf.d
      volumes:
        - name: mysql-pv
          persistentVolumeClaim:
            claimName: mysql-pvc
        - name: mysql-conf
          configMap:
            name: mysql-config
```

# 创建 Service

终于, 我们来到了最后一步! 创建一个 Service 将 MySQL 暴露出去即可.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  type: NodePort
  ports:
    - port: 3306
      protocol: TCP
  selector:
    app: mysql
```
