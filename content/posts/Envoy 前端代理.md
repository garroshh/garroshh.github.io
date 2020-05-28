---
title: "Envoy 前端代理"
date: 2020-05-06T11:12:34+08:00
draft: false

tags: ["Envoy", "实验"]
categories: ["Envoy"]
---

# 目标

使用 Envoy 基于请求的 URL 代理流量到不同的Python服务

实验代码部署架构如图所示：

![image2019-12-26_17-43-49](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-26_17-43-49.png)

# Clone Envoy 仓库，并启动所有的容器

```bash
git clone https://github.com/envoyproxy/envoy.git
```

![image2019-12-26_18-5-38](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-26_18-5-38.png)

# 测试

## 测试 Envoy 路由能力

![image2019-12-26_18-7-8](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-26_18-7-8.png)

![image2019-12-26_18-7-26](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-26_18-7-26.png)

## 测试Envoy负载均衡能力

![image2019-12-26_18-10-36](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-26_18-10-36.png)

![image2019-12-26_18-10-58](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-26_18-10-58.png)

![image2019-12-26_18-12-48](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-26_18-12-48.png)

## 通过管理端口查看相关信息

![image2019-12-26_18-16-7](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-26_18-16-7.png)

![image2019-12-26_18-16-47](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image2019-12-26_18-16-47.png)