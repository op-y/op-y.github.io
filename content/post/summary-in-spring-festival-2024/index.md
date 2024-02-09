---
title: "2024春节前总结"
date: 2024-02-09T20:00:00+08:00
tags: [summary]
toc: true
---

## 新年好！

一晃又春节了，从八月底到现在已经五个多月了。这段时间学的做的内容也不少，在新年之前做一下整理，为假期后的工作做些准备。

## 工作回顾

* [OpenResty](https://op-y.github.io/openresty/)
* [Kong](https://op-y.github.io/kong/)
* [Traefik](https://op-y.github.io/traefik/)
* [CI/CD系统](https://op-y.github.io/cicd/)
* [监控系统](https://op-y.github.io/monitoring/)
* [HTTPS](https://op-y.github.io/https/) 完成了一部分，内容有点庞大
* [DNS项目](https://github.com/op-y/gdns)

## 专项知识点

### Go并发编程

二刷课程[Go 并发编程实战课](https://time.geekbang.org/column/intro/100061801?tab=catalog) 和书籍 [Go语言并发之道](https://book.douban.com/subject/30424330/)，写了一些[代码片段](https://github.com/op-y/hello-world/tree/master/go-concurrency)。

### gRPC

二刷了[《gRPC与云原生应用开发：以Go和Java为例》](https://www.ituring.com.cn/book/2775) 系统学习了gRPC编程。
整理和重新编写了相关的示例[代码](https://github.com/op-y/hello-world/tree/master/grpc/yflow)，这了使用了Go和Python。

### prometheus

在工作回顾中监控系统部分提到过prometheus相关内容，在此基础上，参考官方文档，做了一个[Go代码Instrumentation](https://github.com/op-y/hello-world/tree/master/prometheus-instrumentation)。顺道把prometheus client-go 代码学习了一遍。

### 性能

重新学习了两门课程，在阿里云上做了一些实践。

* [Linux 性能优化实战](https://time.geekbang.org/column/intro/100020901?tab=catalog) 后续做系统整理。
* [eBPF 核心技术与实战](https://time.geekbang.org/column/intro/100104501?tab=catalog) eBPF确实是太底层了，要掌握很不容易，后续还要学习更多资料和做更多的实践。

## kubernetes

### kubernetes 实战

* [kubeadm实战: 新增工作节点](https://op-y.github.io/kubeadm-node-join/)
* [kubeadm实战: 删除工作节点](https://op-y.github.io/kubeadm-node-delete/)
* [kubeadm实战: 升级集群](https://op-y.github.io/kubeadm-upgrade-cluster/)
* [kubeadm实战: 搭建高可用集群](https://op-y.github.io/kubeadm-ha-cluster/)
* [kubernetes存储实战: 使用JuiceFS](https://op-y.github.io/juicefs-in-kubernetes/)
* 还有一个关于 calico 网络插件的实践，由于在阿里云环境上最终没有实现预期效果，所以没有写文档记录。

### kubernetes operator开发

按照内容递进阅读了书籍

* [Kubernetes 编程](https://book.douban.com/subject/35498478/)
* [Kubernetes Operator](https://book.douban.com/subject/34796009/)
* [Kubernetes Operator 开发进阶](https://book.douban.com/subject/36209350/)

阅读了官方文档

* [Kubebuilder](https://kubebuilder.io)
* [Operator SDK](https://sdk.operatorframework.io)

总做了几个[Demo示例](https://github.com/op-y/hello-world/tree/master/operators)

### 代码学习

顺道学习了Kubernetes 以及 client-go 代码，参考网络上资料做几个实践

* [聚合 APIServer](https://github.com/op-y/hello-world/tree/master/apiservice)
* [自定义 Scheduler](https://github.com/op-y/hello-world/tree/master/scheduler)

## LeetCode 算法题

完成了LeetCode上编号 1 至 250 之间没有被锁的算题。

## offer

面试了4家公司，拿到了两个offer，目前都是SRE相关工作。

## 冬天❄️

> 在隆冬，我终于知道，我身上有一个不可战胜的夏天！

{{< figureCupper
img="Albert-Camus-by-Bresson.jpeg"
caption="Albert Camus"
command="Resize"
options="1080x" >}}

