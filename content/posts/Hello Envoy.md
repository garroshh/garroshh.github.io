---
title: "Hello Envoy"
date: 2020-04-30T18:18:41+08:00
draft: false

tags: ["Envoy", "实验"]
categories: ["Envoy"]
---

# 目标

- 配置Envoy代理转发流量到外部网址

- 通过控制流量目的地址，演示基于路径的路由

# 准备 Envoy 的配置文件

如下所示，文件名 baidu_com_proxy.v2.yaml。

```yaml
admin: ##Envoy提供的管理视图，可以查看配置，统计信息，日志和其他内部Envoy数据
  access_log_path: /tmp/admin_access.log
  address:
    socket_address:     
      protocol: TCP
      address: 0.0.0.0
      port_value: 9901
static_resources: #静态API配置
  listeners:  #监听器配置
  - name: listener_0
    address:
      socket_address:
        protocol: TCP
        address: 0.0.0.0
        port_value: 10000
    filter_chains:  #过滤器链配置，路由所有流量到www.baidu.com
    - filters:
      - name: envoy.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          stat_prefix: ingress_http  ##连接管理器发出统计信息时使用的可读性前缀
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]  ##虚拟主机配置
              routes:  ##如果虚拟主机匹配，则检查路由
              - match:
                  prefix: "/"
                route:
                  host_rewrite: www.baidu.com  ##更改HTTP请求的入站主机头
                  cluster: service_baidu  ##处理匹配的路由的集群名称
          http_filters:
          - name: envoy.router  ##该过滤器允许Envoy在处理请求时适应和修改请求
  clusters:  ##当请求与过滤器匹配时，该请求将传递到集群。
  - name: service_baidu
    connect_timeout: 0.25s
    type: LOGICAL_DNS
    # Comment out the following line to test on v6 networks
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: service_baidu
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: www.baidu.com
                port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.api.v2.auth.UpstreamTlsContext
        sni: www.baidu.com
```

# 启动 Envoy

挂载配置文件 baidu_com_proxy.v2.yaml 到 /etc/envoy/envoy.yaml。

```bash
docker run --name=proxy-with-admin -d -p 9901:9901 -p 10000:10000 -v $(pwd)/baidu_com_proxy.v2.yaml:/etc/envoy/envoy.yaml envoyproxy/envoy:latest
```

# 测试

- curl localhost:10000

![image2019-12-26_16-42-49](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-26_16-42-49.png)

- 管理控制台访问地址：http://172.16.99.100:9901/ （注：后续测试宿主机地址都为172.16.99.100）

![image2019-12-26_16-51-46](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-26_16-51-46.png)