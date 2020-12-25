# 1. 概述

K8s使用Liveness和Readiness探测机制设置更精细的健康检查（Pod或者容器存在并且正在正常运行），本章详细描述这两种探测机制。

# 2. Liveness探测

Liveness指针是存活指针，用来判断一个Pod中容器是否处在存活状态（还在运行），可以为Pod中的每一个容器单独指定存活指针，如果探测失败K8s会根据重启策略来对容器进行操作。为了更好地理解Liveness探测存在意义，我们首先来看一下K8s默认的健康检测。

## 2.1 默认的健康检测

K8s默认的健康检查机制：每个容器启动时都会执行一个进程，此进程由Dockerfile的CMD或ENTRYPOINT指定。如果进程退出时返回码非零，则认为容器发生故障，K8s就会根据restartPolicy重启容器。

我们来演示一下效果

首先在worker节点上通过下面的命令来拉取busybox镜像
```
node1@k8s-node1:~$ sudo docker pull busybox
Using default tag: latest
latest: Pulling from library/busybox
Digest: sha256:bde48e1751173b709090c2539fdf12d6ba64e88ec7a4301591227ce925f3c678
Status: Image is up to date for busybox:latest
```

接着创建一个测试默认健康检测机制的Pod

*default-health-check.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: default-health-check
spec:
  restartPolicy: OnFailure
  containers:
  - image: busybox:latest
    name: health-check
    args:
    - /bin/sh
    - -c
    - sleep 10; exit 1
