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

## 2.3 persistentVolumeClaim



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


