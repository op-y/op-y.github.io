---
title: "Traefik Ingress Controller"
date: 2023-09-07T11:07:00+08:00
tags: [Traefik, kubernetes]
toc: true
---

## 引子

之前已经回顾了 OpenResty 和 Kong 有关的内容（当然 Nginx 相关内容默认是必知必会了），这两天隐隐约约觉得还差了一点什么东西，昨晚想起来了，是 Kubernetes 边缘路由（Edge Router），这里和IP层边缘路由协议 BGP 之类的概念做一些区别啊。

什么是边缘路由呢，直观地说，就是将来自 Kubernetes 集群外部请求代理到集群内部服务的反向代理（通常它会以DaemonSet/Deployment 的形式部署在 Kubernetes 集群中，通过 NodePort 方式将 Service 暴露到集群外部以接受请求）。它主要解决了服务在 Kubernetes 集群内部 IP 不固定会随时变化，需要动态更新路由信息的问题。

在 Kubernetes 中把进入流量的描述称为一种 Ingress 资源，边缘路由通常对 Ingress、Service、Endpoint 对象进行监视实时调整自己代理流量的行为，实现对进入请求的正确代理。所以这种边缘路由的服务也被称作 Ingress Controller。

当下Ingress Controller的实现有很多种，之前我们使用的是 Traefik2，所以接下来回顾一下 Traefik。

## Traefik 是什么？

其实在引言部分已经说的很清楚了。

Traefik [官方介绍](https://doc.traefik.io/traefik/)

{{< figureCupper
img="figure1-traefik-logo.png"
caption="Traefik 的 Logo 是一只交通管理员鼹鼠（非常的形象）"
command="Resize"
options="1080x" >}}

Traefik 官方给的架构图如下，很直观，都不用过多解释就能看懂它在干什么。

{{< figureCupper
img="figure2-traefik-arch.png"
caption="Traefik 架构图"
command="Resize"
options="1080x" >}}


## Traefik 安装&测试

我们的运行环境如下

* kuberntes 1.27.2

这里出现了 **Kubernetes**，关于 kubernetes（或者简称k8s）又是一个天坑，这里不打算多说。等有机会...

Traefik 目前有 1.x/2.x/3.x 几个版本（3.0目前还在rc阶段），不同的大版本之间存在不兼容的变更。从1.x到2.x的迭代中，Traefik从使用k8s自带的 Ingress 到使用了很多CRD（Custom Resource Definition）包括自己定义的 IngressRoute。为了更接近当年我们的使用，我使用了2.10版本但是使用 Ingress 作为路由的 provider。

以下是根据官方说明创建各种资源的yaml文件，部分根据我的测试环境有修改，我做注释说明。


创建账户 给traefik用。
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-account
```

创建集群角色 给traefik用。
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-role
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses/status
    verbs:
      - update
```

集群角色与账号绑定。
```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-role
subjects:
  - kind: ServiceAccount
    name: traefik-account
    namespace: default # Using "default" because we did not specify a namespace when creating the ClusterAccount.
```

创建一个traefik daemonset 代理入流量。
```
kind: DaemonSet # 官方说明使用的Deployment这里为了一个node固定启动一个Pod我改成了DaemonSet
apiVersion: apps/v1
metadata:
  name: traefik-daemonset
  labels:
    app: traefik
spec:
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-account
      nodeSelector:
        traefik: "true"
      containers:
        - name: traefik
          image: traefik:v2.10
          args:
            - --api.insecure # 启动非安全http 80端口代理
            - --providers.kubernetesingress # 使用ingress作为provider
          ports:
            - name: web
              containerPort: 80
            - name: dashboard
              containerPort: 8080
```

创建traefik service。
```
apiVersion: v1
kind: Service
metadata:
  name: traefik-dashboard-service
spec:
  type: NodePort # 官方说明使用的LoadBalancer这里我使用的NodePort 使用主机端口 下同
  ports:
    - port: 8080
      targetPort: dashboard
      nodePort: 30288
  selector:
    app: traefik
---
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-service
spec:
  type: NodePort
  ports:
    - targetPort: web
      port: 80
      nodePort: 30280
  selector:
    app: traefik
```

为了测试创建一个http应用 whoami，它的响应就是把请求直接返回。
```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: whoami
  labels:
    app: whoami
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - name: web
              containerPort: 80
```

创建 whoami 的 service。
```
apiVersion: v1
kind: Service
metadata:
  name: whoami
spec:
  ports:
    - name: web
      port: 80
      targetPort: web

  selector:
    app: whoami
```

创建 ingress， 这个是重点， 这个ingress 提供给traefik 路由信息 能让请求代理到 whoami上。
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: whoami
            port:
              name: web
```

将以上配置应用到 k8s 后，我们看一下集群中 service 情况。

{{< figureCupper
img="figure3-k8s-services.png"
caption="traefik 和 whoami 的service"
command="Resize"
options="1080x" >}}

因为 traefik 使用节点端口 30280 接收请求，我们再将 nginx 从公网接收的请求代理到这个节点的 30280 端口查看效果。

{{< figureCupper
img="figure4-traefik-proxy.png"
caption="traefik 代理外部请求到 whoami"
command="Resize"
options="1080x" >}}

可以看到流量被代理到k8s集群内部whoami服务上处理并返回响应了。

## 还能怎么干？

一边做一边浮想连篇，我想起了在 Traefik 出现之前我们是怎么处理这种问题的...

那个时候我们使用了一种 nginx + [confd](https://github.com/kelseyhightower/confd) 的方式来实现边缘路由的效果。

关键在于两点：
* nginx 所在机器需要部署 flanneld（我们k8s使用的CNI网络插件），这样流量才能从nginx 进入集群；
* 使用 confd 这个工具去 watch k8s etcd（k8s的数据都保存在这里）变化，然后通过一个外部脚本渲染模板生成对应服务的 nginx 配置，然后执行 nginx reload 操作，这样 nginx 上的代理配置就能同步 k8s 中服务的变化。

非常的粗糙，但是很管用，不知道当时是哪位设计的这个方案，只要脑洞大，没有解决不了的问题！


## 参考

* Traefik [安装说明](https://docs.konghq.com/gateway/latest/)

再多的参考资料都不如自己动手实践，共勉！