```
Pod的restartPolicy设置为OnFailure，默认为Always。
sleep 10; exit 1模拟容器启动10秒后发生故障。

创建Pod
```
sudo kubectl create -f default-health-check.yaml
```

过一会查看Pod的状态
```
master@k8s-master:~$ sudo kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
default-health-check   1/1     Running   2          48s
```

可看到容器当前已经重启了2次。在该例中，容器进程返回值非零，Kubernetes则认为容器发生故障，需要重启。但是，还有不少情况是发生了故障，但是进程并不会退回，比如访问Web服务器时显示500内部错误，可能是系统超载，也可能是资源死锁，此时httpd进程并没有异常退出，这时候默认的健康检测机制便不能捕获到这种异常，进而无法重启容器。K8s为了解决这个问题，便提出了Liveness探测。

## 2.2 Liveness指针

K8s中有三种探测容器的机制，分别是httpGet，Exec以及tcpSocket，下面来详细介绍每一种方式。

### 2.2.1 httpGet探测方式

它是通过发送http Get请求来进行判断的，当返回码是200-399之间的状态码时，标识这个应用是健康的。

*liveness-http-get-check.yaml*
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
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

文件解析

在原有创建Pod的基础上，增加livenessProbe的定义，指定类型为httpGet，并给清楚访问的路径和端口，以及initialDelaySeconds（pod启动延迟多久进行一次探测）和periodSeconds（探测的周期）。

### 2.2.2 Exec探测方式

它是通过执行容器中的一个命令来判断当前的服务是否是正常的，当命令行的返回结果是0，则标识容器是健康的。

*liveness-exec-check.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec-check
spec:
  restartPolicy: OnFailure
  containers:
  - image: busybox:latest
    name: health-check
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

文件解析

在原有创建Pod的基础上，增加livenessProbe的定义，指定类型为exec，并给清楚执行的命令，以及initialDelaySeconds（pod启动延迟多久进行一次探测）和periodSeconds（探测的周期）。


### 2.2.3 tcpSocket探测方式

它是通过探测容器的IP和Port进行TCP健康检查，如果这个TCP的链接能够正常被建立，那么标识当前这个容器是健康的。

*liveness-tcp-socket-check.yaml*
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
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

文件解析

在原有创建Pod的基础上，增加livenessProbe的定义，指定类型为tcpSocket，并给清楚探测的端口，以及initialDelaySeconds（pod启动延迟多久进行一次探测）和periodSeconds（探测的周期）。

## 2.3 探测结果

从探测结果来讲主要分为三种：

第一种是success，当状态是success的时候，表示container通过了健康检查，也就是Liveness probe是正常的一个状态；

第二种是Failure，Failure表示的是这个container没有通过健康检查，如果没有通过健康检查的话，那么此时就会进行相应的一个处理，Liveness 就是将这个pod进行重新拉起，或者是删除。

第三种状态是Unknown，Unknown是表示说当前的执行的机制没有进行完整的一个执行，可能是因为类似像超时或者像一些脚本没有及时返回，那么此时Liveness-probe 会不做任何的一个操作，会等待下一次的机制来进行检验。


## 2.4 Liveness探测的其它选项字段

successThreshold，它表示的是：当这个pod从探测失败到再一次判断探测成功，所需要的阈值次数，默认情况下是1次，表示原本是失败的，那接下来探测这一次成功了，就会认为这个pod是处在一个探针状态正常的一个状态；

failureThreshold，它表示的是探测失败的重试次数，默认值是3，表示的是当从一个健康的状态连续探测3次失败，那此时会判断当前这个pod的状态处在一个失败的状态。

注：可以通过--previous选项看查看崩溃容器的日志信息，即

```
sudo kubectl logs mypod --previous
```

# 3. Readiness探测

Readiness探测的配置语法与Liveness探测完全一样，探测类型和结果也相同，配置文件只是将liveness替换为了readiness，如下

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
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

不同之处在于探测失败后的行为：Liveness探测是杀死异常的容器并用新的正常容器替代他们来把持Pod正常工作，而Readiness探测则是将Pod设置为不可用，并在Service（后面的章节详细阐述）中对应的端点对象中剔除出去，不接收Service转发的请求。因此就绪探针确保只有准备好处理请求的Pod才能接受请求。

Liveness探测和Readiness探测是独立执行的，二者之间没有依赖，所以可以单独使用，也可以同时使用。用Liveness探测判断容器是否需要重启以实现自愈；用Readiness探测判断容器是否已经准备好对外提供服务。

# 4. 总结

Liveness指针是存活指针，它用来判断容器是否存活、判断Pod是否running。如果Liveness指针判断容器不健康，此时会通过kubelet杀掉相应的Pod，并根据重启策略来判断是否重启这个容器。如果默认不配置Liveness指针，则默认情况下认为它这个探测默认返回是成功的。

Readiness指针用来判断这个容器是否启动完成，即Pod的condition是否ready。如果探测的一个结果是不成功，那么此时它会从Endpoint上移除，也就是说从接入层上面会把该Pod 进行摘除，直到下一次判断成功，这个 Pod才会再次挂到相应的Endpoint之上。

一句话，Liveness指针探测Pod是否正常存在，Readiness指针探测Pod是否能够正常提供服务，Liveness探测可以告诉K8s什么时候通过重启容器实现自愈；Readiness探测则是告诉K8s什么时候可以将Pod加入到Service负载均衡池（后面章节会阐述）中，对外提供服务。另外探测机制是由承载Pod的节点上的Kubelet执行的，与Master节点上的控制器没有关系。

# 附录：应用故障排查-常见应用异常

## Pod停留在Pending
第一个就是pending状态，pending表示调度器没有进行介入。此时可以通过kubectl describe pod来查看相应的事件，如果由于资源或者说端口占用，或者是由于node selector造成pod无法调度的时候，可以在相应的事件里面看到相应的结果，这个结果里面会表示说有多少个不满足的node，有多少是因为CPU不满足，有多少是由于node不满足，有多少是由于tag打标造成的不满足。

## Pod停留在waiting
那第二个状态就是pod可能会停留在waiting 的状态，pod的states处在waiting的时候，通常表示说这个pod的镜像没有正常拉取，原因可能是由于这个镜像是私有镜像，但是没有配置Pod secret；那第二种是说可能由于这个镜像地址是不存在的，造成这个镜像拉取不下来；还有一个是说这个镜像可能是一个公网的镜像，造成镜像的拉取失败。

## Pod不断被拉取并且可以看到crashing
第三种是pod不断被拉起，而且可以看到类似像backoff。这个通常表示说pod已经被调度完成了，但是启动失败，那这个时候通常要关注的应该是这个应用自身的一个状态，并不是说配置是否正确、权限是否正确，此时需要查看的应该是pod的具体日志。

## Pod处在Runing但是没有正常工作
第四种pod处在running状态，但是没有正常对外服务。那此时比较常见的一个点就可能是由于一些非常细碎的配置，类似像有一些字段可能拼写错误，造成了yaml下发下去了，但是有一段没有正常地生效，从而使得这个pod处在running的状态没有对外服务，那此时可以通过apply-validate-f pod.yaml的方式来进行判断当前yaml是否是正常的，如果yaml没有问题，那么接下来可能要诊断配置的端口是否是正常的，以及Liveness或Readiness是否已经配置正确。


