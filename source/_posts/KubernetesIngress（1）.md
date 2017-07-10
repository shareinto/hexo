title: Kubernetes Ingress（1）简介
date: 2017-04-13 20:48:38
categories: kubernetes
tags:
  - kubernetes
  - linux
  - docker
------
## 为什么选择Ingress
kubernets的service和pod在集群内部有着集群网络统一管理和分配的ip地址，但是这些ip地址只有在集群内部可见，任何集群外部的节点都无法直接访问内部节点。显然，我们必须通过其它的渠道来访问。

目前，kubernetes提供了三种访问的方式：

- NodePort
- LoadBalance
- Ingress

其中，NodePort对于主机端口资原的要求非常高，无法应用于大规模的企业私有云，而LoadBalance方式只有在像GCE、Asure等等这些云服务提供商上面才能使用。因此，对于私有云可以采用的最佳入口方式非Ingress莫属。

## 什么是Ingress

Ingress是一系列允许入站链接到达集群内部服务的规则的集合。
未使用Ingress，外部网络无法到达内部服务
```go
    internet
        |
  ------------
  [ Services ]
```
Ingress的加入则使外部网络有了访问内部服务的途径
```go
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

Ingress可以为服务配置一系列的访问域名，负载均衡策略，SSL等等。

## Ingress Resource
Ingress和Pod、Servce等等类似，被定义为kubernetes的一种资源
它的一个简单的示例如下：
```code
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
spec:
  rules:
  - host: foo.bar.com
  - http:
      paths:
      - path: /testpath
        backend:
          serviceName: test
          servicePort: 80
```
这段Ingress的描述的意思是：将host为foo.bar.com且路径为/testpath的访问引导到test:80这个服务上面。

## Ingress Controller

本质上说Ingress只是存储在etcd上面一些数据，我们可以能过kubernetes的apiserver添加删除和修改ingress资源。那么真正让整个Ingress运转起来的一个重要组件是Ingress Controller。
然而，这个Controller并不像其它Controller一样作为kubernetes的核心组件在master启动的时候一起启动起来，它实际上是kubernetes的一个扩展，我们必须选择一个适合自己的Ingress Controller，或者自己去实现一个。

对于Ingress Controller官方的定义是这样子的：

>An Ingress Controller is a daemon, deployed as a Kubernetes Pod, that watches the apiserver's /ingresses endpoint for updates to the Ingress resource. Its job is to satisfy requests for ingress.

Ingress Controller作为一个守户进程，通过监听apiserver的ingresses资源变化，并且根据其指定的规则建立起外部网络访问内部服务的通道。对于官方描述的 deployed as a Kubernetes Pod，实际上是没办法运用到生产环境当中去的，这个我们在后面会提到这个问题，并且会有相应的解决方案。

## Ingress Controller的架构
![ingress-controller](http://7xlovv.com1.z0.glb.clouddn.com/ingress-controller.png)

上图展示了一个nginx ingress controller的部署架构图，ingress controller通过轮询监听apiserver的方式来获取ingress资源的变化，将ingress资源存储到本地缓存，并通知nginx进行相应的配置修改的加载。
ingress controller监控了ingress、service、endpoint、secret、node、configmap一系列资源，一旦资源发生了变化（包括增加、删除和修改），会立即通知backend，例如nginx等。
为了减少对apiserver的请求次数，nginx controllder会将每次请求在本地进行缓存，该缓存import了kubernetes提供的包"k8s.io/kubernetes/pkg/client/cache"。

## Ingress Controller的漂移问题
在官方定义的ingress controller，将它部署在kubernetes内部，以pod的方式存在kubernetes集群内部。既然是pod，那么就会存在漂移的问题，而作为外部网络的访问入口，我们是不允许这样的情况发生的。其中一种解决方案是通过VIP和服务发现来解决，但这无疑增加了整个系统的复杂度。
其实要解决漂移的问题很简单，我们只要将其部署在kubernetes集群外部，那么它就不受kubernetes的控制，自然而然就不会漂移了。
细心的读者可能会发现部署在外部的话，那么集群内外的网络通讯又会成为一个问题。笔者的集群环境的网络覆盖方案选择的是flannel，在每一个node上面初始化kubernetes环境的时候，都会一并装上flannel。
关于flannel的原理，这里有一篇文章分析得很详细[DockOne技术分享（十八）：一篇文章带你了解Flannel](http://dockone.io/article/618)

这是flannel的原理图：
![flannel](http://7xlovv.com1.z0.glb.clouddn.com/flannel.png)

通过该图我们可以看到通过docker0和flannel0这两块网卡打通了宿主机和集群内部的一个网络通道。

笔记在自己的节点上进行了验证
![flannel-if](http://7xlovv.com1.z0.glb.clouddn.com/flannel-if.png)

也就是说只要部署在该宿主机上的程序，都可以访问该节点上的任何docker容器，至于其它节点的docker容器，通过flanneld到节点的物理网卡，在flanned的时候数据包会被另外一种协议包装（如UDP、VxLAN、AWS VPC和GCE路由）成packet，该包到了另外一个节点的物理网卡再交由flanned进行解包，之后再通过虚拟的flannel和docker两块网卡路由到容器内部。

因此，我们可以以docker容器和方式或者直接以宿主机进程的方式部署我们的ingress controller。

## 总结
至此我们简单的介绍了kubernetes ingress 的整体结构的设计，还有ingress controller的实现机制以及部署问题等，在下一篇文章中我们会通过ingress controller的源码分析，详细讲解它的实现原理。
