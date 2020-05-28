---
title: "Gloo 分析"
date: 2020-05-14T15:21:56+08:00
draft: false

tags: ["Gloo", "原理"]
categories: ["Gloo"]
---

以下为个人理解，仅供参考！

# Gloo Envoy Upgrades

As we ship an extended version of Envoy, it is very important for us to stay close to upstream Envoy. To achieve this, our repository structure and CI closely resembles those of Envoy.

Envoy master branch is always considered RC quality. We therefore make sure frequently that Gloo can be built with the latest master code. This ensures that we always have the latest Envoy features and optimizations, and can respond quickly if a security update is needed. Building Envoy on the 32-core CI we use takes about 10 minutes.

![image-20200514152256884](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514152256884.png)      

![image-20200514152319241](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514152319241.png)



# servless 支持

![image-20200514152341972](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514152341972.png)

In order to achieve Knative scale-from-zero, we use a Mixer [out-of-process adapter](https://github.com/istio/istio/wiki/Mixer-Out-Of-Process-Adapter-Dev-Guide) to call the Autoscaler. Out-of-process adapters for Mixer allow developers to use any programming language and to build and maintain your extension as a stand-alone program without the need to build the Istio proxy. 好复杂的感觉。



![image-20200514152402705](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514152402705.png)

只需创建一个 Aws 类型的 Upstream，so easy 的感觉。



# Api 组合

Sqoop (formerly QLoo) is a GraphQL Server built on top of Gloo and the Envoy Proxy.

Sqoop leverages Gloo's function registry and Envoy's advanced HTTP routing features to provide a GraphQL frontend for REST/gRPC applications and serverless functions. Sqoop routes requests to data sources via Envoy, leveraging Envoy HTTP filters for security, load balancing, and more.

![image-20200514152428303](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514152428303.png)

个人觉得基于此可做Api组合的事情，一方面项目大起来之后，减少聚合层做的工作，另一方面，减少后端接口的数量。

Istio 暂无，也不会有，关注点不一样，是东西流量。



# 灰度发布

Istio 做法：哪一个服务要灰度，写哪一个服务的 VirtualService，个人觉得，灰度的配置管理，当前和流控，路由等的配置都集中在了目的服务的vs配置里，不便于管理。另一方面，服务多了之后，因为是为目标服务来做配置，这些配置文件如何管理。如下的这些灰度规则，个人

觉得都有点晕了。

![image-20200514152442003](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514152442003.png)

![image-20200514152459187](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514152459187.png)

Gloo 做法, 通过Upstream Group 统一起来，So Easy，至少视觉效果是这样

![image-20200514152519259](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514152519259.png)





参考：

https://medium.com/solo-io/building-a-control-plane-for-envoy-7524ceb09876

https://github.com/solo-io/sqoop

https://istio.io/blog/2019/knative-activator-adapter/