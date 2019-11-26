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

以 `阿里云ECI` 为例（以下均以 ECI 代指 Virtual Kubelet），ECI 是阿里云的弹性容器实例。可以将每台 ECI 实例看作是一个 `Container`，所以它的创建和销毁是很廉价的。同时它拥有启动快（秒级）、成本低（按运行的秒数收费）、弹性能力强等特点。通过 Virtual Kubelet 提供的 [Kubernetes API](https://github.com/virtual-kubelet/virtual-kubelet#current-features)，ECI 就能和 K8S 交互，我们就可以在 ECI 上执行创建 Pod 或者删除 Pod 等操作。

## 使用 virtual kubelet 执行 drone 任务的好处

在使用 virtual kubelet 之前，为了不影响业务的稳定性。我们的 K8S 集群中开了几台固定的 ECS 实例专门给 CI 使用（给这些 Node 打上了污点，所以业务的服务不会调度到上面）。

这种做法虽然在不影响业务的情况下也保证了 CI 的稳定运行，但是它会造成一定程度的浪费。因为 CI 本身不像大多数业务服务，需要一天 24 小时的运行。CI 的场景是白天需要很多资源，但是到了晚上几乎不消耗任何资源。所以可以说这些机器有接近 1/3 的时间是在浪费 💰 的。

虽说 K8S 本身也提供了动态扩容机制，可以设置很少的固定资源再通过 CA 动态扩容集群来减少资源消耗。但是 CA 的启动速度（分钟级）满足不了 CI 这种时效要求高的场景。

<!-- more -->

## 如何让 Drone 兼容 ECI

### HostPath

关于 Drone In K8S 的运行模式，可以翻看我之前写的文章 [Drone 在 K8S 中执行一次构建都经历了什么](https://blog.domgoer.io/2019/10/22/ck34o2dab0007u79ki5yxke0l/)。

简单来说，`Drone Server` 接收到构建任务后，会在其运行的 `Namespace`（假设为 CICD）下创建一个 `Job`，该 Job 会创建一个随机名称的 Namespace，再在创建出来的 Namespace 按配置文件中的顺序执行每个 Step，每个 `Step` 就是一个 `Pod`。这些 Pods 之间通过 `HostPath` 类型的 `Volume` 来交换文件。

![执行过程](/images/drone_virtual_kubelet/origin.jpg)

那么问题来了，ECI 是不支持 HostPath Volume 的。它只支持 EmptyDir、NFS 和 ConfigFile（也就是 ConfigMap 和 Secret）。

所以要如何解决之前提的 Pods 之间使用 HostPath 交换文件的问题呢？首先想到了是通过 `Mutating Webhook` 将 HostPath 替换成 NFS，这样每个 Pod 之间使用 NFS 共享文件，这样带来了 NFS 文件的清理问题，不像之前 `Pipeline` 执行完之后可以直接使用 `os.Remove(path)` 来清理文件，使用 NFS 后需要实现额外 `Cornjob` 来清理 NFS 上的琐碎的文件。这样便增添了服务之间的关系复杂度。

好在在浏览 Drone 社区相关信息后发现了 Drone 发布了 1.6 版本。在 1.6 版本之后，Drone 为 K8S 实现了单独的 [Runner](https://github.com/drone-runners/drone-runner-kube)。

与之前执行 Job 的方式不同，新的执行方式是 Drone Server 接收到构建信息后会将构建信息存入基于内存的 `Queue` 中，runner 会向 Server 拉取构建信息，然后将构建信息解析成**一个** `Pod`，每个 `Step` 是一个 `Container`。为了保证 Step 执行的顺序行（因为 Pod 创建的时候 Container 执行是无序的），Kube-Runner 将每个还未轮到执行的 Step 的 `Image` 设置成了 `placeholder`，placeholder 是一个占位 image，该 image 不停地 sleep 不作任何操作。等到前置 Step 执行完成了，Kube-Runner 会将下个要执行的 Step 的 Image 由 placeholder 改成其对应的 image。通过上述操作来完成执行的顺序性。

![执行过程](/images/drone_virtual_kubelet/latest.jpg)

因为所有的 Step 都在一个 Pod 内，所以它们的数据就可以使用 EmptyDir 来共享。这便解决了之前的 HostPath 兼容性的问题。

### Privileged Context

在执行 CI 时，重要一步就是构建镜像。以 docker 为例。使用 docker 构建镜像就需要用到 `Docker Deamon`，Docker Deamon 可以使用宿主机上的或者可以在 Container 内启动一个 Docker Deamon，这里就形成了两种不同的模式，也就是 `Docker Outside Docker` 和 `Docker In Docker`。

因为 Docker Outside Docker 需要挂载宿主机的文件，所以自然在这种情况下是无法使用的。而 Docker In Docker 因为需要在容器内启动 Docker Deamon，所以需要 Privileged 权限，遗憾的是目前 ECI 中并不支持 Container 使用 Privileged Context。所有这两种方法在当前情况下都无法有效地构建镜像。

那么如何解决这种问题？

1. 通过一个服务将某台主机的 docker.sock 通过 TCP 的方式暴露出来，再通过 TCP 的方式访问 Docker Deamon。
2. 使用 [Kaniko](https://github.com/GoogleContainerTools/kaniko) 来构建镜像。

这里，我们选用了 kaniko 作为构建工具。

### Knaiko

![Knaiko](https://raw.githubusercontent.com/GoogleContainerTools/kaniko/master/logo/Kaniko-Logo.png)

> Knaiko 是从容器或 Kubernetes 集群内部的 Dockerfile 构建容器映像的工具。不依赖 Docker 守护程序，而是完全在用户空间中执行 Dockerfile 中的每个命令。这样就可以在无法轻松或安全地运行 Docker 守护程序的环境（例如标准Kubernetes集群）中构建容器映像。

Kaniko 执行器首先根据 Dockerfile 中的 `FROM` 一行命令解析基础镜像，按照 Dockerfile 中的顺序来执行每一行命令，在每执行完一条命令之后，会在用户目录空间中产生一个文件系统的快照，并与存储于内存中的上一个状态进行对比，若有改变，则将其认为是对基础镜像进行的修改，并以新层级的形式对文件系统进行增加扩充，并将修改写入镜像的元数据中。在执行完 Dockerfile 中的每一条指令之后， Kaniko 执行器将最终的镜像文件推送到指定的镜像仓库。

Kaniko 可以在不具有 `ROOT` 权限的环境下，完全在用户空间中执行解压文件系统，执行构建命令以及生成文件系统的快照等一系列操作，以上构建的过程完全没有引入 docker 守护进程以及CLI的操作。

到这，构建镜像问题也解决了。接下来就可以将整个构建调度到 ECI 了。

后续：[如何将Drone CI调度到Virtual Kubelet（二）](https://blog.domgoer.io/unknown)
