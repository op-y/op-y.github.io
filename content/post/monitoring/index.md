---
title: "有关监控"
date: 2023-09-06T15:21:00+08:00
tags: [monitoring,open-falcon,prometheus,victoria-metrics]
toc: false
---

# 写在前面

写这篇文章不为其它，只为总结一下之前做过的一些关于监控系统的工作，或许不久的将来能用得上。

# 监控系统的演进

这里说的是曾经的一段工作经历中，公司监控系统的演进过程。包括了每个时期监控系统的建设背景，发展过程中遇到的问题，以及后续的演进方向。

## 古早时期（before 2017）

在2017年之前的监控系统：没有监控系统😂！

那个时期的业务，主要还是线下服务，线上服务刚刚起步，整个机房才20来台机器。我刚过去的时候完全没有监控系统。用户和业务线同学对故障容忍度出奇的高。印象比较深刻的一次，服务挂了半天后才恢复，没有人说啥，默默地恢复服务就好了。

随着2017年初，公司开始大力发展 SaaS 服务，这种情况开始改变了。某一天**乐哥**跟我说：监控系统了解过吗？有个国产的监控系统叫 Open Falcon，你去调研一下，我们部署一套试试！

## 2017-2019

说实话，之前我一直做业务运维，用过监控系统。但是监控系统长啥样？如何实现？我是完全没接触过的。这个时候我只能硬着头皮去学习Open Falcon了。

之后的一段时间只能说过得非常充实。我认真看了Open Falcon的官方介绍和文档（当时还是v0.1.x，后来升级到v0.2.x，之后就停止迭代了，现在这个站点都已经无法打开了...）。跟着官方文档我一步一步搭建起了公司第一套监控系统。之后慢慢扩展到所有的机房。

{{< figureCupper
img="figure1-openfalcon-arch.png"
caption="Open Falcon 架构"
command="Resize"
options="1080x" >}}

之后由于运维部开发统一的运维平台，我们有了将Open Falcon集成到平台的需求。这时候就需要对Open Falcon进行二次开发了。正是这个契机，促使我认为真学习了Open Falcon的源码，对整个系统有了深入了解。在此基础上我们开发了一个定制版本的 api 模块。虽然这个模块现在已经完成了它的历史使命，但是依然在我的仓库了放着。

