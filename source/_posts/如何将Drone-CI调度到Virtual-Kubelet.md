---
title: 如何将Drone CI调度到Virtual Kubelet（一）
date: 2019-11-18 23:53:30
tags:
- devops
- kubernetes
- drone
- ci/cd
---

## 什么是 virtual kubelet

以下是来自 [Virtual Kubelet](https://github.com/virtual-kubelet/virtual-kubelet#virtual-kubelet) 项目的文档的中文翻译。

> Virtual Kubelet 是开源的 Kubernetes kubelet 的实现，它可以伪装成 Kubelet 将 Kubernetes 连接到其他 API。这样就允许 Node 的背后由其他服务支撑，例如：ACI, AWS Fargate, IoT Edge。Virtual Kubelet 的主要作用是扩展无服务器（Serverless）平台，让它能够与 Kubernetes 通信。

以 阿里云ECI 为例，ECI 是阿里云的弹性容器实例。可以将每台 ECI 实例看作是一个 Container，所以它的创建和销毁是很廉价的。同时它拥有启动快（秒级）、成本低（按运行的秒数收费）、弹性能力强等特点。通过 Virtual Kubelet 提供的 [Kubernetes API](https://github.com/virtual-kubelet/virtual-kubelet#current-features)，ECI 就能和 K8S 交互，我们就可以在 ECI 上执行创建 Pod 或者删除 Pod 等操作。

## 使用 virtual kubelet 执行 drone 任务的好处

在使用 virtual kubelet 之前，为了不影响业务的稳定性。我们的 K8S 集群中开了几台固定的 ECS 实例专门给 CI 使用（给这些 Node 打上了污点，所以业务的服务不会调度到上面）。

这种做法虽然在不影响业务的情况下也保证了 CI 的稳定运行，但是它会造成一定程度的浪费。因为 CI 本身不像大多数业务服务，需要一天 24 小时的运行。CI 的场景是白天需要很多资源，但是到了晚上几乎不消耗任何资源。所以可以说这些机器有接近 1/3 的时间是在浪费 💰 的。

虽说 K8S 本身也提供了动态扩容机制，可以设置很少的固定资源再通过 CA 动态扩容集群来减少资源消耗。但是 CA 的启动速度（分钟级）满足不了 CI 这种时效要求高的场景。

