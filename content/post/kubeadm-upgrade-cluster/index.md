---
title: "kubeadm实战: 升级集群"
date: 2024-01-02T20:46:00+08:00
tags: [kubernetes,kubeadm]
toc: true
---

## 目标

这篇笔记记录的是使用 **kubeadm** 完成 Kubernetes 集群从 v1.27.2 到 v1.28.2 的升级。

## 准备工作

查看可以升级的版本

```
apt update
apt-cache madison kubeadm
```

升级kubeadm

```
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm='1.28.2-00*' && \
apt-mark hold kubeadm

# 确认版本
kubeadm version
```

## 升级控制面

```
# 确认升级计划
kubeadm upgrade plan v1.28.2

# dry-run 可以看有哪些更改
# kubeadm upgrade apply v1.28.2 --dry-run

# 升级控制面
kubeadm upgrade apply v1.28.2

# 输出
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade/version] You have chosen to change the cluster version to "v1.28.2"
[upgrade/versions] Cluster version: v1.27.2
[upgrade/versions] kubeadm version: v1.28.2
[upgrade] Are you sure you want to proceed? [y/N]: y
[upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
[upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
[upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
[upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.28.2" (timeout: 5m0s)...
[upgrade/etcd] Upgrading to TLS for etcd
[upgrade/staticpods] Preparing for "etcd" upgrade
[upgrade/staticpods] Renewing etcd-server certificate
[upgrade/staticpods] Renewing etcd-peer certificate
[upgrade/staticpods] Renewing etcd-healthcheck-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-01-01-23-45-23/etcd.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
E0101 23:46:00.227491 1246646 request.go:1116] Unexpected error when reading response body: net/http: request canceled (Client.Timeout or context cancellation while reading body)
[apiclient] Found 1 Pods for label selector component=etcd
[upgrade/staticpods] Component "etcd" upgraded successfully!
[upgrade/etcd] Waiting for etcd to become available
[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests1583548336"
[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Renewing apiserver certificate
[upgrade/staticpods] Renewing apiserver-kubelet-client certificate
[upgrade/staticpods] Renewing front-proxy-client certificate
[upgrade/staticpods] Renewing apiserver-etcd-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-01-01-23-45-23/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
[apiclient] Found 1 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Renewing controller-manager.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-01-01-23-45-23/kube-controller-manager.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
[apiclient] Found 1 Pods for label selector component=kube-controller-manager
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Renewing scheduler.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-01-01-23-45-23/kube-scheduler.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
[apiclient] Found 1 Pods for label selector component=kube-scheduler
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config3821068939/config.yaml
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.28.2". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.

# 升级控制面kubelet
kubectl drain <nodename> --ignore-daemonsets

apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet='1.28.2-00' kubectl='1.28.2-00' && \
apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon <nodename>
```


## 升级工作节点2

{{% note %}}
工作节点上的升级过程应该一次执行一个节点，或者一次执行几个节点， 以不影响运行工作负载所需的最小容量。
{{% /note %}}

```
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.28.2-00 && \
apt-mark hold kubeadm

kubeadm upgrade node

# 输出
[upgrade] Reading configuration from the cluster...
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks
[preflight] Skipping prepull. Not a control plane node.
[upgrade] Skipping phase. Not a control plane node.
[upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config3984534232/config.yaml
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.

# 升级工作节点kubelet
kubectl drain <nodename>  --ignore-daemonsets

apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.28.2-00 kubectl=1.28.2-00 && \
apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

kubectl uncordon <nodename>
```

## 升级工作节点1 操作同上

过程中遇到一些异常

```
error: unable to drain node "iz2zefyb2ejhxw9uh3c52wz" due to error:[cannot delete Pods with local storage (use --delete-emptydir-data to override): <pod list ...>, cannot delete Pods declare no controller (use --force to override): <pod list ...>], continuing command...
There are pending nodes to be drained:
 <nodename>
...
```

以上报错有两个原因引起：

* 有些Pod使用了了本地 emptydir 数据卷，直接删除会导致数据丢失，根据提示使用 `--delete-emptydir-data` 可以直接删除；
* 有些Pod是直接启动的没有控制器，根据提示使用 `--force` 可以强制删除。 

当然生产环境中还是要根据实际情况进行处理。

## 效果


{{< figureCupper
img="figure1-kubeadm-upgrade-cluster.png"
caption="使用kubeadm升级集群"
command="Resize"
options="1080x" >}}

