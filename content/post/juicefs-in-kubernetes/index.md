---
title: "kubernetes存储实战: 使用JuiceFS"
date: 2024-01-11T21:26:00+08:00
tags: [kubernetes,storage,juicefs]
toc: true
---

## 目标

在Kubernetes的使用过程中，有两个主题是无法回避的：网络与存储。经过早期版本的各种尝试，目前Kubernetes 提供了 CNI 与 CSI 以开放接口的形式对接任何第三方厂商开发的网络插件和存储插件。

本次实践的目标是搭建一个 JuiceFS 文件系统，并在 Kubernetes 集群中以动态配置形式使用 JuiceFS 持久数据卷。

## 准备环境

本次实践直接使用上一次实践搭建的高可用Kubernetes集群。

## 安装JuiceFS

Juicefs 的设计理念很前沿（应该说是我很久没关注过存储这个主题了）。大致上JuiceFS分为了客户端、Meta存储、数据存储。

* 客户端用于文件系统相关操作
* Meta存储用于保存文件系统和文件的元信息
* 数据存储按照 Chunk、Slice、Block的粒度存储文件数据

具体可以参考官方[技术架构说明](https://juicefs.com/docs/zh/community/architecture)

### 安装Redis

Redis的安装比较简单，这里不再赘述。

### 准备对象存储

对象存储（Object Storage）服务有很多种可选，云上的 AWS S3、GCS、Azure Blob、阿里云OSS等，也可以使用自己搭建的MinIO、Ceph存储。为了降低操作复杂度，这里直接使用阿里云OSS，特别是如果是第一次开通OSS服务，可以[申请免费试用](https://free.aliyun.com)，真香！

{{< figureCupper
img="figure1-free-aliyun-oss.png"
caption="阿里云OSS免费试用"
command="Resize"
options="1080x" >}}

完成申请后，阿里云会给你的账号发放一个试用的资源包。之后在OSS控制台上按照使用说明，创建好Bucket，创建RAM用户账号，并对账号进行OSS相关操作权限进行授权。

过程中注意要记录好账号的 **AccessKey ID** 与 **AccessKey Secret**，在后续操作中要使用。

如果对对象存储服务不熟悉，在继续后续的操作之前，可以尝试通过Open API和SDK操作一下OSS服务，熟悉一下对象存储服务。

### 本地安装JuiceFS

参考官方文档

* [安装JuiceFS客户端](https://juicefs.com/docs/zh/community/getting-started/installation)
* [分布式模式](https://juicefs.com/docs/zh/community/getting-started/for_distributed)

在一台云服务器上下载JuiceFS客户端

```
JFS_LATEST_TAG=$(curl -s https://api.github.com/repos/juicedata/juicefs/releases/latest | grep 'tag_name' | cut -d '"' -f 4 | tr -d 'v')
wget "https://github.com/juicedata/juicefs/releases/download/v${JFS_LATEST_TAG}/juicefs-${JFS_LATEST_TAG}-linux-amd64.tar.gz
tar -zxvf juicefs-1.1.1-linux-amd64.tar.gz
install juicefs /usr/local/bin/ # 只有这一个二进制文件
```

创建文件系统

```
# 指定: oss 存储相关参数 redis meta存储相关参数 文件系统名称myjfs
juicefs format --storage oss --bucket https://<bucket name>.oss-cn-beijing-internal.aliyuncs.com --access-key <AccessKey ID> --secret-key <AccessKey Secret> redis://:jfsmeta@172.xx.xx.132:6379/1 myjfs

# 输出如下
2024/01/11 16:16:26.889928 juicefs[1954663] <INFO>: Meta address: redis://:****@172.xx.xx.132:6379/1 [interface.go:497]
2024/01/11 16:16:26.891907 juicefs[1954663] <WARNING>: AOF is not enabled, you may lose data if Redis is not shutdown properly. [info.go:84]
2024/01/11 16:16:26.892217 juicefs[1954663] <INFO>: Ping redis latency: 241.928µs [redis.go:3593]
2024/01/11 16:16:26.892669 juicefs[1954663] <INFO>: Data use oss://<bucket name>/myjfs/ [format.go:471]
2024/01/11 16:16:27.054413 juicefs[1954663] <INFO>: Volume is formatted as {
  "Name": "myjfs",
  "UUID": "************",
  "Storage": "oss",
  "Bucket": "https://<bucket name>.oss-cn-beijing-internal.aliyuncs.com",
  "AccessKey": "<AccessKey ID>",
  "SecretKey": "removed",
  "BlockSize": 4096,
  "Compression": "none",
  "EncryptAlgo": "aes256gcm-rsa",
  "KeyEncrypted": true,
  "TrashDays": 1,
  "MetaVersion": 1,
  "MinClientVersion": "1.1.0-A",
  "DirStats": true
} [format.go:508]

```

本地挂载文件系统

```
# 将myjfs文件系统挂载到本地 /root/jfs 目录上
juicefs mount --background --cache-dir /jfscache --cache-size 10240 redis://:jfsmeta@172.xx.xx.132:6379/1 /root/jfs

# 输出
2024/01/11 16:21:47.625038 juicefs[1956146] <INFO>: Meta address: redis://:****@172.xx.xx.132:6379/1 [interface.go:497]
2024/01/11 16:21:47.626856 juicefs[1956146] <WARNING>: AOF is not enabled, you may lose data if Redis is not shutdown properly. [info.go:84]
2024/01/11 16:21:47.627233 juicefs[1956146] <INFO>: Ping redis latency: 321.062µs [redis.go:3593]
2024/01/11 16:21:47.628078 juicefs[1956146] <INFO>: Data use oss://juicefsoss/myjfs/ [mount.go:605]
2024/01/11 16:21:47.628679 juicefs[1956146] <INFO>: Disk cache (/jfscache/cfb8ce85-9a0d-4702-b87d-b07804dcba1c/): capacity (10240 MB), free ratio (10%), max pending pages (15) [disk_cache.go:114]
2024/01/11 16:21:48.130527 juicefs[1956146] <INFO>: OK, myjfs is ready at /root/jfs [mount_unix.go:48]
```

这里因为不是生产环境，没有做开机自动挂载配置，也没有调整太多参数。

{{< figureCupper
img="figure2-jfs-mount.png"
caption="juicesf mount"
command="Resize"
options="1080x" >}}

参考官方文档，跑了一下bench。

```
juicefs bench /root/jfs
```

{{< figureCupper
img="figure3-jfs-bench.png"
caption="juicesf bench"
command="Resize"
options="1080x" >}}

从图中可以看到JuiceFS的一些性能参数。

## kubernetes配置CSI

参考官方文档

* [Kubernetes CSI](https://v1-28.docs.kubernetes.io/zh-cn/docs/concepts/storage/volumes/#csi)
* [JuicsFS Kubernetes CSI安装](https://juicefs.com/docs/zh/csi/getting_started)
* [创建和使用PV](https://juicefs.com/docs/zh/csi/guide/pv)

安装JuicsFS CSI

可以通过Helm安装，可以通过kubectl指定文件安装，由于网络问题，这次操作是下载了对应的配置文件后通过kubectl安装的。

```
# 在 Kubernetes 集群中任意一个非 Master 节点上执行以下命令
ps -ef | grep kubelet | grep root-dir

#如果上一步检查命令返回的结果不为空或者 /var/lib/kubelet，则代表该集群 kubelet 定制了根目录（--root-dir），因此需要在 CSI 驱动的部署文件中更新 kubelet 根目录路径
# 请将下述命令中的 {{KUBELET_DIR}} 替换成 kubelet 当前的根目录路径
# Kubernetes 版本 >= v1.18
curl -sSL https://raw.githubusercontent.com/juicedata/juicefs-csi-driver/master/deploy/k8s.yaml | sed 's@/var/lib/kubelet@{{KUBELET_DIR}}@g' | kubectl apply -f -
# Kubernetes 版本 < v1.18
curl -sSL https://raw.githubusercontent.com/juicedata/juicefs-csi-driver/master/deploy/k8s_before_v1_18.yaml | sed 's@/var/lib/kubelet@{{KUBELET_DIR}}@g' | kubectl apply -f -

#如果上方检查命令返回的结果为空，则无需修改配置，直接部署
# Kubernetes 版本 >= v1.18
kubectl apply -f https://raw.githubusercontent.com/juicedata/juicefs-csi-driver/master/deploy/k8s.yaml
# Kubernetes 版本 < v1.18
kubectl apply -f https://raw.githubusercontent.com/juicedata/juicefs-csi-driver/master/deploy/k8s_before_v1_18.yaml
```

检查部署状态

```
kubectl -n kube-system get pods -l app.kubernetes.io/name=juicefs-csi-driver
```

{{< figureCupper
img="figure4-jfs-csi.png"
caption="juicesf bench"
command="Resize"
options="1080x" >}}

创建Kubernetes Secret

```
apiVersion: v1
kind: Secret
metadata:
  name: juicefs-secret
type: Opaque
stringData:
  name: myjfs
  metaurl: redis://:jfsmeta@172.xx.xx.132:6379/1
  bucket: https://<bucket name>.oss-cn-beijing-internal.aliyuncs.com
  access-key: <AccessKey ID>
  secret-key: <AccessKey Secret>
```

创建Kubernetes StorageClass，这是动态配置必须的。

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: juicefs-sc
provisioner: csi.juicefs.com
parameters:
  csi.storage.k8s.io/provisioner-secret-name: juicefs-secret
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/node-publish-secret-name: juicefs-secret
  csi.storage.k8s.io/node-publish-secret-namespace: default
```

完成StorageClass创建后可以在应用Pod中使用JuiceFS的数据卷了。


## 部署一个服务：挂载JuiceFS数据卷

```
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: juicefs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      # 从 StorageClass 中申请 1GiB 存储容量
      storage: 1Gi
  storageClassName: juicefs-sc
---
apiVersion: v1
kind: Pod
metadata:
  name: juicefs-app
  namespace: default
spec:
  containers:
  - args:
    - -c
    - while true; do echo $(date -u) >> /data/out.txt; sleep 5; done
    command:
    - /bin/sh
    image: centos
    name: app
    volumeMounts:
    - mountPath: /data
      name: juicefs-pv
  volumes:
  - name: juicefs-pv
    persistentVolumeClaim:
      claimName: juicefs-pvc
```

完成部署后进入Pod容器中看看

```
kubectl exec -it juicefs-app -- /bin/bash
```

{{< figureCupper
img="figure5-jfs-in-pod.png"
caption="juicesf bench"
command="Resize"
options="1080x" >}}

至此，我们已经在Kubernetes中用上JuiceFS了。

回过头我们去阿里云上看看OSS服务是否有新增内容。

{{< figureCupper
img="figure6-jfs-in-oss.png"
caption="juicesf bench"
command="Resize"
options="1080x" >}}

确实，看到了myjfs这个文件系统在OSS上存储的数据内容。

