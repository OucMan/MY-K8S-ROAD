# 1、K8s中的基本调度单位：Pod

## 1.2 Pod概述

Pod容器组是一个k8s中一个抽象的概念，用于存放一组container（可包含一个或多个container容器）。Pod是K8s调度的基本单位，是分配资源的一个单位，也就是说即使当一个Pod包含多个容器时，这些容器也总是会运行在同一个节点上（一个Pod绝不会跨越多个工作节点）。Pod中的容器具有“**超亲密关系**”，

什么叫做超亲密关系呢？大概分为以下几类：

* 比如说两个进程之间会发生文件交换，比如一个写日志，一个读日志；
* 两个进程之间需要通过 localhost 或者说是本地的 Socket 去进行通信，这种本地通信也是超亲密关系；
* 这两个容器或者是微服务之间，需要发生非常频繁的 RPC 调用，出于性能的考虑，也希望它们是超亲密关系；
* 两个容器或者是应用，它们需要共享某些 Linux Namespace。最简单常见的一个例子，就是我有一个容器需要加入另一个容器的 Network Namespace。这样我就能看到另一个容器的网络设备，和它的网络信息。

**一言以蔽之，Pod就是将具有“超亲密关系”的容器绑定在一起，共同调度。**

## 1.3 为什么需要Pod

容器被设计为每一个容器只运行一个进程（除非进程本身产生的子进程）。如果多个不相关进程运行在一个容器中，那么需要做很多的工作，来保证所有进程正常运行，并管理它们的日志（每个进程需要记录哪些信息）等情况，因此每个进程运行与自己的容器，多个进程分别起多个容器，关系紧密的进程，将他们对应的容器归属于同一个Pod，实现方便的管理。

换句话说，多个容器的组合就叫做 Pod。并且还有一个概念一定要非常明确，Pod 是Kubernetes分配资源的一个单位，因为里面的容器要共享某些资源，所以Pod也是 Kubernetes 的原子调度单位。


## 1.4 Pod的实现机制

Pod是一个抽象的概念，它没有具体的实体，它背后其实就是一组可以共享资源的容器，所以Pod要解决的核心问题就是如何让一个Pod里的多个容器之间最高效的共享某些资源和数据。因为容器之间原本是被Linux Namespace和cgroups隔开的，所以现在实际要解决的是怎么去打破这个隔离，然后共享某些事情和某些信息。这就是Pod的设计要解决的核心问题所在。具体的包括共享网络和共享存储。

### 1.4.1 共享网络

比如说现在有一个Pod，其中包含了一个容器A和一个容器B，它们两个就要共享 Network Namespace。在K8s里的解法是这样的：在启动Pod中的业务容器之前，首先提前起一个pause容器，整个Pod的Network Namespace就是这个pause容器的Network Namespace，其他所有容器都会通过Join Namespace的方式加入到pause container的Network Namespace中。所以说一个Pod里面的所有容器，它们看到的网络视图是完全一样的。即：它们看到的网络设备、IP地址、Mac地址等等，跟网络相关的信息，其实全是一份，这一份都来自于Pod第一次创建的这个pause容器。pause容器使用了一个非常小的镜像，大概 100~200KB 左右，是一个汇编语言写的、永远处于“暂停”状态的容器。它是整个Pod中第一个启动的容器，Pod生命周期与pause容器相同，而与其它容器无关，这就允许在K8s中去单独更新Pod里的某一个镜像的，即：做这个操作，整个Pod不会重建，也不会重启。

因为一个Pod中的容器共享网络空间，因此它们共享相同的IP地址和端口空间，这意味着同一个Pod中的容器进程不能绑定到相同的端口号，否则会导致端口冲突，而一个Pod中的容器之间通信可以通过localhost:port来完成。

![Pod共享网络](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/pod%E5%85%B1%E4%BA%AB%E7%BD%91%E7%BB%9C.png)


### 1.4.2 共享存储

共享存储十分简单，就是Pod中的容器通过挂载同一数据卷volume来实现共享。

## 1.5 K8s对Pod的操作

我们在搭建好的集群环境中来简单地实操一下，了解K8s对Pod的创建删除等操作。

### 1.5.1 利用YAML描述文件创建Pod

我们创建一个Pod，该Pod中只有一个容器，其中运行着nginx服务器以及对应的app，该应用监听的端口为8080，我们可以提前将该容器对应的镜像在node主机上创建一下，然后编写yaml文件，最后利用kubectl命令来完成Pod的创建。具体步骤如下。

#### 1.5.1.1 创建容器镜像

*Dockerfile*
```
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

*app.js*
```
const http = require('http');
const os = require('os');

