---
layout: post
title:  "翻译：Istio Pilot 配置流程"
date:   2017-09-16 18:30:01
categories: Istio translation
tags: Istio Pilot configuration flow
---

* content
{:toc}
## 前言
本文翻译 Istio Pilot 配置流程，原文链接为 [Configuration flow in Istio Pilot](https://github.com/istio/pilot/blob/master/doc/configuration-flow.md)。

## Istio Pilot的配置流程（Configuration flow）   

Istio 配置由配置对象（configuration objects）组成。每个对象拥有kind（像routing rule就是一种kind），name和namespace，唯一标识对象。 Istio Pilot 提供一套基于rest的接口，并用objects的key去操作objects。Objects可以被用keys来检索并按照kind或namespace列出来(listed)，或删除(deleted)，或创建(created， 就是POST操作)，或更新（updated， 就是PUT 操作）。

配置注册表（configuration registry）提供了一个监视配置存储变更的缓存控制器借口。举个例子，一个功能能订阅接收特性对象的创建objects事件。控制器流化这些订阅者的事件，并维护一个所有存储的本地缓存视图。

配置流开始于操作者(operator 下面的1小节讲解)，接着到平台指定的永久存储(persistent storage 下面的2小节讲解)，然后在远程代理组件（remote proxy component 下面的3小节讲解）上触发一个通知事件。

## 1. 用户输入

Istio Pilot 提供istioctl的命令行工具，用于暴露基于rest的配置存储借口。这个工具为已注册的Isito配置类型验证输入的Yaml文件满足模式要求（schema），并存储验证过的输入信息到存储中。

## 2. 配置存储

Istio 配置存储（configuration storage） 是平台相关的。下面，我们聚焦K8S， 因为Istio Pilot模型与K8S模型相近，类似于第三方的资源（K8S的Resources）。实际上，如果kubectl能够支持第三方资源的验证和补丁语义，kubectl 能够代替istioctl。

每个Istio配置对象 被存储作一个K8S的三方资源类型 istioconfig。这个资源的名字是一个Istio和configuration的组合， 并namespaces在K8S和Istio之间共享。在内部，这意味着K8S为istio在etcd中分配了一个key子集，暴露一个用于流更新的http 端点给存储，并为创建一个缓存控制器接口（cached controller interface）提供了必要的API 机制。你可以使用`kubectl get istioconfigs` 查询K/V存储。

## 3. 代理配置更新

一旦一个配置对象存在于etcd中，存储的更新就会被发送给代理的代理控制器。代理的agent通过创建新的代理配置并本地化到agent所运行的pod来做出反应。如果配置不变，则通知被成功处理。相反地，agent触发代理重配置（proxy reconfiguration）。一个Istio未来的目标，是提供一个反馈回路，当代理配置失败或者其他原因导致代理失败时报告给存储。
