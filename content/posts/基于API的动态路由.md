---
title: "基于API的动态路由"
date: 2020-05-06T15:27:22+08:00
draft: false

tags: ["Envoy", "实验"]
categories: ["Envoy"]
---

# 目标

- 配置EDS
- 使用 REST API 为集群添加端点
- 使用 REST API 删除端点

当上游被定义在配置中，Envoy 需要知道如何解析集群成员，这时需要用到服务发现。EDS 是一个运行在gRPC or REST-JSON API 。

服务端的xDS服务，Envoy 用该服务来获取集群成员。Envoy项目在 [Java](https://github.com/envoyproxy/java-control-plane) 和 [Go](https://github.com/envoyproxy/go-control-plane) 中都提供了EDS和其他发现服务的参考gRPC实现。

在接下来的步骤中，我们将更改配置以使用Endpoint Discovery Service（EDS），从而允许基于来自REST-JSON API的数据来动态添加节点。

# 启动 Envoy

envoy.yaml

```yaml
admin:
  access_log_path: /dev/null
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9000
 
node:
  cluster: mycluster
  id: test-id
 
static_resources:
  listeners:
  - name: listener_0
 
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 }
 
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: targetCluster }
          http_filters:
          - name: envoy.router
 
  clusters:
  - name: targetCluster
    type: EDS
    connect_timeout: 0.25s
    eds_cluster_config:
      service_name: myservice
      eds_config:
        api_config_source:
          api_type: REST
          cluster_names: [eds_cluster]
          refresh_delay: 5s
 
  - name: eds_cluster
    type: STATIC
    connect_timeout: 0.25s
    hosts: [{ socket_address: { address: 172.18.0.4, port_value: 8080 }}]
```

启动 Envoy：

```bash
docker run --name=api-eds -d -p 80:10000 -p 9901:9901 -v /root/envoy-1.12.2/examples/api-bases-dr/:/etc/envoy envoyproxy/envoy
```

# 启动上游服务

```bash
docker run -p 8081:8081 -d -e EDS_SERVER_PORT='8081' katacoda/docker-http-server:v4
```

验证服务：curl [http://localhost:8081](http://localhost:8081/) -i

![image2019-12-30_19-44-1](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-30_19-44-1.png)

# 启动 EDS 服务

```bash
docker run -p 8080:8080 -d katacoda/eds_server
```

![image2019-12-30_19-49-29](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-30_19-49-29.png)

测试：上游服务未注册到 EDS 服务时

![image2019-12-30_19-53-14](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-30_19-53-14.png)

注册上游服务到 EDS 服务

```bash
curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{
  "hosts": [
    {
      "ip_address": "172.17.0.2",
      "port": 8081,
      "tags": {
        "az": "us-central1-a",
        "canary": false,
        "load_balancing_weight": 50
      }
    }
  ]
}' http://localhost:8080/edsservice/myservice
```

测试：上游服务注册到 EDS 服务后

![image2019-12-30_19-56-35](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-30_19-56-35.png)

# 为集群添加新的端点

```bash
for i in 8082 8083 8084 8085
  do
      docker run -d -e EDS_SERVER_PORT=$i katacoda/docker-http-server:v4;
      sleep .5
done
```

![image2019-12-30_20-1-9](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-30_20-1-9.png)

注册新的端点到 EDS 服务

```bash
curl -X PUT --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{
    "hosts": [
        {
        "ip_address": "172.17.0.2",
        "port": 8081,
        "tags": {
            "az": "us-central1-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.5",
        "port": 8082,
        "tags": {
            "az": "us-central1-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.6",
        "port": 8083,
        "tags": {
            "az": "us-central1-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.7",
        "port": 8084,
        "tags": {
            "az": "us-central1-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.8",
        "port": 8085,
        "tags": {
            "az": "us-central1-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        }
    ]
    }' http://localhost:8080/edsservice/myservice
```

测试：共计5个上游服务

```bash
while true; do curl http://localhost; sleep .5; printf '\n'; done
```

![image2019-12-30_20-7-42](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-30_20-7-42.png)

# 删除端点

```bash
curl -X PUT --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{
  "hosts": [  ]
}' http://localhost:8080/edsservice/myservice
```

![image2019-12-30_20-9-49](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-30_20-9-49.png)

# 断联 EDS 服务

重新注册

```bash
curl -X PUT --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{
    "hosts": [
        {
        "ip_address": "172.17.0.2",
        "port": 8081,
        "tags": {
            "az": "us-central1-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.5",
        "port": 8082,
        "tags": {
            "az": "us-central1-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.6",
        "port": 8083,
        "tags": {
            "az": "us-central1-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.7",
        "port": 8084,
        "tags": {
            "az": "us-central1-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        },
        {
        "ip_address": "172.17.0.8",
        "port": 8085,
        "tags": {
            "az": "us-central1-a",
            "canary": false,
            "load_balancing_weight": 50
        }
        }
    ]
    }' http://localhost:8080/edsservice/myservice
```

测试

![image2019-12-30_20-12-25](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-30_20-12-25.png)

删除 EDS 服务

```bash
docker ps -a | awk '{ print $1,$2 }' | grep katacoda/eds_server  | awk '{print $1 }' | xargs -I {} docker stop {};
docker ps -a | awk '{ print $1,$2 }' | grep katacoda/eds_server  | awk '{print $1 }' | xargs -I {} docker rm {}
```

测试：请求全部正常

结论：即使Envoy与EDS服务器断开连接，它也继续对请求做出响应。

![image2019-12-30_20-14-33](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-30_20-14-33.png)