console.log("Kubia server starting...");

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  response.end("You've hit " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

Dockerfile和app.js放置在一个目录，然后运行命令

```bash
sudo docker build -f ./Dockerfile . -t luksa/kubia:v1
```

查看镜像是否创建成功

```
node1@k8s-node1:~$ sudo docker images
REPOSITORY                                                       TAG                 IMAGE ID            CREATED             SIZE
luksa/kubia                                                      v1                  ca22f04a09c7        2 hours ago         660MB
k8s.gcr.io/kube-proxy                                            v1.20.0             10cc881966cf        9 days ago          118MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy   v1.20.0             10cc881966cf        9 days ago          118MB
k8s.gcr.io/coredns                                               1.7.0               bfe3a36ebd25        6 months ago        45.2MB
...

```
自此镜像创建完成，在另一个node主机上执行相同的操作，使之也得到相同的镜像（这一步可采用多种方式，比如docker镜像到处导入的操作）。

#### 1.5.1.2 编写yaml文件

*kubia-mannal.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual
spec:
  containers:
  - image: luksa/kubia:v1
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```

文件解析：

* apiVersion用来指明该描述文件遵循v1版本的K8s Api **
* kind用来指明本段代码描述的是一个Pod类型的资源 **
* metadata为该Pod的元数据，其中设置了该Pod的name为kubia-manual **
* spec用来阐述该Pod的期望状态，其中containers描述了Pod中的容器信息，在本例中，该容器使用的镜像为luksa/kubia:v1，名字是kubia，同时暴露出的端口是tcp:8080

以上的各个部分其实就是yaml文件的基本组成部门，创建其它的k8s资源时，也是采用相同的结构。




#### 1.5.1.3 创建Pod






### 1.5.2 停止和移除Pod

### 1.5.3 说明

本实践中，我们是直接创建的Pod，而没有与之相关的控制器（如Deployment\Job等），但是在实际应用中，一般都是通过创建如Deployment等控制器的方式来创建Pod，以实现Pod的灵活管理和高可用性。

# 2、容器设计模式

## 2.1 容器设计模式概述

设计模式的本质都是**解耦和重用**，容器设计模式也不例外。K8s中的Sidercar模式就是在一个Pod中通过定义一些专门的容器，来执行主业务所需要的一些辅助工作，比如利用Init Container负责将war包拷贝到共享目录，以后主业务容器中运行的Tomcat能够用起来。这种做法一个非常明显的优势就是在于其实将辅助功能从业务容器解耦了，所以Sidecar容器能被独立发布，并且更重要的是这个能力是可以重用的，即同样的一个监控Sidecar或者日志Sidecar，可以被全公司的人共用的。这就是设计模式的一个威力。

## 2.2 Sidecar应用场景

### 2.2.1 应用于日志收集

业务容器将日志写在一个 Volume 里面，而由于 Volume 在 Pod 里面是被共享的，所以日志容器 —— 即 Sidecar 容器一定可以通过共享该 Volume，直接把日志文件读出来，然后存到远程存储里面，或者转发到另外一个例子。现在业界常用的 Fluentd 日志进程或日志组件，基本上都是这样的工作方式。

### 2.2.2 代理容器

假如现在有个Pod需要访问一个外部系统，或者一些外部服务，但是这些外部系统是一个集群，那么这个时候如何通过一个统一的、简单的方式，用一个IP地址，就把这些集群都访问到？有一种方法就是：修改代码。因为代码里记录了这些集群的地址；另外还有一种解耦的方法，即通过Sidecar代理容器。

简单说，单独写一个这么小的Proxy，用来处理对接外部的服务集群，它对外暴露出来只有一个IP地址就可以了。所以接下来，业务容器主要访问Proxy，然后由Proxy去连接这些服务集群，这里的关键在于Pod里面多个容器是通过localhost直接通信的，因为它们同属于一个network Namespace，网络视图都一样，所以它们俩通信localhost，并没有性能损耗。

所以说代理容器除了做了解耦之外，并不会降低性能，更重要的是，像这样一个代理容器的代码就又可以被全公司重用了。


### 2.2.3 适配器

现在业务暴露出来的API，比如说有个API的一个格式是A，但是现在有一个外部系统要去访问我的业务容器，它只知道的一种格式是API B,所以要做一个工作，就是把业务容器怎么想办法改掉，要去改业务代码。但实际上，你可以通过一个Adapter帮你来做这层转换。

比如现在业务容器暴露出来的监控接口是/metrics，访问这个这个容器的metrics的这个URL就可以拿到了。可是现在，这个监控系统升级了，它访问的URL是/health，我只认得暴露出health健康检查的URL，才能去做监控，metrics不认识。那这个怎么办？那就需要改代码了，但可以不去改代码，而是额外写一个Adapter，用来把所有对health的这个请求转发给metrics就可以了，所以这个Adapter对外暴露的是health这样一个监控的URL，这就可以了，你的业务就又可以工作了。这样的关键还在于Pod之中的容器是通过localhost直接通信的，所以没有性能损耗，并且这样一个Adapter容器可以被全公司重用起来。



