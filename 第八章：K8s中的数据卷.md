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

***静态***

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

注：csi-nasplugin是为了在k8s中使用阿里云NAS所需的插件，需要提前部署在K8s集群中。

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

Pod创建后，便根据PVC来找到PV作为其存储。


***动态***

静态产生方式存在不足：首先需要集群管理员预分配，预分配其实是很难预测用户真实需求的，容易造成资源浪费。因此便出现了动态生产，集群管理员无须手工创建PV，而是通过StorageClass的设置对后端存储进行描述，标记为某种 "类型(Class)"。此时要求PVC对存储的类型进行声明，系统将自动完成PV的创建及PVC的绑定。PVC可以声明Class为""，说明该PVC禁止使用动态模式。yinci ,在配置有合适的StorageClass（存储类）且PVC关联了该StorageClass的情况下，K8s集群可以为应用程序动态创建PV。

StorageClass就像是一个创建PV的模板文件，它包含了创建某种具体类型PV所需的参数信息，User无需关系这些PV的细节，只需要通过PVC声明所需大小即可，K8s会结合PVC和StorageClass的信息动态创建PV对象。这样便在没有增加用户使用难度的同时也解放了集群管理员的运维工作，并做到了按需分配。

使用阿里云的云盘来说明动态Provisioning。

首先创建StorageClass

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-disk
provisioner: diskplugin.csi.alibabacloud.com
parameters:
  zoneId: cn-hangzhou-b
  regionId: cn-hangzhou
  fsType: ext4
  type: cloud_ssd
reclaimPolicy: Delete
```
provisioner表明创建PV的时候，应该用哪个存储插件来去创建。
parameters指定的一些细节参数。对于这些参数，用户是不需要关心的。
ReclaimPolicy指定回收策略，即使用方使用结束、Pod 及 PVC 被删除后，PV应该怎么处理，Delete的意思就是说当使用方Pod和PVC被删除之后，这个PV也会被删除掉。

注：csi-disk是为了在k8s中使用阿里云云盘所需要的插件，需要提前部署在K8s集群中。

目前集群中是没有PV的，动态提交一个PVC文件，如下

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: disk-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    request:
      storage: 30Gi
  storageClassName: csi-disk
```
该PVC指明存储大小需求是30G，访问策略为ReadWriteOnce，并指定storageClassName是csi-disk，就是刚才创建的storageclass，也就是说它指定要通过这个模板去生成PV。提交PVC的yaml后，过一会PV便创建成功。这个PV其实就是根据提交的PVC以及PVC里面指定的storageclass动态生成的，之后K8s会将生成的PV以及提交的PVC，就是disk-pvc做绑定。接下来创建Pod，该Pod挂载disk-pvc。

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
    - name: disk-pvc
      mountPath: /data
  volumes:
  - name: disk-pvc
    persistentVolumeClaim:
      claimName: disk-pvc
