# 1. 控制循环

控制器模式最核心的就是控制循环的概念。在控制循环中包括了控制器，被控制的系统，以及能够观测系统的传感器，三个逻辑组件。外界通过修改资源的spec（期望状态）来控制资源，控制器通过比较资源的spec（期望状态）和status（现实状态），来计算出一个diff，diff会用来决定对系统执行怎样的控制操作，控制操作会使得系统产生新的输出，并被传感器以资源status形式上报。控制器的各个组件将都会是独立自主地运行，不断使系统向spec表示终态趋近。

![控制循环](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/%E6%8E%A7%E5%88%B6%E5%BE%AA%E7%8E%AF.png)

在K8s中，apiserver作为统一入口，任何对数据的操作都必须经过apiserver，因此实现控制循环需要依赖于apiserver与K8s其他组件之间的内部通信机制。常见的通信方式有：

* 客户端组件(kubelet, scheduler, controller-manager 等)轮询 apiserver；

* apiserver主动通知客户端；

但是这两种方式或者在性能、实时性或者在可靠性问题上存在弊端，因此K8s采用了一种新型的通信机制List-watch，List-watch作为K8s统一的异步消息处理机制，保证了消息的实时性，可靠性，顺序性，性能等等。


# 2. List-Watch机制


# 3. 实例：ReplicaSet扩容


# 4. 声明式API
