---
title: "自定义 Gloo Proxy Controller"
date: 2020-05-14T15:28:57+08:00
draft: false

tags: ["Gloo", "原理"]
categories: ["Gloo"]
---

使用 Gloo [Proxy API ](https://docs.solo.io/gloo/latest/reference/api/github.com/solo-io/gloo/projects/gloo/api/v1/proxy.proto.sk/#proxy)自动为存在的 kubernetes 服务创建路由器。

# 1. 为什么编写自定义代理控制器？

Building a Proxy controller allows you to add custom Gloo operational logic to your setup. In this example, we will write a proxy controller that creates and manages a second Gloo Proxy (my-cool-proxy) alongside the Gloo-managed one (gateway-proxy). The my-cool-proxy envoy proxy will route to Gloo-discovered kubernetes services using the host header alone, relieving the Gloo admin from creating virtual services or route tables to route to each discovered service.

Other common use cases that can be solved with custom proxy controllers include:

- automatically creating http routes that redirect to equivalent https routes
- automatically routing to services based on service name and removing the service name prefix from the request path

# 2. 工作原理

A custom Proxy controller takes any inputs (in our case, Gloo custom resources in kubernetes) and writes the desired output to managed Proxy custom resource(s).

In our case, we will write a controller that takes Upstreams and Proxys as inputs and outputs a new Proxy. Then we will deploy the new controller to create and manage our new my-cool-proxy Proxy custom resource. Finally, we will deploy a second envoy proxy to kubernetes, have it register to Gloo with its role configured to match the name of our managed Proxy custom resource (my-cool-proxy), and configure it to receive configuration from Gloo.

# 3. 编码步骤

前置准备

依赖

```go
module proxycontroller
 
go 1.13
 
require (
    github.com/solo-io/gloo v1.2.12 // change to update Gloo version to build against
    github.com/solo-io/go-utils v0.11.5
    github.com/solo-io/solo-kit v0.11.15
    k8s.io/client-go v11.0.0+incompatible
)
 
replace (
    github.com/Azure/go-autorest => github.com/Azure/go-autorest v13.0.0+incompatible
    github.com/Sirupsen/logrus => github.com/sirupsen/logrus v1.4.2
    github.com/docker/docker => github.com/moby/moby v0.7.3-0.20190826074503-38ab9da00309
    k8s.io/api => k8s.io/api v0.0.0-20191004120104-195af9ec3521
    k8s.io/apiextensions-apiserver => k8s.io/apiextensions-apiserver v0.0.0-20191204090712-e0e829f17bab
    k8s.io/apimachinery => k8s.io/apimachinery v0.0.0-20191028221656-72ed19daf4bb
    k8s.io/apiserver => k8s.io/apiserver v0.0.0-20191109104512-b243870e034b
    k8s.io/cli-runtime => k8s.io/cli-runtime v0.0.0-20191004123735-6bff60de4370
    k8s.io/client-go => k8s.io/client-go v0.0.0-20191016111102-bec269661e48
    k8s.io/cloud-provider => k8s.io/cloud-provider v0.0.0-20191004125000-f72359dfc58e
    k8s.io/cluster-bootstrap => k8s.io/cluster-bootstrap v0.0.0-20191004124811-493ca03acbc1
    k8s.io/code-generator => k8s.io/code-generator v0.0.0-20191004115455-8e001e5d1894
    k8s.io/component-base => k8s.io/component-base v0.0.0-20191004121439-41066ddd0b23
    k8s.io/cri-api => k8s.io/cri-api v0.0.0-20190828162817-608eb1dad4ac
    k8s.io/csi-translation-lib => k8s.io/csi-translation-lib v0.0.0-20191004125145-7118cc13aa0a
    k8s.io/gengo => k8s.io/gengo v0.0.0-20190822140433-26a664648505
    k8s.io/heapster => k8s.io/heapster v1.2.0-beta.1
    k8s.io/klog => github.com/stefanprodan/klog v0.0.0-20190418165334-9cbb78b20423
    k8s.io/kube-aggregator => k8s.io/kube-aggregator v0.0.0-20191104231939-9e18019dec40
    k8s.io/kube-controller-manager => k8s.io/kube-controller-manager v0.0.0-20191004124629-b9859bb1ce71
    k8s.io/kube-openapi => k8s.io/kube-openapi v0.0.0-20190816220812-743ec37842bf
    k8s.io/kube-proxy => k8s.io/kube-proxy v0.0.0-20191004124112-c4ee2f9e1e0a
    k8s.io/kube-scheduler => k8s.io/kube-scheduler v0.0.0-20191004124444-89f3bbd82341
    k8s.io/kubectl => k8s.io/kubectl v0.0.0-20191004125858-14647fd13a8b
    k8s.io/kubelet => k8s.io/kubelet v0.0.0-20191004124258-ac1ea479bd3a
    k8s.io/legacy-cloud-providers => k8s.io/legacy-cloud-providers v0.0.0-20191203122058-2ae7e9ca8470
    k8s.io/metrics => k8s.io/metrics v0.0.0-20191004123543-798934cf5e10
    k8s.io/node-api => k8s.io/node-api v0.0.0-20191004125527-f5592a7bd6b6
    k8s.io/repo-infra => k8s.io/repo-infra v0.0.0-20181204233714-00fe14e3d1a3
    k8s.io/sample-apiserver => k8s.io/sample-apiserver v0.0.0-20191028231949-ceef03da3009
    k8s.io/sample-cli-plugin => k8s.io/sample-cli-plugin v0.0.0-20191004123926-88de2937c61b
    k8s.io/sample-controller => k8s.io/sample-controller v0.0.0-20191004122958-d040c2be0d0b
    k8s.io/utils => k8s.io/utils v0.0.0-20190801114015-581e00157fb1
)
```

代码结构

```go
package main
 
// all the import's we'll need for this controller
import (
    "context"
    "log"
    "os"
    "time"
 
    v1 "github.com/solo-io/gloo/projects/gloo/pkg/api/v1"
    matchers "github.com/solo-io/gloo/projects/gloo/pkg/api/v1/core/matchers"
    "github.com/solo-io/go-utils/kubeutils"
    "github.com/solo-io/solo-kit/pkg/api/v1/clients"
    "github.com/solo-io/solo-kit/pkg/api/v1/clients/factory"
    "github.com/solo-io/solo-kit/pkg/api/v1/clients/kube"
    core "github.com/solo-io/solo-kit/pkg/api/v1/resources/core"
 
    // import for GKE
    _ "k8s.io/client-go/plugin/pkg/client/auth/gcp"
)
 
 
func main() {}
 
 
// make our lives easy
func must(err error) {
    if err != nil {
        panic(err)
    }
}
```

## 3.2. Gloo API Clients

```go
func initGlooClients(ctx context.Context) (v1.UpstreamClient, v1.ProxyClient) {
    // root rest config
    restConfig, err := kubeutils.GetConfig(
        os.Getenv("KUBERNETES_MASTER_URL"),
        os.Getenv("KUBECONFIG"))
    must(err)
 
    // wrapper for kubernetes shared informer factory
    cache := kube.NewKubeCache(ctx)
 
    // initialize the CRD client for Gloo Upstreams
    upstreamClient, err := v1.NewUpstreamClient(&factory.KubeResourceClientFactory{
        Crd:             v1.UpstreamCrd,
        Cfg:             restConfig,
        SharedCache:     cache,
        SkipCrdCreation: true,
    })
    must(err)
 
    // registering the client registers the type with the client cache
    err = upstreamClient.Register()
    must(err)
 
    // initialize the CRD client for Gloo Proxies
    proxyClient, err := v1.NewProxyClient(&factory.KubeResourceClientFactory{
        Crd:             v1.ProxyCrd,
        Cfg:             restConfig,
        SharedCache:     cache,
        SkipCrdCreation: true,
    })
    must(err)
 
    // registering the client registers the type with the client cache
    err = proxyClient.Register()
    must(err)
 
    return upstreamClient, proxyClient
}
```

## 3.3. Proxy Configuration

```go
// in this function we'll generate an opinionated
// proxy object with a routes for each of our upstreams
func makeDesiredProxy(upstreams v1.UpstreamList) *v1.Proxy {
 
    // each virtual host represents the table of routes for a given
    // domain or set of domains.
    // in this example, we'll create one virtual host
    // for each upstream.
    var virtualHosts []*v1.VirtualHost
 
    for _, upstream := range upstreams {
        upstreamRef := upstream.Metadata.Ref()
        // create a virtual host for each upstream
        vHostForUpstream := &v1.VirtualHost{
            // logical name of the virtual host, should be unique across vhosts
            Name: upstream.Metadata.Name,
 
            // the domain will be our "matcher".
            // requests with the Host header equal to the upstream name
            // will be routed to this upstream
            Domains: []string{upstream.Metadata.Name},
 
            // we'll create just one route designed to match any request
            // and send it to the upstream for this domain
            Routes: []*v1.Route{{
                // use a basic catch-all matcher
                Matchers: []*matchers.Matcher{
                    &matchers.Matcher{
                        PathSpecifier: &matchers.Matcher_Prefix{
                            Prefix: "/",
                        },
                    },
                },
 
                // tell Gloo where to send the requests
                Action: &v1.Route_RouteAction{
                    RouteAction: &v1.RouteAction{
                        Destination: &v1.RouteAction_Single{
                            // single destination
                            Single: &v1.Destination{
                                DestinationType: &v1.Destination_Upstream{
                                    // a "reference" to the upstream, which is a Namespace/Name tuple
                                    Upstream: &upstreamRef,
                                },
                            },
                        },
                    },
                },
            }},
        }
 
        virtualHosts = append(virtualHosts, vHostForUpstream)
    }
 
    desiredProxy := &v1.Proxy{
        // metadata will be translated to Kubernetes ObjectMeta
        Metadata: core.Metadata{Namespace: "gloo-system", Name: "my-cool-proxy"},
 
        // we have the option of creating multiple listeners,
        // but for the purpose of this example we'll just use one
        Listeners: []*v1.Listener{{
            // logical name for the listener
            Name: "my-amazing-listener",
 
            // instruct envoy to bind to all interfaces on port 8080
            BindAddress: "::", BindPort: 8080,
 
            // at this point you determine what type of listener
            // to use. here we'll be using the HTTP Listener
            // other listener types are currently unsupported,
            // but future
            ListenerType: &v1.Listener_HttpListener{
                HttpListener: &v1.HttpListener{
                    // insert our list of virtual hosts here
                    VirtualHosts: virtualHosts,
                },
            }},
        },
    }
 
    return desiredProxy
}
```

## 3.4. Event Loop

```go
// we received a new list of upstreams! regenerate the desired proxy
// and write it as a CRD to Kubernetes
func resync(ctx context.Context, upstreams v1.UpstreamList, client v1.ProxyClient) {
    desiredProxy := makeDesiredProxy(upstreams)
 
    // see if the proxy exists. if yes, update; if no, create
    existingProxy, err := client.Read(
        desiredProxy.Metadata.Namespace,
        desiredProxy.Metadata.Name,
        clients.ReadOpts{Ctx: ctx})
 
    // proxy exists! this is an update, not a create
    if err == nil {
 
        // sleep for 1s as Gloo may be re-validating our proxy, which can cause resource version to change
        time.Sleep(time.Second)
 
        // ensure resource version is the latest
        existingProxy, err = client.Read(
            desiredProxy.Metadata.Namespace,
            desiredProxy.Metadata.Name,
            clients.ReadOpts{Ctx: ctx})
        must(err)
 
        // update the resource version on our desired proxy
        desiredProxy.Metadata.ResourceVersion = existingProxy.Metadata.ResourceVersion
    }
 
    // write!
    written, err := client.Write(desiredProxy,
        clients.WriteOpts{Ctx: ctx, OverwriteExisting: true})
 
    must(err)
 
    log.Printf("wrote proxy object: %+v\n", written)
}
```

## 3.5. Main Function

```go
func main() {
    // root context for the whole thing
    ctx := context.Background()
 
    // initialize Gloo API clients
    upstreamClient, proxyClient := initGlooClients(ctx)
 
    // start a watch on upstreams. we'll use this as our trigger
    // whenever upstreams are modified, we'll trigger our sync function
    upstreamWatch, watchErrors, initError := upstreamClient.Watch("gloo-system",
        clients.WatchOpts{Ctx: ctx})
    must(initError)
 
    // our "event loop". an event occurs whenever the list of upstreams has been updated
    for {
        select {
        // if we error during watch, just exit
        case err := <-watchErrors:
            must(err)
        // process a new upstream list
        case newUpstreamList := <-upstreamWatch:
            // we received a new list of upstreams from our watch,
            resync(ctx, newUpstreamList, proxyClient)
        }
    }
}
```

## 3.6. Run

```shell
go run example/proxycontroller/proxycontroller.go
```

## 3.7. Test

```shell
curl $(glooctl proxy url -n default --name my-cool-proxy)/api/pets -H "Host: default-petstore-8080"
```

returns

```shell
[{"id":1,"name":"Dog","status":"available"},{"id":2,"name":"Cat","status":"pending"}]
```

