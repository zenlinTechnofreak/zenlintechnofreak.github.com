---
layout: post
title:  "翻译：Istio Pilot sidecar 代理流量重定向方法"
date:   2017-09-16 18:30:01
categories: Istio translation
tags: Istio Pilot proxy sidecar injection
---

* content
{:toc}
## 前言
本文翻译 Istio Pilot sidecar 代理流量重定向方法，原文链接为 [**proxy-redirection-configuration**](https://github.com/istio/pilot/blob/master/doc/proxy-redirection-configuration.md)。

查看[ Microservice Proxies in Kubernetes](https://docs.google.com/document/d/1v870Igrj5QS52G9O43fhxbV_S3mpvf_H6Hb8r85KZLY/edit#) 了解对服务网格代理部署（service-mesh proxy deployments）的详细比较和初始设计与实现方案。
本文介绍当前的对于重定向每个pod的sidecar proxy的进站和出站流量的方法和备选方案。

## 要求   

1. 透明度 与 平台独立性和系统稳定性 的平衡
2. 本地代理感知应用的换回旁路（例子：通过本地处理7层服务而不是通过虚IP）
3. 安全性（避免授予代理程序非必要的权限）

如果Envoy受到非提升特权的攻击，攻击影响将是最小化的。如果我们增加任何特权给Envoy 容器，则会引起很大的震动。随着特权升级，增加到Envoy的特性受到很多审查（举例： URL中的正则表达式）。高级特权会被限制成一次性的初始化（通过 init-container）中，并保持在proxy container之外。

4. 特殊对待kube-system的namespace（举例：平滑、ingress 控制器），待续

## proxy的进站流量和出站流量的重定向配置

下面假设所有的进站和出站流量被重定向（通过iptables）到一个独立的port。Envoy进程绑定到这个port并用SO_ORIGINAL_DST 去覆盖原始的目的地和传递给匹配的过滤器。（查看 [config-listeners](https://envoyproxy.github.io/envoy/configuration/listeners/listeners.htmlhttps://envoyproxy.github.io/envoy/configuration/listeners/listeners.html)）。

```bash
 ISTIO_PROXY_PORT=5001
```

Envoy代理集成的UID，用来做基于UID的iptables重定向

```bash
 ISTIO_PROXY_UID=1337
```

基于mark的iptables重定向的数据包标记

```bash
 ISTIO_SO_MARK=0x1
```

### 基于net_cls的iptables重定向的网络分类ID

```bash
 ISTIO_NET_CLS_ID=0x100001
```

### 进站

入栈重定向的iptables规则很简单，直接假设所有的流量需要重定向到代理。如果进站流量需要bypass代理，那么需要增加额外的规则，例子：

```bash
 iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-port ${ISTIO_PROXY_PORT}
```

### 出站

UID方法是当前的计划主要是因为proxy container中不需要提权。该方法的缺点是需要UID协作，但是短期计划里面这不是一个问题，在长期的计划里面，可以探索缓解措施，例如，将可覆盖的UID暴露给最终用户。
UID(--uid-owner)

```bash
 iptables -t nat -A OUTPUT -p tcp -j REDIRECT ! -s 127.0.0.1/32 \
  --to-port ${ISTIO_PROXY_PORT} -m owner '!' --uid-owner ${ISTIO_PROXY_UID}
```

1. 不需要更改Envoy。
2. 在proxy和init-container之间需要进行UID协作。Istio可能不一定能够控制UID(例如，docker的设置)。UID可以被做成可配置或可重写的，因此，proxy UID 不会与其他的终端用户UID或进程冲突。查看 [Security Context](https://kubernetes.io/docs/user-guide/security-context)

## 考虑中的替代方案

### SO_MARK 数据包制作

（查看 http://man7.org/linux/man-pages/man7/socket.7.html）

```bash
 iptables -t nat -A OUTPUT -p tcp -j REDIRECT ! -s 127.0.0.1/32 \ 
  --to-port ${ISTIO_PROXY_PORT} -m mark '!' --mark ${ISTIO_SO_MARK}
```

1. `${ISTIO_SO_MARK} ` 的值被配置在pod spec（例如configmap、envvar），被init-container 用来编程iptables，proxy agent 用每一个上游集群的合适的SO_MARK 值创建envoy config。
2. 要求增加`SO_MARK`以支持envoy，可能配置在每个上游集群？
3. 要求proxy 启动时带着`NET_CAP_ADMIN` 去设置sockets的SO_MARK.

### 网络cgroup分类器

（查看 https://www.kernel.org/doc/Documentation/cgroup-v1/net_cls.txt）

```bash
iptables -t nat -A OUTPUT -p tcp -j REDIRECT ! -s 127.0.0.1/32 \
  --to-port ${ISTIO_PROXY_PORT} -m cgroups '!' --cgroup ${ISTIO_NET_CLS_ID}
```

1. 不要求Envoy变更
2. 要求用新的iptables版本（例如1.6.0）
3. 需要通过pod spec暴露`/sys/fs/cgroup/net_cls`。会在proxy container内要求额外的特权， 通过proxy agent可以在启动或执行envoy之前丢弃特权。另外，节点级的代理可以通过诸如UDS之类的每pod proxy agent来代表管理net_cls。
4. 当proxy 崩溃、重启等的时候，Proxy agent需要用envoy proxy PID更新 /sys/fs/group/net_cls/%ltproxy-group%gt/tasks文件

### 每个服务的明确规则

1. 非常明确的
2. 覆盖所有的额客户端和服务端services的大量iptables rules。由于大量规则导致的系统不稳定问题。