---
title: "Kubeadm搭建Kubernetes集群"
date: 2020-05-11T18:40:05+08:00
draft: false

tags: ["Istio", "实验"]
categories: ["Istio"]
---

# 1. 机器配置

| OS                      | 主机名 | 配置 | ip          |
| :---------------------- | :----- | :--- | :---------- |
| Centos7(Docker 19.03.5) | m-100  | 4C8G | 10.12.1.100 |
| Centos7(Docker 19.03.5) | w-101  | 4C8G | 10.12.1.101 |
| Centos7(Docker 19.03.5) | w-102  | 4C8G | 10.12.1.102 |

# 2. 主节点和工作节点配置

### 安装 kubeadm, kubelet 和 kubectl，其中 kubectl 工作节点可选择性安装

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
 
# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
 
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
 
systemctl enable kubelet
```

### 设置 iptables

```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

### 关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
```

### 设置 cgroup 为 systemd

```bash
cat > /etc/docker/daemon.json <<EOF
{
"exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
systemctl daemon-reload
systemctl enable docker
systemctl restart docker
```

### 关闭 swapoff

```bash
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 设置宿主机 dns

```bash
vim /etc/hosts
```

### 集群初始化

```bash
[root@m-100 ~]# kubeadm init --pod-network-cidr=192.168.0.0/16
W0217 10:01:19.304156   10476 version.go:101] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://dl.k8s.io/release/stable-1.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
W0217 10:01:19.304321   10476 version.go:102] falling back to the local client version: v1.17.3
W0217 10:01:19.304755   10476 validation.go:28] Cannot validate kube-proxy config - no validator is available
W0217 10:01:19.304772   10476 validation.go:28] Cannot validate kubelet config - no validator is available
[init] Using Kubernetes version: v1.17.3
[preflight] Running pre-flight checks
    [WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [m-100 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.12.1.100]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [m-100 localhost] and IPs [10.12.1.100 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [m-100 localhost] and IPs [10.12.1.100 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0217 10:01:24.604930   10476 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0217 10:01:24.606403   10476 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 35.003335 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.17" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node m-100 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node m-100 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 0aizeu.ji5y0dooy8g658n1
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy
 
Your Kubernetes control-plane has initialized successfully!
 
To start using your cluster, you need to run the following as a regular user:
 
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
 
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/
 
Then you can join any number of worker nodes by running the following on each as root:
 
kubeadm join 10.12.1.100:6443 --token 0aizeu.ji5y0dooy8g658n1 \
    --discovery-token-ca-cert-hash sha256:1bc30ca4b0c09582bf0537ca2f516ae2c510becd5bdefe4ec866f9201f3519a5
```

### 配置 kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 验证 kubernetes 集群

```bash
kubectl get po -A
```

![image-20200511184327832](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511184327832.png)

发现 coredns pending，因为还未安装网络插件。

启用 calico 网络插件

```bash
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
```

![image-20200511184405481](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511184405481.png)

![image-20200511184430873](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511184430873.png)

此时，只有一个主节点，calico 网络运行不起来，执行如下命令

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

![image-20200511184458185](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511184458185.png)

# 3. 工作节点配置

### 添加工作节点

```bash
kubeadm join 10.12.1.100:6443 --token 0aizeu.ji5y0dooy8g658n1 \
    --discovery-token-ca-cert-hash sha256:1bc30ca4b0c09582bf0537ca2f516ae2c510becd5bdefe4ec866f9201f3519a5
```

![image-20200511185404042](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511185404042.png)

### 验证工作节点

```bash
kubectl get nodes
 
kubectl get po -A -o wide
```

![image-20200511184534034](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511184534034.png)

![image-20200511184553066](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511184553066.png)

### 添加第二个工作节点

```bash
kubeadm join 10.12.1.100:6443 --token 0aizeu.ji5y0dooy8g658n1 \
    --discovery-token-ca-cert-hash sha256:1bc30ca4b0c09582bf0537ca2f516ae2c510becd5bdefe4ec866f9201f3519a5
```

![image-20200511184629027](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511184629027.png)

### 验证工作节点

```bash
kubectl get nodes
 
kubectl get po -A -o wide
```

![image-20200511184700542](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511184700542.png)

![image-20200511184716638](https://cdn.jsdelivr.net/gh/garroshh/figurebed/img/image-20200511184716638.png)