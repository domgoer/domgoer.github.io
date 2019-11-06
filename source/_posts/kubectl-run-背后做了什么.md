---
title: kubectl run 背后做了什么
date: 2019-08-29 16:18:17
tags:
- kubernetes
---

这篇文章随便写写，只写流程，不写原理

## 执行 kubectl run

1. 本地验证，确保不合法的请求（比如：创建不支持的资源或者格式不对等等）会快速失败，不会发给 api-server，减轻服务端压力

2. 准备向 api-server 发送 HTTP 请求，将修改后的数据进行序列化。但是 URI PATH 是什么呢？这里就依赖资源内的 apiVersion 这个值，再加上资源类型，kubectl 就能在 api 列表中找到应该发往的地址。api 列表可以通过 api-server 的 /apis 这个 URL 获取，获取之后会在本地缓存一份，提高效率。

3. api-server 肯定不会接受不合法的请求，所以 kubectl 还要在请求之前设置好认证信息。认证信息一般可以从 ~/.kube/config 中获取，它支持 4 种。

    * tls: 需要使用 x509 证书
    * token: 在 HEADER 中添加 Authorization
    * basic username password: 基本的账号密码认证
    * openid: 类似于 token，openid 由用户事先手动设置

4. 这时 api-server 已经成功接收到了请求。它会判断我们是不是有权限操作这个资源。那么怎么验证呢，在 api-server 启动的时候，可以通过参数 --authorization_mode 进行设置，这个值有 4 种。

    * webhook: 与集群外的 HTTPS 服务交互
    * ABAC: 静态文件定义的策略
    * RBAC: 动态配置的策略
    * Node: 每个 kubelet 只能访问自己节点上的资源

    如果配置了多种授权方式，只要其中一种能通过那么请求就能继续。

5. 授权通过了，但是现在仍然还不能够向 etcd 中写数据，还需要经过`准入控制链`这道关卡。准入控制链由 `Admission Controller` 控制。官方标准的准入控制链有近 10 个之多，而且支持自定义扩展。不同于授权，准入控制链一旦有一个验证失败，那么请求就会被拒绝。以下介绍三个准入控制器。

    * SecurityContextDeny: 禁止创建设置了 Security Context 的 Pod
    * ResourceQuota: 限制某个 Namespace 下总系统资源的占用量和资源的数量
    * LimitRanger: 限制某个 Namespace 下单个资源的占用量

6. 通过上面所有验证后，api-server 将 kubectl 提交的数据反序列化，然后保存到 etcd 中。

7. 数据已经保存到 etcd 中了，并且初始化逻辑也完成了，下面就需要 k8s 中的 `Controller` 来完成资源的创建了。各个 Controller 会监听各自负责的资源，比如 `Deployment Controller` 就会监听 `Deployment` 资源的变化。当 api-server 将资源保存到 etcd 后，Controller 发现了资源的变化，然后就根据变化类型会调用相应回调函数。每个 Controller 都会尽力将资源当前的状态逐步转化为 etcd 中保存的状态。

> 未完，还在慢慢写中。。。

## 总结

![流程](/images/kubectl_run/short.svg)