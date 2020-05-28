---
title: "Envoy VS Nginx"
date: 2020-05-06T10:38:06+08:00
draft: false

tags: ["Envoy", "实验"]
categories: ["Envoy"]
---

# 目标

对比Envoy 和 Nginx 配置

# Nginx配置

Nginx 配置三要素：

- 全局配置 Nginx 服务器，日志记录结构，Gzip 功能等
- 配置 Nginx 在 10000 端口接收对 one.example.com 主机的请求
- 配置如何处理到达不同 URL 的流量的目标地址

```nginx
user  www www;
pid /var/run/nginx.pid;
worker_processes  2;
 
events {
  worker_connections   2000;
}
 
http {
  gzip on;
  gzip_min_length  1100;
  gzip_buffers     4 8k;
  gzip_types       text/plain;
 
  log_format main      '$remote_addr - $remote_user [$time_local]  '
    '"$request" $status $bytes_sent '
    '"$http_referer" "$http_user_agent" '
    '"$gzip_ratio"';
 
  log_format download  '$remote_addr - $remote_user [$time_local]  '
    '"$request" $status $bytes_sent '
    '"$http_referer" "$http_user_agent" '
    '"$http_range" "$sent_http_content_range"';
 
 
  upstream targetCluster {
    172.18.0.3:80;
    172.18.0.4:80;
  }
 
  server {
    listen        10000;
    server_name   one.example.com  www.one.example.com;
 
    access_log   /var/log/nginx.access_log  main;
    error_log  /var/log/nginx.error_log  info;
 
    location / {
      proxy_pass         http://targetCluster/;
      proxy_redirect     off;
 
      proxy_set_header   Host             $host;
      proxy_set_header   X-Real-IP        $remote_addr;
    }
  }
}
```

# Envoy 配置

Envoy 配置四要素：

- Listeners：定义了 Envoy 如何接收传入的请求。当前，Envoy 仅支持基于 TCP 的监听器，建立连接后，Envoy 会将流量传递到一组过滤器进行处理
- Filters：处理入站和出站数据。比如Gzip过滤器，该过滤器会将数据发送到客户端前压缩数据
- Routers: 将流量转发到所需的目的地（定义为集群）
- Clusters: 定义流量的目的端点和配置设置，如负载均衡策略

```yaml
static_resources: #静态API配置
  listeners:  #监听器配置
  - name: listener_0
    address:
      socket_address:
        protocol: TCP
        address: 0.0.0.0
        port_value: 10000
    filter_chains:  #过滤器链配置
    - filters:
      - name: envoy.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          stat_prefix: ingress_http  ##连接管理器发出统计信息时使用的可读性前缀
          access_log:
          - name: envoy.file_access_log
            config:
              path: "/dev/stdout"
              format: '[%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%" %RESPONSE_CODE% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% "%REQ(X-REQUEST-ID)%" "%REQ(:AUTHORITY)%" "%UPSTREAM_HOST%"\n'
              #json_format: {"protocol": "%PROTOCOL%", "duration": "%DURATION%", "request_method": "%REQ(:METHOD)%"}
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              #domains: ["*"]  ##虚拟主机配置
              domains:
                - "one.example.com"
                - "www.one.example.com"
              routes:  ##如果虚拟主机匹配，则检查路由
              - match:
                  prefix: "/"
                route:
                  cluster: targetCluster  ##处理匹配的路由的集群名称
          http_filters:
          - name: envoy.router  ##该过滤器允许Envoy在处理请求时适应和修改请求
  clusters:  ##当请求与过滤器匹配时，该请求将传递到集群。
  - name: targetCluster
    connect_timeout: 0.25s
    type: STRICT_DNS
    # Comment out the following line to test on v6 networks
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    hosts: [
      { socket_address: { address: 172.17.0.4, port_value: 80 }},
      { socket_address: { address: 172.17.0.5, port_value: 80 }}
    ]
```

# 测试：通过 Envoy 流量转发，轮询的负载均衡到后端两个服务

按照如上Envoy配置，启动 Envoy 容器。后端两个服务未启动时，返回503，属于正常现象。

![image2019-12-27_15-3-22](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-27_15-3-22.png)

启动后端两个服务

![image2019-12-27_15-7-6](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-27_15-7-6.png)

通过 Envoy 流量转发，轮询的负载均衡到后端两个服务

![image2019-12-27_15-9-9](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-27_15-9-9.png)