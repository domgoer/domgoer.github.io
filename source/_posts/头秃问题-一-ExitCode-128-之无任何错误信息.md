---
title: '头秃问题(一) ExitCode: 128 之无任何错误信息'
date: 2019-11-21 02:04:24
tags:
- issues
- ci/cd
- drone
- devops
- kubernetes
---

今天在使用 `Drone CI` 构建项目时出现一个异常奇怪的错误。

![error](/images/exitcode-128/drone-error.png)

在 `clone` 某分支代码时构建直接失败退出了。

首先我排查了 `rpc error`，发现这个错误出现的原因是正在获取 `Pod` 的 `Log` 的过程中，Pod 被删除了。

当时我就觉得很奇怪，就算因为 `Step` 执行失败要删除 Pod，那也会先停止 `getLog`，而不是在 getLog 的过程中直接删除 Pod。

