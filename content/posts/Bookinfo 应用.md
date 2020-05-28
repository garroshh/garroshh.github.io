---
title: "Bookinfo 应用"
date: 2020-05-11T14:59:51+08:00
draft: false

tags: ["Istio", "实验"]
categories: ["Istio"]
---

部署一个样例应用，它由四个单独的微服务构成，用来演示多种 Istio 特性。这个应用模仿在线书店的一个分类，显示一本书的信息。页面上会显示一本书的描述，书籍的细节，以及关于这本书的一些评论。

Bookinfo 应用分为四个单独的微服务：

- productpage：productpage 微服务会调用 details 和 reviews 两个微服务，用来生成页面。
- details：这个微服务包含了书籍的信息。
- reviews：这个微服务包含了书籍相关的评论。它还会调用 ratings 微服务。
- ratings：ratings 微服务中包含了由书籍评价组成的评级信息。

reviews 微服务有 3 个版本：

- v1 版本不会调用 ratings 服务。
- v2 版本会调用 ratings 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
- v3 版本会调用 ratings 服务，并使用 1 到 5 个红色星形图标来显示评分信息。

下图展示了这个应用的端到端架构。

![image-20200511162710192](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511162710192.png)

Bookinfo 是一个异构应用，几个微服务是由不同的语言编写的。这些服务对 Istio 并无依赖，但是构成了一个有代表性的服务网格的例子：它由多个服务、多个语言构成，并且 reviews 服务具有多个版本。

### 开始之前

如果还没开始，首先要遵循 [Istio安装](https://dwiki.daocloud.io/pages/viewpage.action?pageId=45449653) 的指导，根据所在平台完成 Istio 的部署工作。

### 部署应用

要在 Istio 中运行这一应用，无需对应用自身做出任何改变。我们只要简单的在 Istio 环境中对服务进行配置和运行，具体一点说就是把 Envoy sidecar 注入到每个服务之中。这个过程所需的具体命令和配置方法由运行时环境决定，而部署结果较为一致，如下图所示：

![image-20200511162653451](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511162653451.png)

所有的微服务都和 Envoy sidecar 集成在一起，被集成服务所有的出入流量都被 sidecar 所劫持，这样就为外部控制准备了所需的 Hook，然后就可以利用 Istio 控制平面为应用提供服务路由、遥测数据收集以及策略实施等功能。

### 如果在 Kubernetes 中运行

1.进入 Istio 安装目录。

Istio 默认启用自动 Sidecar 注入，为 default 命名空间打上标签 istio-injection=enabled。

```bash
$ kubectl label namespace default istio-injection=enabled
```

2.使用 kubectl 部署简单的服务

```bash
$ kubectl apply -f samples``/bookinfo/platform/kube/bookinfo``.yaml
```

3.（可选）如果在安装过程中禁用了自动 Sidecar 注入并依赖 手工 Sidecar 注入, istioctl kube-inject 命令用于在在部署应用之前修改 bookinfo.yaml。 

```bash
$ kubectl apply -f <(istioctl kube-inject -f samples``/bookinfo/platform/kube/bookinfo``.yaml)
```

上面的命令会启动 bookinfo` 应用程序架构图中全部的四个服务，其中也包括了 reviews 服务的三个版本（v1、v2 以及 v3）

在实际部署中，微服务版本的启动过程需要持续一段时间，并不是同时完成的。

4.确认所有的服务和 Pod 都已经正确的定义和启动：

```bash
$ kubectl get services
$ kubectl get pods
```

5.要确认 Bookinfo 应用程序正在运行，请通过某个 pod 中的 curl 命令向其发送请求，例如来自 ratings：

```bash
$ kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
```

#### 确定 Ingress 的 IP 和端口

现在 Bookinfo 服务启动并运行中，你需要使应用程序可以从外部访问 Kubernetes 集群，例如使用浏览器。一个 Istio Gateway 应用到了目标中。

1.为应用程序定义入口网关：

```bash
$ kubectl apply -f samples``/bookinfo/networking/bookinfo-gateway``.yaml
```

2.确认网关创建完成：

```bash
$ kubectl get gateway
```

3.根据部署istio时候istio-ingressgateway组件的http NodePort，访问bookinfo服务，如

http://172.16.99.181:8500/productpage

### 确认应用在运行中

可以用 curl 命令来确认 Bookinfo 应用的运行情况：

```bash
$ curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"
```

还可以用浏览器打开网址 http://$GATEWAY_URL/productpage，来浏览应用的 Web 页面。如果刷新几次应用的页面，就会看到 productpage 页面中会随机展示 reviewe 服务的不同版本的效果（红色、黑色的星形或者没有显示）。reviews 服务出现这种情况是因为我们还没有使用 Istio 来控制版本的路由。

### 清理

结束对 Bookinfo 示例应用的体验之后，就可以使用下面的命令来完成应用的删除和清理了。

1.删除路由规则，并终结应用的 Pod

```bash
$ samples/bookinfo/platform/kube/cleanup.sh
```

2.确认应用已经关停

```bash
$ kubectl get virtualservices #-- there should be no virtual services
 
$ kubectl get destinationrules #-- there should be no destination rules
 
$ kubectl get gateway #-- there should be no gateway
 
$ kubectl get pods #-- the Bookinfo pods should be deleted
```