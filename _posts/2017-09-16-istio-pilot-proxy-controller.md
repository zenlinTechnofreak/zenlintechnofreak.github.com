---
layout: post
title:  "翻译： Istio Pilot 代理控制器"
date:   2017-09-16 19:00:01
categories: Istio translation
tags: Istio pilot proxy controller
---

* content
{:toc}
## 前言
本文翻译 Istio Pilot 代理控制器，原文链接为 [roxy controller](https://github.com/istio/pilot/blob/master/doc/proxy-controller.md)。

## 代理控制器   

Istio Pilot 通过传播服务注册信息和路由规则给目的代理 来控制Istio代理的网格。当前，Istio 代理是基于Envoy的，Envoy的控制器由以下两部分组成：

1. 代理 agent， 一个管理脚本负责从Services和rules的抽象模型生成Envoy的配置，并且触发代理重启。
2. 服务发现，继承自Envoy的服务发现APIs，负责为Envoy代理发布消息去消费。

## 代理注入

为保证所有的流量能被Istio 代理抓取。Istio Pilot依赖于iptables规则。 服务实例用常规的HTTP headers进行通信，但所有的请求会被Istio 代理基于请求的元数据和Istio路由规则 抓取并重路由。
代理注入的详细细节位于[proxy-injection](https://github.com/istio/pilot/blob/master/doc/proxy-injection.md)。由于所有的网络流量会被抓取，外部服务请求在服务模型中需要特殊的外部服务representation。

## 代理agent

代理agent是一个简单的agent，主要的工作是订阅网状拓扑和配置存储中的变化，并重配置代理。随着越来越多部分的Envoy配置通过服务发现变得有价值，我们这正渐渐地将配置生成委托给服务发现。举个例子，TCP 代理配置大部分由本地代理agent来配置，因为Envoy尚未实现对tcp_proxy的路由发现的支持。

## 发现服务（discovery service）

服务发现发布服务拓扑和路由信息给网格中所有的代理。每个代理都携带一个身份（在k8s sidecar 部署的场景中为pod name和IP address）。Envoy 使用这个身份构造一个请求给发现服务（discovery service）。发现服务（discovery service）通过服务注册表（service registry）计算跑在这个代理address上的服务实例集合，并创建适配于发出请求的代理的Envoy配置。
Istio Pilot暴露了三种类型的发现服务：

1. SDS： 是服务发现（service discovery），负责监听集群中的 `ip:port` 对的集合
2. CDS： 是集群发现（cluster discovery），负责监听所有的Envoy 集群。
3. RDS：是路由发现（route discovery），负责监听HTTP路由，代理身份（proxy identity） 对于应用具有源服务条件（with source service conditions）的路由规则很重要。

## 路由规则

路由规则被定义在Istio API [proto schema](https://github.com/istio/api/blob/master/proxy/v1/config/route_rule.proto) 中。可用的例子放在[integration tests](https://github.com/istio/pilot/tree/master/test/integration)中。

## Ingress 和 egress

待续