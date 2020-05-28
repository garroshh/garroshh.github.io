---
title: "实现Metrics和Tracing的能力"
date: 2020-05-06T15:54:52+08:00
draft: false

tags: ["Envoy", "实验"]
categories: ["Envoy"]
---

# 目标

- 暴露 Envoy 的 Metrics 到 Prometheus
- 使用 Prometheus 数据源在 Grafana 展示
- 配置 Envoy 发送 traces 到 Jaeger

# 启动上游服务

```bash
docker run -d katacoda/docker-http-server:healthy
docker run -d katacoda/docker-http-server:healthy
```

# 启动 Envoy

```yaml
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 }
 
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          codec_type: auto
          stat_prefix: ingress_http
          generate_request_id: true
          tracing:
            operation_name: egress
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
                - "*"
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: targetCluster
          http_filters:
          - name: envoy.router
  clusters:
  - name: targetCluster
    connect_timeout: 0.25s
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    hosts: [
      { socket_address: { address: 172.17.0.2, port_value: 80 }},
      { socket_address: { address: 172.17.0.3, port_value: 80 }}
    ]
    health_checks:
      - timeout: 1s
        interval: 10s
        interval_jitter: 1s
        unhealthy_threshold: 6
        healthy_threshold: 1
        http_health_check:
          path: "/health"
  - name: jaeger
    connect_timeout: 1s
    type: strict_dns
    lb_policy: round_robin
    load_assignment:
      cluster_name: jaeger
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 172.18.0.6
                port_value: 9411
tracing:
    http:
      name: envoy.zipkin
      config:
        collector_cluster: jaeger
        collector_endpoint: "/api/v1/spans"
        shared_span_context: false
```

prometheus.yml

```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s
 
scrape_configs:
  - job_name: 'envoy'
    metrics_path: /stats/prometheus
    static_configs:
      - targets: ['172.18.0.2:9901']
        labels:
          group: 'envoy'
```



//todo