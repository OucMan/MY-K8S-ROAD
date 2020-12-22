# 1. 控制循环

控制器模式最核心的就是控制循环的概念。在控制循环中包括了控制器，被控制的系统，以及能够观测系统的传感器，三个逻辑组件。外界通过修改资源的spec（期望状态）来控制资源，控制器通过比较资源的spec（期望状态）和status（现实状态），来计算出一个diff，diff会用来决定对系统执行怎样的控制操作，控制操作会使得系统产生新的输出，并被传感器以资源status形式上报。控制器的各个组件将都会是独立自主地运行，不断使系统向spec表示终态趋近。

![控制循环](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/%E6%8E%A7%E5%88%B6%E5%BE%AA%E7%8E%AF.png)

在K8s中，apiserver作为统一入口，任何对数据的操作都必须经过apiserver，因此实现控制循环需要依赖于apiserver与K8s其他组件之间的内部通信机制。常见的通信方式有：

* 客户端组件(kubelet, scheduler, controller-manager 等)轮询 apiserver；

* apiserver主动通知客户端；

但是这两种方式或者在性能、实时性或者在可靠性问题上存在弊端，因此K8s采用了一种新型的通信机制List-watch，List-watch作为K8s统一的异步消息处理机制，保证了消息的实时性，可靠性，顺序性，性能等等。


# 2. List-Watch机制

## 2.1 传感器

K8s中，控制循环中逻辑传感器主要由 Reflector、Informer、Indexer 三个组件构成。Reflector通过List和Watch K8s api server 来获取资源的数据，其中List用来在Controller重启以及Watch中断的情况下，进行系统资源的全量更新，而Watch则在多次List之间进行增量的资源更新。Reflector在获取新的资源数据后，会在Delta队列中塞入一个包括资源对象信息本身以及资源对象事件类型的Delta记录，Delta队列中可以保证同一个对象在队列中仅有一条记录，从而避免Reflector重新List和Watch的时候产生重复的记录。

Informer组件不断地从Delta队列中弹出delta记录，然后把资源对象交给indexer，让indexer把资源记录在一个Local Store中，缓存在默认设置下是用资源的命名空间来做索引的，并且可以被Controller Manager或多个 Controller所共享。之后，再把这个事件交给事件的回调函数。

可以看出里面涉及到两个存储空间，一个是Delta队列用来存储对资源的操作事件，一个是本地存储用来存储当前集群中资源的信息，该存储用来实现K8s组件更快地返回List/Get请求的结果、减少对Kubenetes API的直接调用。


![传感器](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/%E4%BC%A0%E6%84%9F%E5%99%A8.png)

## 2.2 控制器

控制循环中的控制器组件主要由事件处理函数以及worker组成，事件处理函数之间会相互关注资源的新增、更新、删除的事件，并根据控制器的逻辑去决定是否需要处理。对需要处理的事件，会把事件关联资源的命名空间以及名字塞入一个工作队列中，并且由后续的worker池中的一个Worker来处理，工作队列会对存储的对象进行去重，从而避免多个Woker处理同一个资源的情况。

Worker在处理资源对象时，一般需要用资源的名字来重新获得最新的资源数据，用来创建或者更新资源对象，或者调用其他的外部服务，Worker如果处理失败的时候，一般情况下会把资源的名字重新加入到工作队列中，从而方便之后进行重试。


## 2.3 系统

控制循环中的系统就是K8s中的各种资源


# 3. 实例：ReplicaSet扩容

这里通过一个简单的例子来说明一下控制循环的工作原理。

ReplicaSet是一个用来描述无状态应用的扩缩容行为的资源，ReplicaSet controler通过监听ReplicaSet资源来维持应用希望的状态数量，ReplicaSet中通过selector来匹配所关联的Pod，在这里考虑ReplicaSet rsA 的，replicas从2被改到3的场景，yaml文件如下

```
apiVersion： apps/v1
kind: ReplicaSet
metadata:
  name: rsa
  namespace: nsa
spec:
  replicas: 3 #Updated from 2 to 3
  selector:
    matchLabels:
      env: prod
  template:
    matedata:
      Labels:
        env: prod
    spec:
      containers:
      - images: nginx
        name: nginx
status:
  replicas: 3
```

![更新ReplicaSet](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/whole_step.jpg)


## 3.1 监听到ReplicaSet资源的变化

Reflector会watch到ReplicaSet资源的变化，发现ReplicaSet发生变化后，在delta队列中塞入了对象是rsA，而且类型是更新的记录。Informer一方面把新的ReplicaSet更新到Local Store中，并与Namespace nsA 作为索引。另外一方面，调用Update的回调函数，ReplicaSet控制器发现ReplicaSet发生变化后会把字符串的nsA/rsA字符串塞入到工作队列中，工作队列后的一个Worker从工作队列中取到了nsA/rsA这个字符串的 key，并且从Local Store中取到了最新的ReplicaSet数据。Worker通过比较ReplicaSet中spec和status里的数值，发现需要对这个ReplicaSet进行扩容，因此ReplicaSet的Worker创建了一个Pod，这个pod中的Ownereference取向了ReplicaSet rsA。

![监听到ReplicaSet资源的变化](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/reflector_step_1.png)

## 3.2 监听到Pod资源的变化

因为ReplicaSet要扩容，Worker会向api server发送创建Pod的命令，Reflector Watch到的Pod新增事件，在delta队列中额外加入了Add类型的delta记录，一方面把新的Pod记录通过Indexer存储到了缓存中，另一方面调用了ReplicaSet控制器的Add回调函数，Add回调函数通过检查pod ownerReferences找到了对应的ReplicaSet，并把包括ReplicaSet命名空间和字符串塞入到了工作队列中。ReplicaSet的Woker在得到新的工作项之后，从缓存中取到了新的ReplicaSet记录，并得到了其所有创建的Pod，因为ReplicaSet的状态不是最新的，也就是所有创建Pod的数量不是最新的。因此在此时ReplicaSet更新status使得spec和status达成一致。

![监听到Pod资源的变化](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/reflector_step_2.png)


# 4. 声明式API


