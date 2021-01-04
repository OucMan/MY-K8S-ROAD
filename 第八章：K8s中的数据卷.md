# 1. 数据卷概述

Volume（数据卷）主要解决了如下两方面问题：

* 数据持久性：通常情况下，容器运行起来之后，写入到其文件系统的文件暂时性的。当容器崩溃后，kubelet将会重启该容器，此时原容器运行后写入的文件将丢失，因为容器将重新从镜像创建。
* 数据共享：同一个Pod（容器组）中运行的容器之间，经常会存在共享文件/文件夹的需求。

在K8s中，Volume被定义为Pod的一部分，并和Pod共享相同的生命周期。这意味着在Pod启动时创建卷，并在删除Pod时销毁卷。因此在容器重新启动期间，卷的内容将保持不变，在重启容器之后，新容器可以识别出前一个容器写入卷的所有文件。另外，如果Pod中包含多个容器，那么这个卷可以同时被所有容器使用。

使用数据卷时，首先需要在Pod中定义一个数据卷，并将其挂载到容器的挂载点上。容器中的一个进程所看到（可访问）的文件系统是由容器的docker镜像和容器所挂载的数据卷共同组成的。Docker镜像将被首先加载到该容器的文件系统，在此之后任何数据卷都被挂载到指定的路径上。数据卷不能被挂载到其他数据卷上，或者通过引用其他数据卷。同一个Pod中的不同容器各自独立地挂载数据卷，即同一个容器组中的两个容器可以将同一个数据卷挂载到各自不同的路径上。

容器组、容器、挂载点、数据卷、存储介质的关系如下图：

# 2. 数据卷类型

K8s目前支持多达28种数据卷类型（其中大部分特定于具体的云环境如 GCE/AWS/Azure 等），如需查阅所有的数据卷类型，请查阅官方文档，https://kubernetes.io/docs/concepts/storage/volumes/

经常使用的数据卷的类型描述如下：

## 2.1 emptyDir

emptyDir是最基础的Volume类型。正如其名字所示，一个emptyDir Volume是Host上的一个空目录。emptyDir Volume对于容器来说是持久的，对于Pod则不是。当Pod从节点删除时，Volume的内容也会被删除。但如果只是容器被销毁而Pod还在，则Volume不受影响。也就是说：emptyDir Volume的生命周期与Pod一致。

Pod中的所有容器都可以共享Volume，它们可以指定各自的mount路径。下面通过例子来实践emptyDir。

### 2.1.1 emptyDir演示

这里模拟了一个producer-consumer场景。Pod有两个容器producer和consumer，它们共享一个Volume。producer负责往Volume中写数据，consumer则是从Volume读取数据。

*producer-consumer.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: producer-consumer
spec:
  containers:
  - image: busybox:latest
    name: producer
    volumeMounts:
    - mountPath: /producer_dir
      name: share-volume
    args:
    - /bin/sh
    - -c
    - echo "hello world" > /producer_dir/hello; sleep 30000;
  - image: busybox:latest
    name: consumer
    volumeMounts:
    - mountPath: /consumer_dir
      name: share-volume
    args:
    - /bin/sh
    - -c
    - cat /consumer/hello; sleep 30000;
    
  volumes:
  - name: share-volume
    emptyDir: {}
