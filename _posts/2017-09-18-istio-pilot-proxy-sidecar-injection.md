---
layout: post
title:  "翻译：Istio Pilot Proxy sidecar 注入"
date:   2017-09-16 18:30:01
categories: Istio translation
tags: Istio Pilot proxy sidecar injection
---

* content
{:toc}
## 前言
本文翻译 Istio Pilot Proxy sidecar 注入，原文链接为 [**proxy-injection**t](https://github.com/istio/pilot/blob/master/doc/proxy-injection.md)。

## 自动注入   

Istio的目标是给最终用户部署一个透明的切最小影响的代理注入。理想情况下，一个K8S注入控制器会通过重写specs以在提交之前包含必要的初始化和代理容器，但是当前这需要先upstream K8S的变化，而这是我们当前想要避免的。如果存在一种动态插件机制以使准入控制器可以保持在out-of-tree的状态将会是较好的替代方法。当前还没有平台支持这样，但是已经创建了一个增加这个特性的proposal（查阅 [ Proposal: Extensible Admission Control](https://github.com/kubernetes/community/pull/132/)）。
Istio 自动代理注入将被长期跟踪在[ Kubernetes Admission Controller for proxy injection](https://github.com/istio/pilot/issues/57)。

## 手动注入

一个短期的解决缺少合适的istio准入控制器的解决办法是客户端注入。使用 `istioctl kube-inject` 去增加必要的额配置到K8S resources files中。

```bash
istioctl kube-inject -f deployment.yaml -o deployment-with-istio.yaml
```


或者 在applying之前即时更新resource。

```bash
istioctl kube-inject -f depoyment.yaml | kubectl apply -f -
```


或者更新一个存在的deployment

```bash
kubectl get deployment -o yaml | istioctl kube-inject -f - | kubectl apply -f -
```


istioctl kube-inject 会在K8S obj、DaemonSet和Deployment YAML 资源文件中更新 [PodTemplateSpec](https://kubernetes.io/docs/api-reference/v1.7/#_v1_podtemplatespec)。支持额外的基于pod的必要的resource类型。

不受支持的reources未修改，例如，在一个包含多个Service和Configmap以及Deployment定义的复杂应用的独立文件上执行  `istioctl kube-inject` 。

Istio 项目正持续发展，所以low-level 的代理配置可能在没有通知的情况下发生变化。当你产生疑惑时，请在你的原始的部署上重新运行`istioctl kube-inject`。

```bash
$ istioctl kube-inject --help
Inject istio runtime into existing kubernete resources

Usage:
   inject [flags]

Flags:

​```
  --discoveryPort int     Pilot discovery port (default 8080)
​```

  -f, --filename string       Unmodified input kubernetes resource filename

​```
  --initImage string      Istio init image (default "docker.io/istio/init:latest")
  --mixerPort int         Mixer port (default 9091)
​```

  -o, --output string         Modified output kubernetes resource filename

​```
  --runtimeImage string   Istio runtime image (default "docker.io/istio/runtime:latest")
  --sidecarProxyUID int   Sidecar proxy UID (default 1337)
  --verbosity int         Runtime verbosity (default 2)
​```

```


