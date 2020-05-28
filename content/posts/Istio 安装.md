---
title: "Istio 安装"
date: 2020-05-11T14:48:49+08:00
draft: false

tags: ["Istio", "实验"]
categories: ["Istio"]
---

# 三种方式

1.Quick Start

参考官网：https://istio.io/docs/setup/install/kubernetes/

这里无需安装 Helm，只使用基本的 Kubernetes 命令，就能设置一个预配置的 Istio **demo**。

要正式在生产环境上安装 Istio，官网推荐使用 Helm 进行安装，其中包含了大量选项，可以对 Istio 的具体配置进行选择和管理，来满足特定的使用要求。

2.Helm + Tiller

3.Helm template

# 使用 helm template 进行安装

Helm Tiller 或 Helm template 差别不大，这里选择Helm template 方式。

1.为 Istio 组件创建命名空间 istio-system：

````bash
$ kubectl create namespace istio-system 
````

2.使用 kubectl apply 安装所有的 Istio CRD，命令执行之后，会隔一段时间才能被 Kubernetes API Server 收到。这里为了后续研究Istio特性，把Helm及部署yaml单独拷贝出来，放在一个名为istio目录内。

```bash
$ helm template install/kubernetes/helm/istio-init --name dc-istio-init --namespace istio-system > install/kubernetes/helm/istio-init/istio-init.yaml  //当前目录为解压istio-1.3.4.tar.gz后的根目录
 
$ kubectl apply -f install/kubernetes/helm/istio-init/istio-init.yaml
```

3.使用如下命令以确保全部 23 个 Istio CRD 被提交到 Kubernetes api-server：

```bash
$ kubectl get crds | grep 'istio.io' | wc -l
```

4.选择一个配置文件，接着部署与你选择的配置文件相对应的 Istio 的核心组件，官网建议在生产环境部署中使用 **default** 配置文件:

```bash
$ helm template install/kubernetes/helm/istio --name dc-istio --namespace istio-system > install/kubernetes/helm/istio/istio.yaml  //当前目录为解压istio-1.3.4.tar.gz后的根目录
 
$ vim install/kubernetes/helm/istio/charts/gateways/value.yaml   //修改istio-ingressgateway.type=NodePort
 
$ kubectl apply -f install/kubernetes/helm/istio/istio.yaml
```

你可以添加一个或多个 --set = 来进一步自定义 helm 命令的安装选项，或者采用更改配置文件的方式。

注：默认情况下，Istio 使用 LoadBalancer 服务类型，而有些平台是不支持 LoadBalancer 服务的。对于缺少 LoadBalancer 支持的平台，执行下面的安装步骤时，可以在 Helm 命令中加入 --set gateways.istio-ingressgateway.type=NodePort 选项，使用 NodePort 来替代 LoadBalancer 服务类型，也可以如上述第三条命令所示，进入相应的文件修改。

注：后续所有文章当前所处目录都为解压istio-1.3.4.tar.gz后的根目录，就不再注明

# 验证安装

确认 Istio Kubernetes services 服务是否正常，确保部署了相应的 Kubernetes pod 并且 STATUS 是 Running

```bash
$ kubectl get svc -n istio-system
 
$ kubectl get po -n istio-system
```

# 卸载

```bash
$ kubectl delete -f install/kubernetes/helm/istio/istio.yaml
 
$ kubectl delete ns istio-system
 
$ kubectl delete -f install/kubernetes/helm/istio-init/files
```

# 趟过的坑:

1.对于二进制方式部署的kubernetes集群，kube-apiserver 的准入控制参数，需添加MutatingAdmissionWebhook,ValidatingAdmissionWebhook 这两个webhook。

![image-20200511145618380](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511145618380.png)

2.kubernetes集群master节点需部署kube-proxy，kubelet这两个组件，否则sidecar注入不进去。