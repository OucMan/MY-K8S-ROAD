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
    - c
    - sleep 10; exit 1
```
Pod的restartPolicy设置为OnFailure，默认为Always。

创建Pod
```
sudo kubectl create -f default-health-check.yaml
```



# 3. Readiness探测



# 4. 总结

Liveness指针是存活指针，它用来判断容器是否存活、判断Pod是否running。如果Liveness指针判断容器不健康，此时会通过kubelet杀掉相应的Pod，并根据重启策略来判断是否重启这个容器。如果默认不配置Liveness指针，则默认情况下认为它这个探测默认返回是成功的。

Readiness指针用来判断这个容器是否启动完成，即Pod的condition是否ready。如果探测的一个结果是不成功，那么此时它会从Endpoint上移除，也就是说从接入层上面会把该Pod 进行摘除，直到下一次判断成功，这个 Pod才会再次挂到相应的Endpoint之上。

一句话，Liveness指针探测Pod是否正常存在，Readiness指针探测Pod是否能够正常提供服务。