``` 
  
文件解析：

yaml文件创建了一个Pod，在Pod中创建了两个容器producer和consumer，一个emptyDir数据卷。producer容器将shared-volume挂载到/producer_dir目录，并通过echo将数据写到文件hello里。consumer容器将shared-volume mount到/consumer_dir目录，并通过过cat从文件hello读数据。

总结：emptyDir是Host上创建的临时目录，其优点是能够方便地为Pod中的容器提供共享存储，不需要额外的配置。它不具备持久性，如果Pod不存在了，emptyDir也就没有了。根据这个特性，emptyDir特别适合Pod中的容器需要临时共享存储空间的场景，比如前面的生产者消费者用例。

## 2.2 hostPath

hostPath Volume的作用是将Docker Host文件系统中已经存在的目录mount给Pod的容器。大部分应用都不会使用hostPath Volume，因为这实际上增加了Pod与节点的耦合，限制了Pod的使用。不过那些需要访问Kubernetes或Docker内部数据（配置文件和二进制库）的应用则需要使用hostPath，比如kube-apiserver和kube-controller-manager就是这样的应用。

如果Pod被销毁了，hostPath对应的目录还是会被保留，从这一点来看，hostPath的持久性比emptyDir强。不过一旦Host崩溃，hostPath也就无法访问了。

## 2.3 PersistentVolumeClaim

这是本章的重点内容。。。

### 2.3.1 存储卷PersistentVolume

首先介绍一下PV（PersistentVolume）出现的背景（可解决什么问题）：pod重建销毁，如用Deployment管理的pod，在做镜像升级的过程中，会产生新的pod并且删除旧的pod ，那新旧pod之间如何复用数据？多个pod之间，如果想要共享数据，应该如何去声明呢？同一个pod中多个容器想共享数据，可以借助Pod Volumes来解决；当多个pod想共享数据时，Pod Volumes就很难去表达这种语义；

为解决上述问题，K8s中又引入了PV，它可以将存储和计算分离，通过不同的组件来管理存储资源和计算资源，然后解耦pod和Volume之间生命周期的关联。这样，当把pod删除之后，它使用的PV仍然存在，还可以被新建的pod复用。

PV存储卷是集群中的一块存储空间，由集群管理员管理、或者由Storage Class（存储类）自动管理。PV和node一样，是集群中的资源（K8s集群由存储资源和计算资源组成）。

### 2.3.2 存储卷声明PersistentVolumeClaim

PersistentVolumeClaim（PVC存储卷声明）代表用户使用存储的请求。Pod容器组消耗node计算资源，PVC存储卷声明消耗PersistentVolume存储资源。Pod容器组可以请求特定数量的计算资源（CPU/内存）；PVC可以请求特定大小/特定访问模式（只能被单节点读写/可被多节点只读/可被多节点读写）的存储资源。

这里PV和PVS职责分离，PVC中只需用自己需要的存储size、access mode(单node独占还是多node共享？只读还是读写访问？)等业务真正关心的存储需求(不用关心存储实现细节)，PV和其后端的存储信息则由交给cluster admin统一运维和管控，安全访问策略更容易控制。两者通过kube-controller-manager中的Persisent中的PersisentVolumeController将PVC与合适的PV bound到一起，从而满足User对存储的实际需求。

### 2.3.3 PVC和PV之间的关系

PV可以看作可用的存储资源，PVC则是对存储资源的需求，两者关系如下：

* PV是集群中的存储资源，通常由集群管理员创建和管理
* PVC是使用存储资源的请求，通常由应用程序提出请求，并指定对应的访问策略以及需求的空间大小（或者StorageClass和需求的空间大小）
* PVC可以做为数据卷的一种，被挂载到Pod中使用

总结，PVC是用户对存储的需求，PV是真正存储的实现，Pod中通过挂载PVC来使用具体的PV，PVC与PV的绑定过程由K8s中的PersisentVolumeController实现。

搞清楚PV和PVC的关系，下面来看一下PV是如何创建的，PVC是如何与PV绑定的，解除绑定后PV又是怎么回收的等等问题。

### 2.3.4 PV与PVC的生命周期

首先Pv和PVC的生命周期与Pod的生命周期是相互独立的。

#### 2.3.4.1 资源供应（Provisioning）

K8s中有两种方式为PVC提供PV: 静态、动态。资源供应的结果就是创建好的PV。

*静态*

由集群管理员事先去规划这个集群中的用户会怎样使用存储，它会先预分配一些存储，也就是预先创建一些PV；然后用户在提交自己的存储需求（也就是PVC）的时候，K8s内部相关组件会帮助它把PVC和PV做绑定；之后用户再通过pod去使用存储的时候，就可以通过PVC找到相应的PV，它就可以使用了。

以使用阿里云存储静态创建PV为例，集群管理员首先通过阿里云文件存储控制台，创建NAS文件系统和挂载点，然后将将NAS文件系统大小，挂载点，以及PV的access mode,reclaim policy等信息添加到创建PV对象的yaml文件中，如下
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nas-csi-pv
spec:
  capacity:
    storage:  5Gi # 该volume的总容量大小
  accessModes:
  - ReadWriteMany  # 该volume可以被多个node上的pod挂载使用而且都具有读写特权
  persistentVolumeReclaimPolicy: Retain # 该volume使用后被release之后的回收策略
  csi:
    driver: nasplugin.csi.alibabacloud.com # 指定由什么volume plugin来挂载该volume(需要提前在node上部署)
    volumeHandle: data-id
    volumeAttributes:
      host: "***.cn-beijing.nas.aliyuncs.com"
      path: "/k8s"
      vers: "4.0"
```

接下来，User创建PVC声明存储需求，根据访问策略和所需空间大小，PersistentVolumeController会将PVC与PV绑定在一起。

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: nas-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    request:
      storage: 5Gi
```

最后创建Pod挂载该PVC

```
apiVersion: v1
kind: Pod
metadata:
  name: producer
spec:
  containers:
  - name:
    image: nginx
    ports:
    - containersPort: 80
    volumeMounts:
    - name: nas-pvc
      mountPath: /data
  volumes:
  - name: nas-pvc
    persistentVolumeClaim:
      claimName: nas-pvc
```



*动态*
```
```







## 2.4 其它类型

nfs\cephfs\configMap\secret等


# 3. 数据卷内子路径

有时候需要在同一个Pod的不同容器间共享数据卷。使用volumeMounts.subPath属性，可以使容器在挂载数据卷时指向数据卷内部的一个子路径，而不是直接指向数据卷的根路径。

下面的例子中，一个LAMP（Linux Apache Mysql PHP）应用的Pod使用了一个共享数据卷，HTML内容映射到数据卷的html目录，数据库的内容映射到了mysql目录：

```
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
        readOnly: false
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
        readOnly: false
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

在数据卷中创建mysql目录，然后挂载在mysql容器的/var/lib/mysql目录；在数据卷中创建html目录，然后挂载在php容器的/var/www/html目录

注：readOnly用来设置容器对数据卷的读写权限，readOnly为false表示可以读写，readOnly为true表示只可以读。


