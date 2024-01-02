---
title: "kubeadm实战: 新增工作节点"
date: 2024-01-02T20:00:00+08:00
tags: [kubernetes,kubeadm]
toc: true
---

## 目标

这篇笔记记录的是使用 **kubeadm** 完成一个新的工作节点（worker node）加入已有 Kubernetes 集群（这个集群也是通过 kubeadm 创建的，版本为 v1.27.2。）

## 准备机器

已有集群是使用阿里云轻应用服务器作为实验环境搭建的，所以先准备一台新的集群，选用了 Ubuntu 镜像（原有节点使用的 Debian，操作验证了不同 Linux 发行版组成集群没啥问题）。

##  安装容器运行时

由于集群使用的 Kubernetes 1.27 Containerd，这里保持一致。

### 内核模块安装和内核参数设置

```
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sysctl --system
```

### 安装containerd

```
# 参考https://github.com/containerd/containerd/blob/main/docs/getting-started.md
# 版本保持一致 1.7.2
wget https://github.com/containerd/containerd/releases/download/v1.7.2/containerd-1.7.2-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-1.7.2-linux-amd64.tar.gz

# 创建 containerd 的systemd service 配置文件 /usr/local/lib/systemd/system/containerd.service
systemctl daemon-reload
systemctl enable --now containerd
```

### 安装runc

```
# 版本保持一致 1.1.7
wget https://github.com/opencontainers/runc/releases/download/v1.1.7/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc

# 如果要支持 seccomp
# apt-get install libseccomp-dev
```

### 安装 CNI 插件

```
#保持版本一致 1.3.0
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz
```

### 配置containerd

```
# 主要是 systemd cgroups驱动 和 sandbox image的配置
# 当前Linux发行版使用的是 systemd 初始化系统，后续容器运行时也需要使用 systemd cgroup 驱动
containerd config default > /etc/containerd/config.toml #生成一份默认配置后修改

systemctl restart containerd
```

## 安装 kubeadm

```
apt-get update
apt-get install -y apt-transport-https ca-certificates curl

# 官方的源由于dddd的原因无法访问 可以使用阿里云的源
#curl -fsSL https://dl.k8s.io/apt/doc/apt-key.gpg | gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
#echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
#apt-get update
#apt-get install -y kubelet kubeadm kubectl
#apt-mark hold kubelet kubeadm kubectl

apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

# 当前版本已经升级到了1.29，所以指定了一下版本
apt-get update
apt-get install kubeadm=1.27.2-00
apt-get install kubectl=1.27.2-00
apt-get install kubelet=1.27.2-00

```

## 创建token

```
# 在控制面节点上创建一个token 供worker节点加入时使用
kubeadm token create
```

## 节点加入

```
# token 和 hash 可以参考官方文档获取
kubeadm join --token xxxx master.xxx.xxx:6443 --discovery-token-ca-cert-hash sha256:xxxx
```

{{% note %}}
如果是创建集群控制面，参考官方文档在控制面节点执行 kubeadm init 操作。
在其它实践练习中再进行相关的操作记录。
{{% /note %}}


## 可选： 安装nerdctl 和 buildkit

由于直接使用了containerd，没有docker工具可用了，可以安装nerdctl和buildkit支持镜像构建和管理

```
# nerdctl 以下两个版本均可
#wget https://github.com/containerd/nerdctl/releases/download/v1.4.0/nerdctl-full-1.4.0-linux-amd64.tar.gz
wget https://github.com/containerd/nerdctl/releases/download/v1.4.0/nerdctl-1.4.0-linux-amd64.tar.gz

# .zshrc 做几个别名方便使用
# alias crictl="crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock"
# alias docker="nerdctl"
# alias docker-compose="nerdctl compose"
# source <(kubectl completion zsh)

# buildkit
wget https://github.com/moby/buildkit/releases/download/v0.11.6/buildkit-v0.11.6.linux-amd64.tar.gz
tar -zxvf buildkit-v0.11.6.linux-amd64.tar.gz
mv bin/* /usr/local/bin

# 配置 /etc/systemd/system/buildkitd.service
systemctl enable --now buildkitd
```

## 效果

{{< figureCupper
img="figure1-k8s-new-node.jpg"
caption="使用kubeadm新增工作节点"
command="Resize"
options="1080x" >}}