> [falcon-api](https://github.com/op-y/falcon-api)


在使用Open Falcon过程中，顺带还开发了很多采集器（官方的采集器不少但是真的不够用）。

> [falcon-plugins](https://github.com/op-y/falcon-plugins)

此外就是Open Falcon的 alarm 组件需要外部提供的 provider 才能真正实现告警消息的发送，为此我开发了一个功能比较丰富的 provider 程序。

> [MixProvider](https://github.com/op-y/MixProvider)

日子就这样快乐地走到了2019年，Open Falcon的使用过程中逐渐浮现初了很多问题需要解决...


## 2019-2021

2019年我们面临的问题是 Kubernetes 的流行以及在公司的全面铺开。这个时候很多业务都在向 Kubernetes 中迁移。

对于那些在物理机/虚拟机上部署的中间件Open Falcon尚能支持；对于上Kubernetes的业务，Open Falcon的能力就越来越捉襟见肘了。

由于业务程序的Pod会在集群中不停的迁移，业务程序只有两种选择：
* 一种是将监控采集逻辑集成到代码里，主动向 agent 报告数据，这种方式对业务入侵太大；
* 另一种是集群中每个节点上 agent 能动态感知当前节点上正运行着哪些服务（其实本质上就是服务发现的问题），采集数据。
我们选择了前者，但显然做得不好。

另外一个问题是，之前Open Falcon的部署，我们一直都是每个IDC独立部署一套。所有的Open Falcon系统不能统一管理，跨IDC的数据也不能聚合，存在一些使用局限。

为了解决这些问题，我们的目光又一次投向了开源社区，这个时候 Prometheus 已经成为 Kubernetes 环境下监控的*事实标准*；我们果断用了Prometheus！过程中发现它存在扩展性问题（简单的说就是单节点的极限），之后调研了几个方案最后选择了Thanos。再之后我们把监控系统彻底改造了。

{{< figureCupper
img="figure2-thanos-arch.png"
caption="Thanos Sidecar Mode 架构"
command="Resize"
options="1080x" >}}

这套架构一直持续到我离开...

## 当前的进展

后来，在最近工作经历中（虽然目前我已经很久没有做监控相关的工作了），了解到现在比较常用的架构是 Prometheus + VictoriaMetrics 的方案。主要的原因是VictoriaMetrics作为时序数据库在性能和资源使用上有远超竞品的优势。接触到好几家大厂都是使用的这种方案。

好奇也驱使我在时间充裕时做一个demo试试效果...


# 做一个Demo

主要是为了体验一下 VictoriaMetrics，我准备按照以下方式搭建一个监控系统的Demo：

* 两台云主机；
* node_exporter 采集云主机的监控指标，所以这里两台云主机上都需要部署；
* 在 ecs1 上部署一个 prometheus 完成从两个node_exporter 上抓取数据的工作；
* 在 ecs2 上部署一个单节点版本的 victoria-metrics 存储 prometheus remote_write 传输过来的数据；
* 在 ecs1 上部署一个 grafana 用于验证分别从 victoria-metrics 上查询数据的效果；
* 临时启动 ecs1 上的 nginx 将 grafana 暴露到公共网络上查看效果，完成验证后关闭。

## 安装&运行

我们的运行环境如下

* kernel: 5.10.0-15-amd64
* 系统发行版：Debian GNU/Linux 11

使用各个组件的版本

* node_exporter 1.6.1
* prometheus 2.44.0
* victoria-metrics 1.93.3
* grafana 9.5.3

### step1: 配置启动 node_exporter

由于这里只是一个demo，所以我们使用默认的 9100 端口，对其监控的指标也不做其它修改。
由于 node_exporter 是一个二进制可执行文件，所以我准备一个 nodeexporter.service 配置文件，使用 systemd 管理。

```
[Unit]
Description=Node Exporter
Documentation=https://github.com/prometheus/node_exporter

[Service]
ExecStart=/home/node_exporter-1.6.1.linux-amd64/node_exporter

[Install]
WantedBy=multi-user.target
```

```
# 启动
systemctl daemon-reload
systemctl start nodeexporter
systemctl status nodeexporter

# 查看采集指标
curl http://localhost:9100/metrics
# 如果能看到采集数据正常输出表示node_exporter正常run起来了
# 这里localhost也可以换成本机内网地址看看
# 另一台ecs也同样操作即可

```

### step2: 配置启动 victoria-metrics

由于要使用 victoria-metrics 作为 prometheus 的存储，所以先配置启动 victoria-metrics。

按照官网的说明，最简单的启动方式只需要指定 `-storageDataPath` 和 `-retentionPeriod` 选项即可。
同样的，准备一个 service 文件使用 systemd 来管理。

```
[Unit]
Description=Victoria Metrics
Documentation=https://github.com/VictoriaMetrics/VictoriaMetrics

[Service]
ExecStart=/home/victoria-metrics/victoria-metrics-prod -storageDataPath /home/victoria-metrics/victoria-metrics-data -retentionPeriod 1

[Install]
WantedBy=multi-user.target
```


```
# 启动
systemctl daemon-reload
systemctl start victoriametrices
systemctl status victoriametrices

# 确认默认的 8428 等端口已经被监听
```

### step3: 配置启动 prometheus

对于 prometheus的 配置，这里出于演示目的，仅仅修改 scrape_configs 和 remote_write 部分，用于指定从两个node_exporter上抓取机器监控数据并写入 victoria-metrics 中。

```
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  external_labels:
    datasource: prom

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "ecs"

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['<ecs1_ip>:9100', '<ecs2_ip>:9100']
        labels:
          vendor: 'aliyun'

remote_write:
  - url: http://<ecs2_ip>:8428/api/v1/write
```

同样的，写一个 service 配置，使用 systemd 启动

```
[Unit]
Description=Prometheus
Documentation=https://prometheus.io

[Service]
ExecStart=/home/prometheus-2.44.0.linux-amd64/prometheus --config.file=/home/prometheus-2.44.0.linux-amd64/prometheus.yml

[Install]
WantedBy=multi-user.target
```

```
# 启动
systemctl daemon-reload
systemctl start prometheus
systemctl status prometheus

# curl 查看 prometheus 运行状态
curl http://localhost:9090/metrics
```

至此如果前3步服务都正常了，可以先对 nginx 配置，看看 prometheus graph 效果。
此外 victoria-metrics 也自己带了一个 vmui，也可以将此 vmui 暴露出去看看效果。


### step4: 配置启动 grafana

grafana 基本按照官方配置使用默认3000端口，主要配置一下 domain 参数，创建一个 service 文件，使用 systemd 启动即可。


### step5: 配置启动nginx

这里可以分三次查看，分别是看 prometheus graph、vmui、和 grafana。

grafana 需要使用 victoria-metrics 作为一个 prometheus数据源，然后再创建一个dashboard查看效果。


## 效果

{{< figureCupper
img="figure3-prometheus-graph.png"
caption="Prometheus Graph 页面效果"
command="Resize"
options="1080x" >}}

{{< figureCupper
img="figure4-vmui.png"
caption="VictoriaMetrics vmui 页面效果"
command="Resize"
options="1080x" >}}

{{< figureCupper
img="figure5-grafana.png"
caption="Grafana 使用VictoriaMetrics数据 页面效果"
command="Resize"
options="1080x" >}}

## 继续

监控真的是一个很大的话题，有时间肯定是要继续深入的。

本次的实践涉及到的 prometheus、xxx_exporter、victoria-metrics、grafana 每一个单拎出来都有很多内容可以讲。

深感自己不足，需要更加深入学习。后续有啥进展会更新这个文档的。

# 参考

* OpenFalcon [Github 地址](https://github.com/open-falcon/falcon-plus)
* Prometheus [官方网站](https://prometheus.io)
* NodeExporter [官方使用指导](https://prometheus.io/docs/guides/node-exporter/)
* VictoriaMetrics [官方网站](https://victoriametrics.com/products/open-source/)
* Grafana [官方网站](https://grafana.com/grafana/)

再多的参考资料都不如自己动手实践，共勉！


