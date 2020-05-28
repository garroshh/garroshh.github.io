---
title: "XDS API"
date: 2020-05-11T14:41:33+08:00
draft: false

tags: ["Envoy", "原理"]
categories: ["Envoy"]
---

- [Listeners Discovery Service API — LDS](https://www.envoyproxy.io/docs/envoy/v1.9.0/configuration/listeners/lds#config-listeners-lds) to publish ports on which to listen for traffic
- [Endpoints Discovery Service API- EDS](https://www.envoyproxy.io/docs/envoy/v1.9.0/api-v2/api/v2/eds.proto#envoy-api-file-envoy-api-v2-eds-proto) for service discovery,
- [Routes Discovery Service API- RDS](https://www.envoyproxy.io/docs/envoy/v1.9.0/configuration/http_conn_man/rds#config-http-conn-man-rds) for traffic routing decisions
- [Clusters Discovery Service- CDS](https://www.envoyproxy.io/docs/envoy/v1.9.0/configuration/cluster_manager/cds#config-cluster-manager-cds) for backend services to which we can route traffic
- [Secrets Discovery Service — SDS](https://www.envoyproxy.io/docs/envoy/v1.9.0/configuration/secret) for distributing secrets (certificates and keys)

![image-20200511143655362](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511143655362.png)

![image-20200511143738770](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511143738770.png)



xDS REST and gRPC protocol：https://github.com/envoyproxy/data-plane-api/blob/master/xds_protocol.rst

xDS 控制面开发 SDK：https://github.com/envoyproxy/go-control-plane



参考：

Guidance for Building a Control Plane to Manage Envoy Proxy at the edge, as a gateway, or in a mesh

https://medium.com/solo-io/guidance-for-building-a-control-plane-to-manage-envoy-proxy-at-the-edge-as-a-gateway-or-in-a-mesh-badb6c36a2af

Guidance for Building a Control Plane for Envoy Part 5: Deployment Tradeoffs

https://medium.com/solo-io/guidance-for-building-a-control-plane-for-envoy-part-5-deployment-tradeoffs-a6ef55c06327