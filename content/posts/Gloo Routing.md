---
title: "Gloo Routing"
date: 2020-05-14T14:32:33+08:00
draft: false

tags: ["Gloo", "原理"]
categories: ["Gloo"]
---

![image-20200514143331031](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514143331031.png)

3个层：网关侦听器，虚拟服务和上游。通常，我们与虚拟服务进行交互，从而可以配置希望在网关上公开的API的详细信息以及如何路由到后端。

上游代表那些后端。网关对象可帮助我们控制侦听器的传入流量。

# 1. 部署 Pet Store

```bash
kubectl apply -f https://raw.githubusercontent.com/solo-io/gloo/v1.2.9/example/petstore/petstore.yaml
```

验证 Pod 和 Svc

![image-20200514143346589](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514143346589.png)

验证 Pet Store 上游服务

Gloo 服务发现会监视添加到 kubernetes 集群的新服务。当 petstore 服务被创建后，Gloo 会自动为 petstore 服务创建一个 Upstream。如果一切正常，

Upstream STATUS 为 Accepted。

![image-20200514143403050](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514143403050.png)

# 2. 标记命名空间

查看 upstream yaml

![image-20200514143416900](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514143416900.png)

默认情况下，上游创建的过程非常简单。它代表特定的 kubernetes 服务。但是，petstore应用程序是一个 swagger 的服务。Gloo可以发现这种 swagger 规格，但是默认情况下，Gloo的功能发现功能已关闭，以提高性能。

要在我们的petstore上启用功能发现服务（fds），我们需要标记名称空间。

```bash
kubectl label namespace default discovery.solo.io/function_discovery=enabled
```

![image-20200514143432499](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514143432499.png)

![image-20200514143452411](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514143452411.png)

# 3. 配置路由

即使已创建上游，在我们向虚拟服务添加一些路由规则之前，Gloo仍不会将流量路由到该路由。现在，我们使用glooctl通过--prefix-rewrite标志为此上游创建一条基本路由，以在传入请求中重写路径以匹配我们的petstore应用程序期望的路径。

```bash
glooctl add route \ --path-exact /all-pets \ --dest-name default-petstore-8080 \ --prefix-rewrite /api/pets
```

![image-20200514143507796](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514143507796.png)

petstore虚拟服务的初始状态为Pending。几秒钟后，应更改为“已接受”。让我们通过使用glooctl检索虚拟服务来验证这一点。

```shell
glooctl get virtualservice
```

![image-20200514143520981](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514143520981.png)

路由与Gloo中的虚拟服务相关联。在上一步中创建路由时，我们没有提供虚拟服务，因此Gloo创建了一个名为default的虚拟服务并添加了路线。

```shell
glooctl get virtualservice --output kube-yaml
```

![image-20200514143535300](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514143535300.png)

创建虚拟服务后，Gloo立即更新代理配置。由于此虚拟服务的状态为“已接受”，因此我们知道此路由现在处于活动状态。

至此，我们有了一个带有路由规则的虚拟服务，该路由规则将/ all-pets路径上的流量发送到/ api / pets路径上的上游petstore。

# 4. 测试

```shell
curl $(glooctl proxy url)/all-pets
```