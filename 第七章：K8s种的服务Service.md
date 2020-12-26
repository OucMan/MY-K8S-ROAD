# 1. Service概述

## 1.1 为什么需要Service

K8s中应用是健壮的，但是Pod是脆弱的，也就是很可能因为各种原因发生故障而死掉（节点故障、容器内应用程序错误等原因），因此才会有需要控制器如Deployment来在Pod失败时重新拉起，以实现应用的健壮性。每一个Pod有自己的IP地址，因此对于Deployment等控制器而言，对应Pod集合是动态变化的，我们客户端如果能准确地知道当前提供服务的Pod的IP是什么的，从而能够实现通信呢，Service存在的意义，就是为了解决这个问题。

## 1.2 Service

K8s中Service是一种资源类型，通过定义一个Service，可以将符合Service指定条件的Pod作为可通过网络访问的服务提供给服务调用者。

Service是K8s中的一种服务发现机制：

* Pod有自己的IP地址
* Service被赋予一个唯一的dns name
* Service通过label selector选定一组Pod
* Service实现负载均衡，可将请求均衡分发到选定这一组Pod中

例如，假设有一个无状态的图像处理后端程序运行了3个Pod副本。这些副本是相互可替代的（前端程序调用其中任何一个都可以）。在后端程序的副本集中的Pod经常变化（销毁、重建、扩容、缩容等）的情况下，前端程序不应该关注这些变化。K8s通过引入Service的概念，将前端与后端解耦。

总结：Controller用来保证存在集群中的Pod符合期望，Service用来将Pod暴露出去，实现统一访问。

# 2. Service的创建

Service从逻辑上代表了一组Pod，具体是哪些Pod则是由label来挑选的。Service有自己的IP，而且这个IP是不变的。客户端只需要访问Service的IP，K8s则负责建立和维护Service与Pod的映射关系。无论后端Pod如何变化，对客户端不会有任何影响，因为Service没有变。

假设集群中存在一组Pod：每个Pod都监听9376 TCP端口；每个Pod都有标签app=MyApp，则Service的创建yaml文件如下

*my-service.yaml*
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

文件解析

* v1是Service的apiVersion。
* 指明当前资源的类型为Service。
* Service的名字为my-service。
* selector指明挑选那些label为app: MyApp的Pod作为Service的后端。
* 将Service的80端口映射到Pod的9376端口，使用TCP协议。

执行下面的命令创建Service
```
sudo kubectl create -f my-service.yaml
```
创建Service后，K8s将为该Service分配一个IP地址（ClusterIP 或集群内 IP），K8s将不断扫描符合该selector的Pod，并将最新的结果更新到与Service同名my-service的Endpoint对象中。假如客户端想要请求Pod提供的服务，则只需要向对应的Service的IP和Port发起请求，Service从自己的IP地址和port端口接收请求，并将请求映射到符合条件的Pod的targetPort。Service的默认传输协议是TCP，也支持UDP，http，https等协议，因此可以在Service的spec.ports字段中定义多个端口，不同的端口可以使用相同或不同的传输协议。



# 3. Service的类型


# 4. Service演示
