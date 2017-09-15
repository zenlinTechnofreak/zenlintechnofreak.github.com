---
layout: post
title:  "翻译：Istio Pilot 设计概览"
date:   2017-09-15 15:00:01
categories: Istio translation
tags: Istio service model
---

* content
{:toc}
## 前言
本文翻译 Istio Pilot 设计概述，原文链接为 [Istio Pilot design overview]https://github.com/istio/pilot/blob/master/doc/design.md)。

## Istio Pilot 设计概述   

Istio Pilot 负责消费和传递 Istio 配置给Istio 组件。它在如K8S这样的底层集群管理平台上提供了一层抽象层，和代理控制器动态重配置Istio代理。

## Service 模型

Service模型的概述文档位于 [Service model](https://github.com/istio/pilot/blob/master/doc/service-registry.md)。它介绍了Service version的概念，Service version是一个用versions(v1,v2)或者environment(staging环境，prod环境)来细分service实例精细化的Service 概念。Istio routing rules可以通过引用service version提供服务之间的增强的流控能力。

## Configuration 模型

Istio Pilot的配置流（configuration flow）的概述文章位于 [configuration-flow](https://github.com/istio/pilot/blob/master/doc/configuration-flow.md)。指定routing rules的模式(schema) 位于代码中的`istio/pilot/api/proxy/v1/config/`。Istio配置以分布式K/V存储为后端。Istio pilot组件通过在配置存储中订阅变更事件（change events）以执行实时配置更新。

## Proxy controller（代理控制器）

Istio Pilot的代理控制器的概述文档位于 [Proxy controller](https://github.com/istio/pilot/blob/master/doc/proxy-controller.md)。Istio Pilot 监管与服务实例以sidecar container共存的代理的网格。一个proxy的代理人（agant）从服务和配置模型生成适合本地代理实例的新配置，并触发代理更新配置。

![proxy-controller](/assets/images/proxy-controller.jpg){: .align-center}

图中使用黑箭头表示数据通道，用红箭头表示控制通道。代理（Proxies）从服务抓取流量，并使用从服务发现（discovery services）和代理生成的配置（agaent-gnerated configurations）获取的控制信息将流量在内部和外部进行路由。这里获取的控制信息（control information）存储在K8S的API server中，并被kubectl或者istioctl这些操作工具管理。