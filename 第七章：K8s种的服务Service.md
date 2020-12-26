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
创建Service后，K8s将为该Service分配一个IP地址（ClusterIP 或集群内 IP），K8s将不断扫描符合该selector的Pod，并将最新的结果更新到与Service同名my-service的Endpoint对象中。假如客户端想要请求Pod提供的服务，则只需要向对应的Service的IP和Port发起请求，Service从自己的IP地址和port端口接收请求，并将请求映射到符合条件的Pod的targetPort。Service的默认传输协议是TCP，也支持UDP，http，https等协议，因此可以在Service的spec.ports字段中定义多个端口，不同的端口可以使用相同或不同的传输协议（这是必须为每个端口定义名字），如

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
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377   
```

# 3. kube-proxy

K8s为Service分配的IP是一个虚拟IP，也就是说集群中不存在一个实体与该IP匹配（直接ping这个IP不同），这个不像Pod的IP。那么客户端访问Service分配的虚拟IP以及端口号的流量是怎样到达对应的Pod的呢，答案就是K8s中的kube-proxy组件。K8s集群中的每个节点都运行了一个kube-proxy，负责将发往Service的流量引导至后端Pod，这里的流量指的是运输层及以上的流量。

目前kube-proxy支持三种代理模式：User space代理模式、Iptables代理模式（默认）、以及IPVS代理模式。

## 3.1 User space代理模式

工作流程：

* kube-proxy监听master节点以获得添加和移除Service/Endpoint的事件
* kube-proxy在其所在的节点（每个节点都有kube-proxy）上为每一个Service打开一个随机端口
* kube-proxy安装iptables规则，将发送到该Service的ClusterIP（虚拟IP）/Port的请求重定向到该随机端口
* 任何发送到该随机端口的请求将被代理转发到该Service的后端Pod上（kube-proxy从Endpoint信息中获得可用Pod）
* kube-proxy在决定将请求转发到后端哪一个Pod时，默认使用round-robin（轮询）算法，并会考虑到Service中的SessionAffinity的设定

也就是说，kube-proxy监听Service/Endpoint的变化，并为每一个Service分配一个端口X（该端口由kube-proxy请求），并在iptables中安装将发送到Service的请求重定向到端口X，进而kube-proxy接收到这个请求，然后由kube-proxy将该请求按照轮询算法转发给后端的Pod。因为流量转发的判定工作在kube-proxy中实现，而kube-proxy是用户空间的组件，因此该方案叫做User space代理模式。

以上面提到的图像处理程序为例。当后端Service被创建时，master节点为其分配一个虚拟IP地址（假设是10.0.0.1），并假设Service的端口是1234。集群中所有的kube-proxy都实时监听者Service的创建和删除。Service创建后，kube-proxy将打开一个新的随机端口，并设定iptables的转发规则（以便将该Service虚拟IP的网络请求全都转发到这个新的随机端口上），并且kube-proxy将开始接受该端口上的连接。当一个客户端连接到该Service的虚拟IP地址时，iptables的规则被触发，并且将网络报文重定向到kube-proxy自己的随机端口上。kube-proxy接收到请求后，选择一个后端Pod，再将请求以代理的形式转发到该后端Pod。

![User space代理模式](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/userspace-mode.png)

注：Service定义文件中的spec.sessionAffinity字段默认为None，如果设定为 "ClientIP"，则同一个客户端的连接将始终被转发到同一个Pod。


## 3.2 Iptables代理模式（默认）

工作流程：

* kube-proxy监听master节点以获得添加和移除Service/Endpoint的事件
* kube-proxy在其所在的节点（每个节点都有kube-proxy）上为每一个Service安装iptable规则
* iptables将发送到Service的ClusterIP/Port的请求重定向到Service的后端Pod上
  - 对于Service中的每一个Endpoint，kube-proxy安装一个iptable规则
  - 默认情况下，随机选择一个Service的后端Pod
  
以上面提到的图像处理程序为例。当Service被创建时，master节点为其分配一个虚拟IP地址（假设是10.0.0.1），并假设Service的端口是1234。集群中所有的kube-proxy都实时监听者Service的创建和删除。Service创建后，kube-proxy设定了一系列的iptables规则（这些规则可将虚拟IP地址映射到per-Service的规则）。per-Service规则进一步链接到per-Endpoint规则，并最终将网络请求重定向（使用DNAT）到后端 Pod。当一个客户端连接到该Service的虚拟IP地址时，iptables的规则被触发。一个后端Pod将被选中（基于session affinity或者随机选择），且网络报文被重定向到该后端Pod。与userspace proxy不同，网络报文不再被复制到userspace，kube-proxy也无需处理这些报文，且报文被直接转发到后端Pod。

![Iptables代理模式](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/iptable-mode.png)


## 3.3 IPVS代理模式

工作流程：
* kube-proxy监听master节点以获得添加和移除Service/Endpoint的事件
* kube-proxy根据监听到的事件，调用netlink接口，创建IPVS规则；并且将Service/Endpoint的变化同步到IPVS规则中
* 当访问一个Service时，IPVS将请求重定向到后端Pod

IPVS proxy mode基于netfilter的hook功能，与iptables代理模式相似，但是在一个大型集群中（例如，存在10000个Service）iptables的操作将显著变慢。IPVS的设计是基于in-kernel hash table执行负载均衡。因此，使用IPVS的kube-proxy在Service数量较多的情况下仍然能够保持好的性能。同时，基于IPVS的kube-proxy可以使用更复杂的负载均衡算法（最少连接数、基于地址的、基于权重的等）。

![IPVS代理模式](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/ipvs.png)


# 4. Service的类型


# 5. Service演示
