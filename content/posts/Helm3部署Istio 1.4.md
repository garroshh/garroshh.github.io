---
title: "Helm3部署Istio 1.4"
date: 2020-05-11T18:56:48+08:00
draft: false

tags: ["Istio", "实验"]
categories: ["Istio"]
---

# 1. 安装 Helm3

![image-20200511185817940](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511185817940.png)

# 2. 安装 Istio 1.4

![image-20200511185835335](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511185835335.png)

# 3. 修改 Istio 参数

```bash
vim install/kubernetes/helm/istio/values.yaml
```

| 参数                             | 值            | 描述                  |
| :------------------------------- | :------------ | :-------------------- |
| grafana.enabled                  | true          | 安装 Grafana 插件     |
| tracing.enabled                  | true          | 安装 Jaeger 插件      |
| kiali.enabled                    | true          | 安装 Kiali 插件       |
| global.proxy.disablePolicyChecks | false         | 启用策略检查功能      |
| global.proxy.accessLogFile       | "/dev/stdout" | 获取 Envoy 的访问日志 |

# 4. 修改 Istio Gateway 类型为 NodePort

```bash
vim install/kubernetes/helm/istio/charts/gateways/values.yaml
```

![image-20200511185852941](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511185852941.png)

# 5. 安装 Istio CRD 资源

```bash
helm install istio-init -n istio-system install/kubernetes/helm/istio-init/
```

# 6. 安装 Istio 组件

```bash
helm install istio -n istio-system install/kubernetes/helm/istio/
```

# 7. 验证

```bash
helm list --all-namespaces
```

![image-20200511185910163](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511185910163.png)

```bash
kubectl get po -n istio-system
```

![image-20200511185927549](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511185927549.png)