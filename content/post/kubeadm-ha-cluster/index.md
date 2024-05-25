---
title: "kubeadm实战: 搭建高可用集群"
date: 2024-01-07T20:30:00+08:00
tags: [kubernetes,kubeadm]
toc: true
---

## 目标

本篇记录使用 **kubeadm** 完成一个高可用 Kubernetes 集群搭建。

参考文档

* [容器运行时](https://v1-28.docs.kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/)
* [安装 kubernetes](https://v1-28.docs.kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
* [高可用拓扑选项](https://v1-28.docs.kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/ha-topology/)
* [使用 Kubeadm 创建一个高可用 etcd 集群](https://v1-28.docs.kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/)
* [etcd 配置](https://etcd.io/docs/v3.5/op-guide/configuration/)
* [利用 kubeadm 创建高可用集群](https://v1-28.docs.kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/high-availability/)
* [flannel 配置](https://github.com/flannel-io/flannel/blob/master/Documentation/configuration.md)

{{% note %}}
使用的版本 Kubernetes 1.28.2
{{% /note %}}


## 环境准备

已有集群是使用阿里云轻应用服务器作为实验环境搭建的，所以先准备一台新的集群，选用了 Ubuntu 镜像（原有节点使用的 Debian，操作验证了不同 Linux 发行版组成集群没啥问题）。

### 安装一些必要的软件包

```
apt-get update
apt-get install git lrzsz tmux zsh fonts-powerline

# lrzsz 用于本地终端与云主机传输小文件
# tmux 方便登录云主机操作
# zsh 替换云主机shell环境 如果喜欢bash也可以保持默认环境
# fonts-powerline 是后续使用oh-my-zsh主题需要的字体
```

### 安装 oh-my-zsh

参考官方文档 https://ohmyz.sh/#install 过程非常简单

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```
 
安装完成后，可以编辑 `~/.zshrc` 配置文件更换主题，这里我个人习惯使用 *agnoster* 。

### 配置本地Mac上iTerm2的rzsz

{{% note %}}
使用习惯问题，不实用rzsz工具情况下，也可不做以下配置。
{{% /note %}}

在 /usr/local/bin 下准备两个脚本文件

iterm2-recv-zmodem.sh（从网上抄的）
```
#!/bin/bash
# Author: Matt Mastracci (matthew@mastracci.com)
# AppleScript from http://stackoverflow.com/questions/4309087/cancel-button-on-osascript-in-a-bash-script
# licensed under cc-wiki with attribution required
# Remainder of script public domain

osascript -e 'tell application "iTerm2" to version' > /dev/null 2>&1 && NAME=iTerm2 || NAME=iTerm
if [[ $NAME = "iTerm" ]]; then
	FILE=$(osascript -e 'tell application "iTerm" to activate' -e 'tell application "iTerm" to set thefile to choose folder with prompt "Choose a folder to place received files in"' -e "do shell script (\"echo \"&(quoted form of POSIX path of thefile as Unicode text)&\"\")")
else
	FILE=$(osascript -e 'tell application "iTerm2" to activate' -e 'tell application "iTerm2" to set thefile to choose folder with prompt "Choose a folder to place received files in"' -e "do shell script (\"echo \"&(quoted form of POSIX path of thefile as Unicode text)&\"\")")
fi

if [[ $FILE = "" ]]; then
	echo Cancelled.
	# Send ZModem cancel
	echo -e \\x18\\x18\\x18\\x18\\x18
	sleep 1
	echo
	echo \# Cancelled transfer
else
	cd "$FILE"
	/usr/local/bin/rz --rename --escape --binary --bufsize 4096
	sleep 1
	echo
	echo
	echo \# Sent \-\> $FILE
fi
```

iterm2-send-zmodem.sh（从网上抄的）
```
#!/bin/bash
# Author: Matt Mastracci (matthew@mastracci.com)
# AppleScript from http://stackoverflow.com/questions/4309087/cancel-button-on-osascript-in-a-bash-script
# licensed under cc-wiki with attribution required
# Remainder of script public domain

osascript -e 'tell application "iTerm2" to version' > /dev/null 2>&1 && NAME=iTerm2 || NAME=iTerm
if [[ $NAME = "iTerm" ]]; then
	FILE=$(osascript -e 'tell application "iTerm" to activate' -e 'tell application "iTerm" to set thefile to choose file with prompt "Choose a file to send"' -e "do shell script (\"echo \"&(quoted form of POSIX path of thefile as Unicode text)&\"\")")
else
	FILE=$(osascript -e 'tell application "iTerm2" to activate' -e 'tell application "iTerm2" to set thefile to choose file with prompt "Choose a file to send"' -e "do shell script (\"echo \"&(quoted form of POSIX path of thefile as Unicode text)&\"\")")
fi
if [[ $FILE = "" ]]; then
	echo Cancelled.
	# Send ZModem cancel
	echo -e \\x18\\x18\\x18\\x18\\x18
	sleep 1
	echo
	echo \# Cancelled transfer
else
	/usr/local/bin/sz "$FILE" --escape --binary --bufsize 4096
	sleep 1
	echo
	echo \# Received "$FILE"
fi
```

在iTerm2对应的Profile上配置触发器

{{< figureCupper
img="figure1-iterm2-triggers.png"
caption="配置 iTerm2 触发器"
command="Resize"
options="1080x" >}}

| Regular Expression                | ... |
| ----------------------------------| --- |
|rz waiting to receive.\\\*\\\*B0100| ... |
|\\\*\\\*B00000000000000            | ... |

### 配置本地域名解析

在所有机器（实验环境是3台）的 /etc/hosts 中增加解析条目

```
# 以下是新增条目IP和域名做了部分掩盖

172.xx.xx.198 blue.xxx.apps
172.xx.xx.129 red.xxx.apps
172.xx.xx.132 yellow.xxx.apps

172.xx.xx.132 apiserver.xxx.apps

```

因为操作是为了方便区分机器，在终端上给三台机器笔记了红黄蓝三种颜色，索性就用了 blue、red、yellow做了主机域名。
同时先预设了 kube-apiserver 控制面域名先解析到 yellow上。

### 其它配置

为了方便机器间互相访问，可以给机器都互相添加ssh信任关系，实现免密ssh登录和scp传文件等。


##  容器运行时

Kubernetes 底层依赖符合CNI规范的容器运行时，所以在部署集群之前需要安装容器运行时，这里选择最常用的 containerd。
安装步骤可以参考前述官方文档。

### 内核模块安装和内核参数设置

```
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sysctl --system

# 通过运行以下指令确认 br_netfilter 和 overlay 模块被加载
lsmod | grep br_netfilter
lsmod | grep overlay

# 通过运行以下指令确认 net.bridge.bridge-nf-call-iptables、net.bridge.bridge-nf-call-ip6tables 和 net.ipv4.ip_forward 系统变量在你的 sysctl 配置中被设置为 1
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

### 安装containerd

```
# 参考https://github.com/containerd/containerd/blob/main/docs/getting-started.md
wget https://github.com/containerd/containerd/releases/download/v1.7.2/containerd-1.7.2-linux-amd64.tar.gz
mkdir -p /usr/local/lib/systemd/system/
tar Cxzvf /usr/local containerd-1.7.2-linux-amd64.tar.gz

# 创建 containerd 的systemd service 配置文件 /usr/local/lib/systemd/system/containerd.service
systemctl daemon-reload
systemctl enable --now containerd
```

containerd.service
```
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

### 安装runc

```
wget https://github.com/opencontainers/runc/releases/download/v1.1.7/runc.amd64 
install -m 755 runc.amd64 /usr/local/sbin/runc

# 如果要支持 seccomp
# apt-get install libseccomp-dev
```

### 安装 CNI 插件

```
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz
```

### 配置containerd

当前Linux发行版使用的是 systemd 初始化系统，配置容器运行时也使用 systemd cgroup 驱动，之后启动kubelet也保持一致。

```
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml #生成一份默认配置

# vim /etc/containerd/config.toml
# 修改 SystemdCgroup = true 
# 修改 sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"

systemctl restart containerd
```

### nerdctl 和 buildkit

{{% note %}}
containerd 自身提供了一个crictl 做完客户端工具进行操作，十分不好用，可以安装nerdctl 和 buildkit 来实现近似 docker的功能。
{{% /note %}}


```
# nerdctl
wget https://github.com/containerd/nerdctl/releases/download/v1.4.0/nerdctl-1.4.0-linux-amd64.tar.gz
tar Czxvf /usr/local/bin/ nerdctl-1.4.0-linux-amd64.tar.gz

# .zshrc 做几个别名方便使用
# alias crictl="crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock"
# alias docker="nerdctl"
# alias docker-compose="nerdctl compose"

# buildkit
wget https://github.com/moby/buildkit/releases/download/v0.11.6/buildkit-v0.11.6.linux-amd64.tar.gz
tar Czxvf /usr/local buildkit-v0.11.6.linux-amd64.tar.gz

# 配置 /usr/local/lib/systemd/system/buildkit.service
systemctl enable --now buildkit
```
buildkitd.service

```
[Unit]
Description=BuildKit
Documentation=https://github.com/moby/buildkit

[Service]
Type=notify
ExecStart=/usr/local/bin/buildkitd --oci-worker=false --containerd-worker=true

[Install]
WantedBy=multi-user.target
```


## 安装 kubeadm

{{% note %}}
如果使用 openssl 等工具给 etcd 生成证书可以先安装etcd，然后再安装 kubeadm。
这里借用kubeadm来生成证书，所以先把 kubeadm kubelet kubectl 一起安装了。
{{% /note %}}


```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# 使用阿里云的源 原因大家都懂
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

# 当前版本已经升级到了1.29，所以指定了一下版本
sudo apt-get update
sudo apt-get install kubeadm kubectl kubelet
sudo apt-mark hold kubelet kubeadm kubectl

# .zshrc 做个kubectl补全配置
# source <(kubectl completion zsh)
```

## 安装etcd集群

这里参考了官方文档使用 kubeadm 来生成相关证书文件，但是不使用 kubelet 托管etcd进程，而是在机器上直接启动。

使用以下脚本为三台机器的kubeadm生成 kubeadm-config.yaml

```
# 使用你的主机 IP 更新 HOST0、HOST1 和 HOST2 的 IP 地址
# 这里做了掩码
export HOST0=172.xx.xx.132
export HOST1=172.xx.xx.129
export HOST2=172.xx.xx.198

# 使用你的主机名更新 NAME0、NAME1 和 NAME2
# 这里做了掩码
export NAME0="yelllow.xxx.apps"
export NAME1="red.xxx.apps"
export NAME2="blue.xxx.apps"

# 创建临时目录来存储将被分发到其它主机上的文件
mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

HOSTS=(${HOST0} ${HOST1} ${HOST2})
NAMES=(${NAME0} ${NAME1} ${NAME2})

for i in "${!HOSTS[@]}"; do
HOST=${HOSTS[$i]}
NAME=${NAMES[$i]}
cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
---
apiVersion: "kubeadm.k8s.io/v1beta3"
kind: InitConfiguration
nodeRegistration:
 name: ${NAME}
localAPIEndpoint:
 advertiseAddress: ${HOST}
---
apiVersion: "kubeadm.k8s.io/v1beta3"
kind: ClusterConfiguration
etcd:
 local:
     serverCertSANs:
     - "${HOST}"
     peerCertSANs:
     - "${HOST}"
     extraArgs:
         initial-cluster: ${NAMES[0]}=https://${HOSTS[0]}:2380,${NAMES[1]}=https://${HOSTS[1]}:2380,${NAMES[2]}=https://${HOSTS[2]}:2380
         initial-cluster-state: new
         name: ${NAME}
         listen-peer-urls: https://${HOST}:2380
         listen-client-urls: https://${HOST}:2379
         advertise-client-urls: https://${HOST}:2379
         initial-advertise-peer-urls: https://${HOST}:2380
EOF
done
```

在yellow节点上创建ca证书

```
kubeadm init phase certs etcd-ca

# 生成文件
# /etc/kubernetes/pki/etcd/ca.crt
# /etc/kubernetes/pki/etcd/ca.key
```

为成员节点创建证书

```
# 使用你的主机 IP 更新 HOST0、HOST1 和 HOST2 的 IP 地址
# 这里做了掩码
export HOST0=172.xx.xx.132
export HOST1=172.xx.xx.129
export HOST2=172.xx.xx.198

# 使用你的主机名更新 NAME0、NAME1 和 NAME2
# 这里做了掩码
export NAME0="yelllow.xxx.apps"
export NAME1="red.xxx.apps"
export NAME2="blue.xxx.apps"

kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST2}/
# 清理不可重复使用的证书
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST1}/
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
# 不需要移动 certs 因为它们是给 HOST0 使用的

# 清理不应从此主机复制的证书
find /tmp/${HOST2} -name ca.key -type f -delete
find /tmp/${HOST1} -name ca.key -type f -delete
```

复制证书和kubeadm配置到目标机器

```
HOST=${HOST1}
scp -r /tmp/${HOST}/* root@${HOST}:
ssh root@${HOST}
mv pki /etc/kubernetes/

HOST=${HOST2}
scp -r /tmp/${HOST}/* root@${HOST}:
ssh root@${HOST}
mv pki /etc/kubernetes/
```

操作完成后可以登录所有机器检查证书和配置文件是否正确。

然后在三台机器上下载etcd的二进制包，按照集群模式启动。以下是三台机器上启动etcd的脚本。

```
#!/bin/bash

nohup /home/etcd/etcd \
--advertise-client-urls=https://172.xx.xx.132:2379 \
--cert-file=/etc/kubernetes/pki/etcd/server.crt \
--client-cert-auth=true \
--data-dir=/var/lib/etcd \
--experimental-initial-corrupt-check=true \
--experimental-watch-progress-notify-interval=5s \
--initial-advertise-peer-urls=https://172.xx.xx.132:2380 \
--initial-cluster=yellow.xxx.apps=https://172.xx.xx.132:2380,red.xxx.apps=https://172.xxx.xxx.129:2380,blue.xxx.apps=https://172.xx.xx.198:2380 \
--key-file=/etc/kubernetes/pki/etcd/server.key \
--listen-client-urls=https://172.xx.xx.132:2379 \
--listen-metrics-urls=http://127.0.0.1:2381 \
--listen-peer-urls=https://172.xx.xx.132:2380 \
--name=yellow.xxx.apps \
--peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt \
--peer-client-cert-auth=true \
--peer-key-file=/etc/kubernetes/pki/etcd/peer.key \
--peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt \
--snapshot-count=10000 \
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt \
--initial-cluster k8s \
--initial-cluster-state new --initial-cluster-token k8s \
1>start.log 2>start.err &


# 这个是yellow节点的
# red 和 blue 节点对应修改IP地址和域名即可
```

etcd 集群启动成功后，可以使用etcdctl客户端工具查看集群状态。

```
ETCDCTL_API=3 /home/etcd/etcdctl --cert /etc/kubernetes/pki/apiserver-etcd-client.crt --key /etc/kubernetes/pki/apiserver-etcd-client.key --cacert /etc/kubernetes/pki/etcd/ca.crt --endpoints https://yellow.xxx.apps:2379,https://red.xxx.apps:2379,https://blue.xxx.apps:2379 member list
```

{{< figureCupper
img="figure2-etcd-cluster.png"
caption="etcd 集群"
command="Resize"
options="1080x" >}}

## 启动第一个控制面

### 为控制面apiserver配置一个负载均衡

这里没有做LoadBalancer配置，如果需要可以用 Nginx、LVS 等配置，实验中直接把域名 apiserver.xxx.apps 配置到yellow节点上。

### 准备kubeadm配置文件

~/kubeadm-config.yaml

```
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "apiserver.xxx.apps:6443"
imageRepository: registry.aliyuncs.com/google_containers
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.74.0.0/16
etcd:
  external:
    endpoints:
      - https://172.xx.xx.132:2379
      - https://172.xx.xx.129:2379
      - https://172.xx.xx.198:2379
    caFile: /etc/kubernetes/pki/etcd/ca.crt
    certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
    keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
```

这里要注意几个配置

* controlPlaneEndpoint 高可用集群必须要配置这个参数
* imageRepository 指定了从阿里云镜像仓库下载镜像 原因都懂
* serviceSubnet 分配service IP网段
* podSubnet 分配节点pod IP 网段
* etcd.external 下是外部etcd 集群的相关配置


{{% warning %}}
由于 podSubnet 没有配置导致 flannel 第一次启动异常！
{{% /warning %}}

### 初始化第一个控制面

```
kubeadm init --config kubeadm-config.yaml --upload-certs
```

输出内容如下，IP 域名 机器名经过处理。

```
I0106 12:53:52.403941   61596 version.go:256] remote version is much newer: v1.29.0; falling back to: stable-1.28
[init] Using Kubernetes version: v1.28.5
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [apiserver.xxx.apps yellow kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.xx.xx.132]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] External etcd mode: Skipping etcd/ca certificate authority generation
[certs] External etcd mode: Skipping etcd/server certificate generation
[certs] External etcd mode: Skipping etcd/peer certificate generation
[certs] External etcd mode: Skipping etcd/healthcheck-client certificate generation
[certs] External etcd mode: Skipping apiserver-etcd-client certificate generation
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 7.004820 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
<key>
[mark-control-plane] Marking the node yellow as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node yellow as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: <token>
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join apiserver.xxx.apps:6443 --token <token> \
	--discovery-token-ca-cert-hash sha256:<hash> \
	--control-plane --certificate-key <key>

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join apiserver.xxx.apps:6443 --token <token> \
	--discovery-token-ca-cert-hash sha256:<hash>
```

### 安装CNI插件

经过一番琢磨还是选择了熟悉的 flannel。

```
# 这里是把文件下载到了本地
kubectl apply -f kube-flannel.yaml
```

{{% warning %}}
由于一开始 kubeadm-config.yaml 中 podSubnet 没有配置导致 flannel 一开启动异常。报错 *Error registering network: failed to acquire lease: node pod cidr not assigned* 。

期间以为是 flannel 启动参数没有设置 --etcd-endpoints 为外部集群端点，加上之后同时加上证书相关配置，也无法正常启动。后来搜索了一下，基本都指向是节点网段没有正常分批的问题。所以在 kubeadm-config.yaml 中加上 podSubnet 配置，去掉flannel etcd证书配置，然后启动成功了。

这里又引入一个问题，Kubernetes中的flannel 没有配置etcd证书，为什么能正常运行？后来经过与朋友讨论，和查证资料，发现flannel可以与kubernetes通过API交互。最终确认恐怕还需要查看flannel代码。
{{% /warning %}}

## 启动第二个控制面

从第一个控制面初始化输出中cp出命令执行。

```
kubeadm join apiserver.xxx.apps:6443 --token <token> \
        --discovery-token-ca-cert-hash sha256:<hash> \
        --control-plane --certificate-key <key>
```

输出内容如下，IP 域名 机器名经过处理。

```
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks before initializing the new control plane instance
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[download-certs] Downloading the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[download-certs] Saving the certificates to the folder: "/etc/kubernetes/pki"
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [apiserver.xxx.apps red kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.xx.xx.129]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[certs] Using the existing "sa" key
[kubeconfig] Generating kubeconfig files
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[check-etcd] Skipping etcd check in external mode
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
[control-plane-join] Using external etcd - no local stacked instance added
The 'update-status' phase is deprecated and will be removed in a future release. Currently it performs no operation
[mark-control-plane] Marking the node red as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node red as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]

This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.


To start administering your cluster from this node, you need to run the following as a regular user:

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
```

## 启动一个工作节点

```
kubeadm join apiserver.xxx.apps:6443 --token <token> \
        --discovery-token-ca-cert-hash sha256:<hash>
```

输出内容如下，IP 域名 机器名经过处理。

```
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

## 集群效果

如图

{{< figureCupper
img="figure3-k8s-cluster.png"
caption="两个控制面节点的kubernetes集群"
command="Resize"
options="1080x" >}}

## 验证高可用

创建一个2副本的nginx Deployment和Service

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dp
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.14
        name: nginx
        ports:
          - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-srv
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

正常部署。

将yellow节点上，kube-controller-manager 静态Pod的 manifest 文件mv到其它地方，这个Pod会主动消失，根据观察，red节点上的 kube-controller-manager 正常接手了controller manager的工作。

此时将nginx的副本书调整为4，新增的Pod正常被创建出来。

{{< figureCupper
img="figure4-nginx-deployment.png"
caption="Nginx Deployment"
command="Resize"
options="1080x" >}}

至此初步验证了组件的高可用。


## 后记

折腾kubernetes真是一件很复杂又有趣的事情。在实验环境部署这个玩具高可用集群前前后后花了几个小时。最终还是折腾出来了。过程中也暴露出一些问题。

* kubernetes 各个组件的参数很多，真的需要认真研究各个参数的意义和作用；
* kubeadm 配置非常的灵活，需要花时间多做实验；
* etcd和flannel 有些生疏，有时间多练习，看看源码；
* kubernetes 网络是绕不过去的问题，本来打算尝试calico的，后来还是放弃了，抽时间单独学习calico、weave、canal、cilium 这些CNI网络组件的使用吧；
* 一直说网络和存储带来痛苦，实在是没办法，痛苦也要学呀... 

希望自己能一直保持这样的好奇心和求知欲，*stay foolish, stay hungry*，共勉！




