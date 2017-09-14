---
layout: post
title:  "翻译：Istio 服务模型"
date:   2017-09-14 21:00:01
categories: Istio translation
tags: Istio service model
---

* content
{:toc}
## **前言**
　　本文翻译 Istio服务模型，原文链接为 [Istio service model](https://github.com/istio/pilot/blob/master/doc/service-registry.md)。

## **Istio 服务模型**   

本文描述了Istio的服务和实例的抽象模型。该模型独立于K8S和Mesos这一类的底层平台。平台(istio/pilot/platform, 有consul，eureka，kube)下的特定适配器通过平台的元数据填充模型对象的各个字段。(istio/pilot/proxy)下的代码是独立于平台的代理，它使用模型中的representaion去生成七层sidecar代理的配置文件。proxy的代码具体到各个代理的实现。

## **词汇和概念**

　　Service是具有唯一名称的应用程序的单位，其他服务在引用它的功能的时候会通过名称来调用。实例实现了Service的功能，如 pods/VMs/containers。

　　有多种版本的Service，在一个持续部署的环境，对于一个Service，会有多组实例以多种二进制的不同变体潜在运行着。这些变体不一定是不同的API版本，它们对同一个服务进行迭代变更，部署在不同的环境上，如生产环境、演示环境和开发环境等，常见发生的场景是A/B测试和金丝雀等。

## **Service**

　　每个服务拥有一个全域名(FQDN)和一个或多个监听连接的端口，有些服务会拥有独立的负载均衡器或虚IP，如DNS，使得DNS查询全域名去解析为虚IP或负载均衡IP。

　　例子：在K8S中，一个Service foo 被 foo.default.svc.cluster.local hostname 关联，拥有一个虚IP 10.0.1.1 和80以及8080监听端口。

## **Instance**

　　每个服务拥有一个或多个实例，就像服务的实际表现。实例代表实体，就像containers, pod(kubernetes/mesos), VMs等，举个例子，设想配置一个叫catalog的nodeJs后端服务，hostname为`catalogservice.mystore.com`, 跑在10个虚拟机上，监听8080端口。

　　注意在上面例子中，后端的虚机并不需要暴露服务在一个端口上。实例可以是NAT-ed（mesos）或者跑在一个隧道网络上，这取决于不同的网络规划。这些虚机可以用来托管监听在一个或多个端口上的服务，直到负载均衡器直到如何去转发一个连接到正确的pod上。

　　举个例子，一个调用，`http://catalogservice.mystore.com:8080/getCatalog`    会解析到负载局衡器IP `10.0.0.1:8080`， 负载均衡器会转发连接到十个虚机的其中一个，如`172.16.1.1:55446` 或 `172.16.0.2:22425`, `172.16.0.3:35345`等。

　　网络端点：网络IP地址和端口与各个实例关联（如上面的`172.16.0.1:55446`），这被称作网络端点。到负载均衡器IP `10.0.0.1:8080` 或主机名 `catalog.mystore.com:8080` 的调用会最终路由到一个实际的网络端点。

　　Service并不一定需要一个负载器IP，也可以拥有一个简单的基于DNV 服务器的系统，这样，DNS 服务器通过解析`catalog.mystore.com:8080`，调用到10个后端的IP。

## **Service versions**

　　一个服务的每个version 被与版本关联的一组唯一的labels区别开。Lables是分配给service version的实例的简单键值对，如， 同一个version的所有实例会拥有相同的标签。举个例子，我们可以说，`catalog.mystore.com` 拥有V1和V2 两个version。

　　我们可以说，v1版本有labels对应于`gitCommit=aeiou234，region=us-east`， V2版本拥有labels `name=kittyCat,region=us-east`。我们可以说， 实例`172.16.0.1 .. 171.16.0.5` 跑这个服务的v1版本。

　　这些实例会使用labels `gitCommit=aeiou234`, `region=us-east`通过服务注册自注册他们自己，同时， `172.16.0.6 .. 172.16.0.10` 的实例会使用labels `name=kittyCat,region=us-east`通过服务注册进行自注册。

　　Isitio期望底层平台能够提供服务注册和发现的机制。大多数的容器平台,像K8S或mesos会内置服务注册，并且pod规范会包含所有与版本相关的额labels。pod启动后，平台会使用注册表以及labels自动注册pod。对于其他的平台，可能需要一个像Consul这样的特定的服务注册代理使用服务祖册或发现方案来自动注册服务。

　　当前，Istio已经集成K8S 服务注册，可以自动发现pod的服务，或代表一个服务版本的一组唯一的pod集合。未来，Istio将支持从Mesos注册表或其他注册表拉取同类信息。

## **Service version labels**

　　当监听一个服务的不同实例时，labels将不同实例组合分开到不同的子集合里，举个例子，由labels `gitCommit=aeiou234,region=us-east` 标识的这组pod，会给所有的实例标识成 service `catalog.mystore.com`的V1版本。

　　在缺少多version的情况下，每个service拥有一个由该服务的所有实例组成的默认version，九个例子，如果`catalog.mystore.com`下的所有pod没有任何labels，那么 Istio会认为`catalog.mystore.com` 是这个服务的默认version，由10个IP范围在 `172.16.0.1 .. 172.16.0.10`的虚机组成。

　　应用不会感知service的不同version。他们通过hostname/ip 地址访问服务，但Istio会基于管理员设置的routing rules路由连接或请求到对于对应的version。这个模型使得应用代码能够从所依赖服务的演进中解耦出来， 同时提供其他的好处。

　　请注意，Istio并没有DNS能力。应用可以使用底层平台的DNS服务区解析全域名。在一些平台，像K8S， DNS名字解析成服务的负载均衡器IP地址，对于其他的平台，DNS名字可能会解析成一个或多个实例的IP地址，像mesos-dns。两种情况都是可行的，对应用程序没有影响。

## **Routing**

　　应用不会感知service的不同version。他们通过hostname/ip 地址访问服务，但Istio会基于管理员设置的routing rules路由连接或请求到对于对应的version。这个模型使得应用代码能够从所依赖服务的演进中解耦出来， 同时提供其他的好处。

　　请注意，Istio并没有DNS能力。应用可以使用底层平台的DNS服务区解析全域名。在一些平台，像K8S， DNS名字解析成服务的负载均衡器IP地址，对于其他的平台，DNS名字可能会解析成一个或多个实例的IP地址，像mesos-dns。两种情况都是可行的，对应用程序没有影响。

　　Istio sidecar 代理 在应用和服务之间拦截和转发所有的额请求和响应。对于service version的实际选择是有代理sidecar进程基于由管理员设置的routing rules动态确定。4层和7层routing roules都支持。

　　Routing rules允许代理基于像headers或url等标准， 或source/destination，或权重等标准选择labels关联version。