```

#### 2.3.4.2 PV与PVC绑定

假设用户创建了一个PVC存储卷声明，并指定了需求的存储空间大小以及访问模式。K8s master将立刻为其匹配一个PV存储卷，并将存储卷声明和存储卷绑定到一起。如果一个PV是动态提供给一个新的PVC，K8s master会始终将其绑定到该PVC。除此之外，应用程序将被绑定一个不小于（可能大于）其PVC中请求的存储空间大小的PV。一旦绑定，PVC将拒绝其他PV的绑定关系。PVC与PV之间的绑定关系是一对一的映射。

注：PVC将始终停留在未绑定unbound状态，直到有合适的PV可用。

#### 2.3.4.3 PV回收（ReclaimPolicy）

当用户不在需要其数据卷时，可以删除掉其PVC，此时其对应的PV将被集群回收并再利用。K8s集群根据PV中的reclaim policy（回收策略）决定在其被回收时做对应的处理。当前支持的回收策略有：Retained（保留）、Recycled（重复利用）、Deleted（删除）。

Retained：保留策略需要集群管理员手工回收该资源。当绑定的PVC被删除后，PV仍然存在，并被认为是“已释放”。但是此时该存储卷仍然不能被其他PVC绑定，因为前一个绑定的PVC对应容器组的数据还在其中。集群管理员可以通过如下步骤回收该PV：删除该PV，PV删除后，其数据仍然存在于对应的外部存储介质中（nfs、cefpfs、glusterfs等）；手工删除对应存储介质上的数据；手工删除对应的存储介质，也可以创建一个新的PV并再次使用该存储介质。

Recycled：再利用策略将在PV回收时，执行一个基本的清除操作rm -rf /thevolume/，并使其可以再次被新的PVC绑定。

Deleted：删除策略将从K8s集群移除PV以及其关联的外部存储介质（云环境中的 AWA EBS、GCE PD、Azure Disk 或 Cinder volume）。


#### 2.3.4.4 PV状态机

首先在创建PV对象后，它会处在短暂的pending状态；等真正的PV创建好之后，它就处在available状态。available状态意思就是可以使用的状态，用户在提交PVC之后，被K8s相关组件做完bound（即：找到相应的 PV），这个时候PV和PVC就结合到一起了，此时两者都处在bound状态。当用户在使用完PVC，将其删除后，这个PV就处在released状态，之后它应该被删除还是被保留呢？这个就会依赖ReclaimPolicy。当PV已经处在released状态下，它是没有办法直接回 available状态，也就是说接下来无法被一个新的PVC去做绑定。如果想把已经released的PV复用，一般有两种方式：新建一个PV对象，然后把之前的released的PV的相关字段的信息填到新的PV对象里面，这样的话，这个PV就可以结合新的PVC了；在删除pod之后，不要去删除PVC对象，这样给PV绑定的PVC还是存在的，下次pod使用的时候，就可以直接通过PVC去复用。


### 2.3.5 实现机制

CSI的全称是Container Storage Interface，是K8s实现外部存储插件的方式，CSI的实现大体可以分为两部分：第一部分是由K8s社区驱动实现的通用的部分，像csi-provisioner controller和csi-attacher controller，它们属于K8s的组件；另外一种是由云存储厂商实践的，对接云存储厂商的OpenApi，主要是实现真正的create/delete/mount/unmount存储的相关操作，比如csi-controller-server和csi-node-server。

按照上图来看一下，用户提交PVC和Pod时，K8s内部的处理流程

用户在提交PVC yaml的时候，首先会在集群中生成一个PVC对象，然后PVC对象会被csi-provisioner controller watch到，csi-provisioner会结合PVC对象以及PVC对象中声明的storageClass，通过GRPC调用 csi-controller-server，然后，到云存储服务这边去创建真正的存储，并最终创建出来PV对象。最后，由集群中的PV controller将PVC和PV对象做bound之后，这个PV就可以被使用了。


用户在提交Pod yaml之后，首先会被调度器调度选中某一个合适的node，之后该node上面的kubelet在创建Pod流程中会通过首先csi-node-server将之前创建的PV挂载到Pod可以使用的路径，然后kubelet开始create &&start Pod中的所有container。

可以看到使用CSI使用存储可大致分为三个步骤

* 第一个阶段(Create阶段)是用户提交完PVC，由csi-provisioner创建存储，并生成PV对象，之后PV controller将PVC及生成的PV对象做bound，bound之后，create阶段就完成了；
* 用户在提交Pod yaml的时候，首先会被调度选中某一个合适的node，等Pod的运行node被选出来之后，会被AD Controller watch到Pod选中的node，它会去查找pod中使用了哪些PV。然后它会生成一个内部的对象叫 VolumeAttachment对象，从而去触发csi-attacher去调用csi-controller-server去做真正的attache操作，attach操作调到云存储厂商OpenAPI。这个attach操作就是将存储attach到Pod将会运行的node上面。第二个阶段，attach阶段完成；
* 第三个阶段发生在kubelet创建Pod的过程中，它在创建Pod的过程中，首先要去做一个mount，这里的mount操作是为了将已经attach到这个node上面那块盘，进一步mount到Pod可以使用的一个具体路径，之后 kubelet才开始创建并启动容器。这就是PV+PVC创建存储以及使用存储的第三个阶段，mount阶段。


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


