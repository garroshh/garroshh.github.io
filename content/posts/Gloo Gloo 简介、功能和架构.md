---
title: "Gloo Gloo 简介、功能和架构"
date: 2020-05-14T15:26:31+08:00
draft: false

tags: ["Gloo", "原理"]
categories: ["Gloo"]
---

# 1. Gloo 简介

Gloo is a feature-rich, Kubernetes-native ingress controller, and next-generation API gateway. Gloo is uniquely designed to support hybrid applications, in which multiple technologies, architectures, protocols, and clouds can coexist.

![image-20200514152710904](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514152710904.png)

- Gloo is exceptional in its function-level routing;
- its support for legacy apps, microservices and serverless;
- its discovery capabilities; 
- its numerous features;
- its tight integration with leading open-source projects.

# 2. Gloo 功能

| connect                                                      | secure                                                       | control                                                      |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| All Workload TypesAuto Service DiscoveryHTTP RoutingTCP ProxygRPC WebCORSKubernetes ServicesConsul ServicesServerless FunctionsConfiguration ValidationRequest / Response TransformationService Mesh integration | 社区版：TLSHashicorp Vault SecretsLet’s EncryptCustom Authentication (DIY)企业版：Data Loss PreventionWeb App Firewall (WAF)Basic AuthenticationAPI KeyJSON Web Token (JWT)LDAP SupportOAuth / OIDCOpen Policy Agent | 社区版：Admin Dashboard (Read Only)Role DelegationAccess Logging & Usage StatsPrometheus and GrafanaTracingCircuit BreakingRetriesTimeoutsTraffic ShiftingTraffic ShadowingRate Limiting (DIY)企业版：Admin Dashboard (Full Access)Advanced Rate Limiting |

# 3. Gloo 架构

![image-20200514152732115](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514152732115.png)

**Secret Watcher**

The *Secret Watcher* watches a secret store for updates to secrets (which are required for certain plugins such as the AWS Lambda Plugin . The secret storage could be using secrets management in Kubernetes, HashiCorp Vault, or some other secure secret storage system.

**Config Watcher**

The *Config Watcher* watches the storage layer for updates to user configuration objects, such as Upstreams and Virtual Services. The storage layer could be a custom resource in Kubernetes or an key/value entry in HashiCorp Consul.

**Reporter**

The *Reporter* receives a validation report for every Upstream and Virtual Service processed by the translator. Any invalid config objects are reported back to the user through the storage layer. Invalid objects are marked as *Rejected* with detailed error messages describing mistakes found in the configuration.

**Endpoint Discovery**

*Endpoint Discovery* watches service registries such as Kubernetes, Cloud Foundry, and Consul for IPs associated with services. Endpoint Discovery is plugin-specific, so each endpoint type will require a plug-in that supports the discovery functionality. For example, the Kubernetes Plugin runs its own Endpoint Discovery goroutine.

**Gloo Translator**

The *Gloo Translator* receives snapshots of the entire state, composed of the following configuration data:

- Artifacts
- Endpoints
- Proxies
- Upstreams
- UpstreamGroups
- Secrets
- AuthConfigs

![image-20200514152752413](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514152752413.png)

The translator takes all of this information and initiates a new *translation loop* with the end goal of creating a new Envoy xDS Snapshot.

**xDS Server**

The final snapshot is passed to the *xDS Server*, which notifies Envoy of a successful config update, updating the Envoy cluster with a new configuration to match the desired state set expressed by Gloo.

**Discovery Architecture**

Gloo is supported by a suite of optional discovery services that automatically discover and configure Gloo with Upstreams and functions to simplify routing for users and self-service.

![image-20200514152815821](https://cdn.jsdelivr.net/gh/garroshh/figurebed/2020/image-20200514152815821.png)

Discovery services act as automated Gloo clients, automatically populating the storage layer with Upstreams and functions to facilitate easy routing for users. Discovery is optional, but when enabled, it will attempt to discover available Upstreams and functions.

The following discovery methods are currently supported:

- Kubernetes Service-Based Upstream Discovery
- AWS Lambda-Based Function Discovery
- Google Cloud Function-Based Function Discovery
- OpenAPI-Based Function Discovery
- Istio-Based Route Rule Discovery (Experimental)



参考：

Announcing Gloo: The Function Gateway

https://medium.com/solo-io/announcing-gloo-the-function-gateway-3f0860ef6600