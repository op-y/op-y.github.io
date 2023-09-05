---
title: "OpenResty上手"
date: 2023-09-05T14:42:00+08:00
tags: [nginx,openresty,lua]
toc: false
---

# 引子

没有想过要针对 Nginx、OpenResty、Kong 这类接入层中间件和系统专门写一些东西。最近因为DDDD的原因，需要回顾一下之前使用过的一些接入层开源系统以及做过的一些事情，所以简单的重新操作一下，做个记录。


# OpenResty 是什么？

官方首页上有比较[官方的介绍](http://openresty.org/cn/)

比较简单的理解：OpenResty 就是一个加强版的 Nginx，具体的加强之处就在于它给 Nginx 开发了一个 [lua-nginx-module](https://github.com/openresty/lua-nginx-module)，这玩意儿在Nginx中直接集成了一个 Lua 解释器，让 Nginx 在处理请求的过程中可以执行内嵌的 Lua 脚本从而动态的实现一些 Nginx 原本很难做到的功能。

此外 OpenResty 官方还开发了很多可以直接使用的[组件](http://openresty.org/cn/components.html)，用户可以直接拿来用；OpenResty 社区还有很多[第三方的组件](https://opm.openresty.org)，方便用户开发自己的功能。


# OpenResty 使用

## 安装

常见的操作系统都可以通过包管理工具直接安装对应的二进制版本。为了~~装逼~~有点参与感，我们还是源码编译安装。

我们的运行环境如下

* kernel: 5.10.0-15-amd64
* 系统发行版：Debian GNU/Linux 11

当前 OpenResty 的最新版本是 openresty-1.21.4.2，我们就选择整个版本，开始操作

```
cd ~
mkdir download && cd download # 创建一个工作目录
wget https://openresty.org/download/openresty-1.21.4.2.tar.gz # 从官网下载源码
tar -zxvf openresty-1.21.4.2.tar.gz # 直接解压到当前目录

apt-get install libpcre3-dev libssl-dev perl make build-essential curl # 安装依赖

# 编译过程基本与nginx相同
# 我们指定了安装目录和一些需要安装的功能选项
cd openresty-1.21.4.2
./configure --prefix=/home/openresty --with-luajit --with-http_iconv_module
make
make install
```

{{< figureCupper
img="figure1-openresty-dir.png"
caption="openresty目录内容"
command="Resize"
options="1080x" >}}


## 运行

俗话说写出一个 `hello world!` 你就掌握了一门新的编程语言。话不多说，开撸！

我们按照官方demo修改 `nginx.conf` 如下，这里只给出了了 http server 配置中 location 的配置。这里的关键就是 **content_by_lua_block** 这个指令了。使用 Nginx 的知道在 Nginx 中是没有这个指令的，这个指令来自于 lua-nginx-module。它具体的行为是在 Nginx 处理请求的11个（如果我没记错的话）阶段中生成 response 的阶段中使用 Lua 脚本生成了内容。具体 Nginx 是如何处理请求的那又是一个更大的主题了。

```
location / {
    default_type text/html;
    content_by_lua_block {
		ngx.say("<h1>Hello, World!</h1>")
	}
}

```

现在可以执行命令启动 OpenResty 了

```
# 启动前先检测配置是个好习惯
nginx/sbin/nginx -t
# 启动 openresty，其实就是启动nginx
nginx/sbin/nginx

```

这时候我们可以看看效果了

{{< figureCupper
img="figure2-openresty-helloworld.png"
caption="openresty 启动演示"
command="Resize"
options="1080x" >}}

恭喜你🎉，掌握了 OpenResty 的使用😂！


## 功能使用

现在真的没有时间一个一个库的演示 OpenResty的功能了...我印象中之前业务上使用OpenResty主要处理了两个问题：

* 业务框架上一个Header的大小写问题到导致功能无法使用，通过Lua脚本进行的转换;
* 对特定Host的请求动态路由到不同的upstream。


# 参考

* [OpenResty官方网站](https://openresty.org/cn/)
* [《OpenResty 完全开发指南》](https://item.jd.com/12414779.html) 
* 极客时间课程 [《OpenResty 从入门到实战》](https://time.geekbang.org/column/intro/100028301)

再多的参考资料都不如自己动手实践，共勉！


