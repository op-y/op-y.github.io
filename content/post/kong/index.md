---
title: "Kong 回顾"
date: 2023-09-05T22:52:00+08:00
tags: [nginx,openresty,kong,lua]
toc: false
---

# 引子

稍早之前对 OpenResty 进行了一个回顾，记得之前工作中，在接入网关升级改造中，使用 OpenResty 之后，我们又引入了 Kong API Gateway，所以也有必要对 Kong 简单的重新操作一下，做个记录。


# Kong 是什么？

Kong Gateway 自己的[官方的介绍](https://docs.konghq.com/gateway/latest/)

{{< figureCupper
img="figure1-kong-logo.png"
caption="Kong 的名字来自金刚 Logo就是这只猩猩"
command="Resize"
options="128x" >}}

比较简单的理解：Kong Gateway 就是一个基于 OpenResty （不知道 OpenResty 是什么？去目录上找相关的文章）的 API Gateway （所以本质上还是 Nginx + Lua)。

{{< figureCupper
img="figure2-kong-arch.png"
caption="Kong Gateway 结构"
command="Resize"
options="1080x" >}}

至于 API Gateway 是什么？我个人比较认可它是一个贴近微服务架构的概念，它实现了一种Facade模式，统一各个服务的反向代理和请求管理，包括并不限于 认证、权限、限流、熔断、安全、监控、日志...
从实现上说，它可以是个Nginx，也可以是一个OpenResty，还可以是 Kong、APISIX、BFE 等等。


# 这些年的变化

上一次自己部署 Kong 的时候，Kong 的版本还是 1.3，当时相比 0.1.x 版本已经有了很大的变化，是那种概念都不一样的非兼容变化。今天看了下官网，好家伙，都已经是 3.4.x 了，从 Changelogs 上看有了不小的调整（但是主要概念没有发生变化）。我看完后认为比较大的变化是：

* 现在有4种部署模式
	* Konnect：这个1.3的时候还没注意到，是一个Kong Gateway的SaaS服务；
	* Hybird: 这个是一个新的模式，将服务分成了 Control Panel和Data Panel，也是当下一种比较流行的做法；
	* Tranditional：这个就是依赖DB部署的传统做法了，本质上时通过DB实现配置的中心化管理和分发；
	* Db-less and declarative: 这个时无DB的声明式配置模式，单机部署相当的友好。
* Traditional with-DB 模式下不再支持 Cassandra DB了（从3.4开始的）。之前的版（1.3.x之后）增加了对 Redis、InfluxDB、Kafka 的支持。当然 Postgres 一直都被支持。之前我部署时选择自己搭建了一个 Cassandra 集群，就是因为 Postgres 的双主同步使用了一个 Perl 编写的 bucardo 工具，那简直是的搞不明白。
* 官方增加了一个 Kong Manager 项目，是一个GUI，早先没有官方GUI的，当时我使用的一个第三方GUI项目[Konga](https://github.com/pantsel/konga)，部署的时候还向前端同学求助过。

# Kong 使用

## 安装

本次操作由于不想依赖一个DB，所以我在开发环境部署一个DB-less模式的 Kong Gateway 服务。顺便验证一些这种模式下 Kong Manager是不是可以使用。

我们的运行环境如下

* kernel: 5.10.0-15-amd64
* 系统发行版：Debian GNU/Linux 11

开始安装
{{<cmd>}}
# 前置要求 curl lsb-release apt-transport-https 已经安装

# 添加 Kong 的 apt 源
curl -1sLf "https://packages.konghq.com/public/gateway-34/gpg.6B5D054B0707DE3B.key" |  gpg --dearmor | sudo tee /usr/share/keyrings/kong-gateway-34-archive-keyring.gpg > /dev/null
curl -1sLf "https://packages.konghq.com/public/gateway-34/config.deb.txt?distro=debian&codename=$(lsb_release -sc)" | sudo tee /etc/apt/sources.list.d/kong-gateway-34.list > /dev/null

# apt-get 安装
apt install -y kong-enterprise-edition=3.4.0.0

# 安装完成后根据过往经验确认了几点
# 自动创建了一个用户 kong 用于，用于在non-root模式下启动kong
# kong安装在/usr/local/kong下，顺带安装了一个 /usr/local/openresty
# 同时顺道安装了/usr/local/bin/luarocks
# 可以看看 /usr/local/lib
# lua 5.1 作为动态链接库安装到了 /usr/local/lib/lua 目录下
# lua 的各种包在 /usr/local/lib/luarocks 目录下
# /etc/kong 目录中放着kong的初始配置文件模板

# 为了方便管理到配置目录下操作
cd /etc/kong
kong config init # 生成DB-less模式声明式配置
cp kong.conf.default kong.conf # 复制一个运行使用的kong配置文件

# 启动前需要修改 kong.conf 文件
{{</cmd>}}


主要修改的配置有

```
# nginx 监听端口暴露到公网方便测试
proxy_listen = 0.0.0.0:80 reuseport backlog=16384, 0.0.0.0:443 http2 ssl reuseport backlog=16384

# DB-less 模式
database = off
declarative_config = /etc/kong/kong.yml

# Kong Manager 配置，验证当前模式下确实这么配置无效
admin_gui_listen = 0.0.0.0:80, 0.0.0.0:443 ssl
admin_gui_url = http://<my_ip>:80/manager
admin_gui_path = /manager
```


我们修改 kong.yml services部分如下，让请求代理到字节跳动首页

```
services:
- name: example-service
  url: https://www.bytedance.com
  # Entities can store tags as metadata
  tags:
  - example
  # Entities that have a foreign-key relationship can be nested:
  routes:
  - name: example-route
    paths:
    - /
```


## 运行

{{<cmd>}}
# 启动前先检测配置是个好习惯
kong check /etc/kong/kong.conf

# 启动 kong
kong start -c /etc/kong/kong.conf
{{</cmd>}}

这时候我们请求 *我自己的IP地址* 看看效果, 果然去了字节首页。

{{< figureCupper
img="figure3-kong-demo.png"
caption="Kong Gateway 代理请求的效果"
command="Resize"
options="1080x" >}}

恭喜你🎉，又掌握了一门技术😂！


## 插件开发

Kong Gateway 提供了一套PDK（Plugin Development Kit），按照[官方指导](https://docs.konghq.com/gateway/3.4.x/plugin-development/)可以开发一些自己的插件。当然需要有一定的Lua语言基础和学习luarocks包管理器的使用。

之前的工作中我曾经开发维护过两个插件，github链接放下边了，供学习交流。

* 多租户路由插件 [route-by-id ](https://github.com/op-y/route-by-id)
* 根据referer限流插件 [referer-limiting](https://github.com/op-y/referer-limiting)


# 参考

* Kong Gateway [官方网站](https://docs.konghq.com/gateway/latest/)

再多的参考资料都不如自己动手实践，共勉！


