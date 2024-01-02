---
title: "kubeadm实战: 删除工作节点"
date: 2024-01-02T20:00:00+08:00
tags: [kubernetes,kubeadm]
toc: true
---

## 目标

这篇笔记记录的是使用 **kubeadm** 从当前 Kubernetes 集群中删除一个工作节点。

## 移除节点

```
# 驱逐节点上全部Pod
kubectl drain <nodename> --delete-emptydir-data --force --ignore-daemonsets

# reset
kubeadm reset

# 输出
W0102 11:42:46.315993 2170165 preflight.go:56] [reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
[reset] Are you sure you want to proceed? [y/N]: y
[preflight] Running pre-flight checks
W0102 11:42:49.869811 2170165 removeetcdmember.go:106] [reset] No kubeadm config, using etcd pod spec to get data directory
[reset] Deleted contents of the etcd data directory: /var/lib/etcd
[reset] Stopping the kubelet service
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Deleting contents of directories: [/etc/kubernetes/manifests /var/lib/kubelet /etc/kubernetes/pki]
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]

The reset process does not clean CNI configuration. To do so, you must remove /etc/cni/net.d

The reset process does not reset or clean up iptables rules or IPVS tables.
If you wish to reset iptables, you must do so manually by using the "iptables" command.

If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
to reset your system's IPVS tables.

The reset process does not clean your kubeconfig files and you must remove them manually.
Please, check the contents of the $HOME/.kube/config file.

# 清除iptables条目
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X

# 清除ipvs数据
ipvsadm -C

# 从集群中删除工作节点
kubectl delete node <nodename>
```

