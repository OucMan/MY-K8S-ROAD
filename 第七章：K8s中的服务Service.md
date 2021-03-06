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

K8s为Service分配的IP是一个虚拟IP，也就是说集群中不存在一个实体与该IP匹配（直接ping这个IP不通），这个不像Pod的IP。那么客户端访问Service分配的虚拟IP以及端口号的流量是怎样到达对应的Pod的呢，答案就是K8s中的kube-proxy组件。K8s集群中的每个节点都运行了一个kube-proxy，负责将发往Service的流量引导至后端Pod，这里的流量指的是运输层及以上的流量。

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

Kubernetes Service支持的不同访问方式，这是通过spec.ServiceType字段指定，该字段的默认值是ClusterIP，其它可选值还包括：NodePort、LoadBalancer以及ExternalName。

## 4.1 ClusterIP

默认值，K8s为服务分配一个集群内部的IP地址，因此它只能被集群中的Pod访问，访问方式为ClusterIP:Port

## 4.2 NodePort

通过每一个节点上的的静态端口（NodePort）暴露Service（对于同一 Service，在每个节点上的节点端口都相同），NodePort的范围在初始化apiserver时可通过参数 --service-node-port-range 指定（默认是：30000-32767）。在创建Service时将使用该节点端口，如果该端口已被占用，则创建Service将不能成功。节点将该端口上的网络请求转发到对应的Service上。同时自动创建ClusterIP类型的访问方式，因此在集群内部通过ClusterIP:Port访问，在集群外部通过NodeIP:NodePort访问。

*my-service.yaml*
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
    nodePort: 90
```

注：
* nodePort是节点上监听的端口。
* port是ClusterIP上监听的端口。
* targetPort是Pod监听的端口。

Node和ClusterIP在各自端口上接收到的请求都会通过iptables转发到Pod的targetPort。nodePort不是请执行的，如果忽略它，K8s将选择一个随机的端口。还有个事就是，接收到客户端请求的节点与提供服务的Pod所在的节点有可能不是一个，这里使用的重定向机制还是利用Iptables实现的。

![NodePort](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/nodePort.png)


## 4.3 LoadBalancer

在支持外部负载均衡器的云环境中（例如 GCE、AWS、Azure 等），将.spec.type字段设置为LoadBalancer，Kubernetes将为该Service自动创建一个负载均衡器。负载均衡器的创建操作异步完成，可能要稍等片刻才能真正完成创建，负载均衡器的信息将被回写到Service的.status.loadBalancer字段。

*my-service.yaml*
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 9376
```

负载均衡器拥有自己独一无二的可公开访问的IP地址，并将所有连接重定向到服务。LoadBalancer类型的服务本质上是一个具有额外的基础设施提供的负载均衡器NodePort服务，因此除创建负载均群器外，集群内部的节点也会被分配一个节点端口，就像NodePort服务一样。

![LoadBalancer](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/LoadBalancer.png)

注意：接收外部连接的节点可能和最终提供服务的Pod所在的节点不是一个，这就有可能引起不必要的网络跳数，可以通过设置服务spec的externalTrafficPolicy字段为Local仅将外部通信重定向到接收连接的节点上运行的pod来阻止额外跳数。但是如果接收连接的节点没有本地Pod，那么连接将挂起（不会像不使用该设置意向，将其转发到随机的全局Pod），因此需要确保负载均衡器将连接转发到至少具有一个Pod的节点。

## 4.4 ExternalName

ExternalName类型的Service映射到一个外部的DNS name，而不是一个pod label selector。可通过spec.externalName字段指定外部DNS name。下面的例子中，Service my-service将映射到 someapi.somecompany.com：
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ExternalName
  externalName: someapi.somecompany.com
```

服务创建完成后，Pod可以通过my-service.default.svc.cluster.local域名（甚至my-service）连接到外部服务。ExternalName类型的Service仅在DNS级别实施——为服务创建简单的CNAME DNS记录。因此，连接到服务的客户端将直接连接到外部服务，完全绕过服务代理。


## 4.5 External IP

如果有外部可以通过IP路由到Kubernetes集群的一个或多个节点，Kubernetes Service可以通过这些external IPs进行访问。externalIP需要由集群管理员在K8s之外配置。在Service的定义中，ExternalIPs可以和任何类型的.spec.type一通使用。在下面的例子中，客户端可通过80.11.12.10:80（externalIP:port）访问my-service。
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
  externalIPs:
    - 80.11.12.10
```

# 5. 服务发现

集群中的Pod如何知道当前集群中存在哪些Service？Kubernetes支持两种主要的服务发现模式：环境变量和DNS。

## 5.1 环境变量

kubelet查找有效的Service，并针对每一个Service，向其所在节点上的Pod注入一组环境变量。如果要在Pod中使用基于环境变量的服务发现方式，必须先创建Service，再创建调用Service的Pod。否则，Pod中不会有该Service对应的环境变量。如果使用基于DNS的服务发现，您无需担心这个创建顺序的问题。

## 5.2 DNS

当我们配置完K8s集群环境时，就已经将DNS服务安装，如下图：
```
master@k8s-master:~$ sudo kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-74ff55c5b-cq7hh              1/1     Running   0          11d
kube-system   coredns-74ff55c5b-wx4nf              1/1     Running   0          11d
...
```

CoreDNS监听Kubernetes API上创建和删除Service的事件，并为每一个Service创建一条DNS记录。集群中所有的Pod都可以使用DNS Name解析到Service的IP地址。

例如，名称空间my-ns中的Service my-service，将对应一条DNS记录my-service.my-ns。 名称空间my-ns中的Pod可以直接nslookup my-service（my-service.my-ns也可以）。其他名称空间的Pod必须使用my-service.my-ns。my-service和my-service.my-ns都将被解析到Service的Cluster IP。

Kubernetes同样支持DNS SRV（Service）记录，用于查找一个命名的端口。假设my-service.my-ns Service有一个TCP端口名为http，则可以
```
nslookup _http__tcp_my-service.my-ns
```
以发现该Service的IP地址及端口http。

# 6. 利用Ingress暴露服务

## 6.1 Ingress概述

根据上面的描述，将Service暴露给集群外部的客户端有两种方法：NodePort和LoadBalancer，这两种方法都是在四层运输层上进行操作，还有一种利用Ingress暴露服务的方式，它将集群内部的 Service 通过HTTP/HTTPS方式暴露到集群外部，并通过规则定义HTTP/HTTPS的路由，因此Ingress是在七层应用层进行操作的，可以提供一下诸如cookie的会话亲和性等功能。具体地，Ingress作为Kubernetes的一种API对象，其中定义了URL路径和服务之间的关系，Ingress Controller监听节点上的特定端点，用来发现HTTP/HTTPS连接，并根据URL中的主机名和路径来解析出对应的服务，接着通过kube-proxy将请求转发到Service对应的任意一个Pod上。由此可见，Ingress只是Kubernetes中的一种配置信息，Ingress Controller才是监听端口，并根据Ingress上配置的路由信息执行HTTP路由转发的组件，因此必须先在K8s集群中安装了Ingress Controller，配置的Ingress才能生效。

Ingress Controller有多种实现可供选择，比较常用的有Traefic、 Nginx Ingress Controller for Kubernetes等，后面会实践安装 Nginx Ingress Controller。Ingress Controller的部署是DaemonSet，记在集群中的每一个节点都一个运行Ingress Controller的Pod。


## 6.2 利用Ingress访问Service流程

客户端首先对URL（如kubia.example.com）进行DNS查询，DNS服务器（或本地操作系统）返回了Ingress控制器的IP。客户端然后向Ingress控制器发送HTTP请求，并在Host头中指定URL，如kubia.example.com。Ingress控制器确定客户端尝试访问哪个服务，通过与该服务关联的Endpoint对象查看pod IP，并利用kube-proxy将客户端请求转发给其中一个Pod。

![Ingress](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/Ingress.png)

## 6.3 外网连接到Ingress控制器

想要利用Ingress，需要将URL映射到Ingress控制器IP，每一个集群节点都有一个Ingress控制器，因此要将URL映射到哪一个Ingress控制器IP？这里有两种常见的方案：暴露单worker节点和外部负载均衡器。

### 6.3.1 暴露单worker节点

步骤如下：
* 为K8s集群中的某一个worker节点配置外网IP地址Z.Z.Z.Z
* 将在Ingress中使用到的域名（假设是a.demo.kuboard.cn）解析到该外网IP地址Z.Z.Z.Z
* 设置合理的安全组规则（开放该外网IP地址80/443端口的入方向访问）

![单节点](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/single-node.png)

### 6.3.2 外部负载均衡器

步骤如下：
* 创建一个集群外部的负载均衡器，该负载均衡器拥有一个外网IP地址Z.Z.Z.Z，并监听80/443端口的TCP协议
* 将负载均衡器在80/443端口上监听到的TCP请求转发到K8s集群中所有（或某些）worker节点的80/443端口，可开启按源IP地址的会话保持
* 将在Ingress中使用到的域名（假设是a.demo.kuboard.cn）解析到该负载均衡器的外网IP地址Z.Z.Z.Z

![负载均衡器](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/balancer.png)

## 6.4 创建Ingress

### 6.4.1 使用Ingress暴露一个服务
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: kubia  # Ingress的名字，仅用于标识
spec:
  rules:                      # Ingress 中定义L7路由规则
  - host: kubia.example.com   # 根据 virtual hostname进行路由
    http:
      paths:                  # 按路径进行路由
      - path: /
        backend:
          serviceName: nginx-service  # 指定后端的Service为之前创建的nginx-service
          servicePort: 80
```
### 6.4.2 相同的Ingress暴露多个服务

rules和paths都是数组，因此可以包含多个条目，一个Ingress可以将多个主机和路径映射到多个服务。

*使用同一主机、不同路径，暴露多个服务*

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /kubia
        backend:
          serviceName: kubia
          servicePort: 80
      - path: /foo
        backend:
          serviceName: bar
          servicePort: 80
```

*将不同的服务映射到不同的主机上*

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
  - host: foo.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubia
          servicePort: 80
  - host: bar.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: bar
          servicePort: 80
```

# 7. Headless Service

使用Service服务提供稳定的IP地址，从而允许客户端连接到支持服务的每个Pod，到服务的每个连接都被转发到一个随机选择的Pod上。假如客户端想要连接到所有的Pod呢？K8s中提供了一种无头服务Headless Service来解决这个需求，Headless Service没有Cluster Ip，通过DNS查询Headless Service的记录，返回的是所有Pod的IP地址，而不是和常规Service一样，返回Cluster Ip。

要想创建Headless Service，只需要将spec.clusterIP设置为None，如
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  clusterIP: None
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
    nodePort: 90
```

尽管Headless服务看起来和常规服务不同，但是在客户角度它们并没有什么不同。即使使用Headless服务，客户也可以通过连接到服务的DNS名称来连接到Pod上，和常规服务一样。但是对于Headless服务，由于DNS返回了Pod的IP，客户端直接连接到该Pod，而不是通过kube-proxy。

注：Headless服务仍然提供跨Pod的负载均衡，但是是通过DNS轮询机制而不是通过kube-proxy。

# 8. Service演示

## 8.1 在集群中部署Pod

*run-my-nginx.yaml*
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: luksa/kubia:v1
        ports:
        - containerPort: 8080
          protocol: TCP
```
我们使用Deployment部署副本数量等于2的Pod，Pod中的容器使用luksa/kubia:v1（即，nginx服务器上运行了一个简单的App，详情见第二章）。

执行以下命令，部署Pod并检查运行情况：
```
master@k8s-master:~$ sudo kubectl apply -f run-my-nginx.yaml
[sudo] password for master: 
deployment.apps/my-nginx created
master@k8s-master:~$ sudo kubectl get pods -l run=my-nginx -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
my-nginx-96d4cc4cc-kzzt4   1/1     Running   0          26s   10.244.2.25   k8s-node2   <none>           <none>
my-nginx-96d4cc4cc-ll48t   1/1     Running   0          26s   10.244.1.25   k8s-node1   <none>           <none>
```
接着在Master节点上利用Pod IP来请求nginx的响应，如下：
```
master@k8s-master:~$ curl 10.244.2.25:8080
You've hit my-nginx-96d4cc4cc-kzzt4
master@k8s-master:~$ curl 10.244.1.25:8080
You've hit my-nginx-96d4cc4cc-ll48t
```
至此，利用Deployment部署Pod完成！

## 8.2 创建Service

根据上面的实验已经验证直接Pod IP可以直接访问nginx，但是假如某一个Pod终止，Deployment会重新拉起一个Pod，这是IP地址会发生变化，因此下面利用Service来解决IP地址动态变化的问题。

*nginx-svc.yaml*
```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    run: my-nginx
```
该yaml文件将创建一个Service：
* 通过label selector选取包含run: my-nginx标签的Pod作为后端Pod
* 该Service暴露一个端口80
* 该Service将80端口上接收到的网络请求转发到后端Pod的8080（targetPort）端口上，支持负载均衡

使用下面命令来创建Service，并查看Service状态
```
master@k8s-master:~$ sudo kubectl apply -f nginx-svc.yaml
[sudo] password for master: 
service/my-nginx created
master@k8s-master:~$ sudo kubectl get svc my-nginx
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
my-nginx   ClusterIP   10.106.223.82   <none>        80/TCP    21s
```
我们没有指定Service的类型，默认是Cluster IP，K8s为该Service分配了一个Cluster IP：10.106.223.82。

Service的后端Pod实际上通过Endpoints来暴露。K8s会持续检查Service的标签选择器，并将符合条件的Pod更新到与Service同名（my-nginx）的Endpoints对象。如果Pod终止了，该Pod将被自动从Endpoints中移除，新建的Pod将自动被添加到该Endpoint，执行下面的命令来看一下该Service中的Endpoints中的IP地址与上面获得的Pod地址是否相同。
```
master@k8s-master:~$ sudo kubectl describe svc my-nginx
Name:              my-nginx
Namespace:         default
Labels:            run=my-nginx
Annotations:       <none>
Selector:          run=my-nginx
Type:              ClusterIP
IP Families:       <none>
IP:                10.106.223.82
IPs:               10.106.223.82
Port:              <unset>  80/TCP
TargetPort:        8080/TCP
Endpoints:         10.244.1.25:8080,10.244.2.25:8080
Session Affinity:  None
Events:            <none>
```
也可以直接查看Endpoint资源
```
master@k8s-master:~$ sudo kubectl get ep my-nginx
NAME       ENDPOINTS                           AGE
my-nginx   10.244.1.25:8080,10.244.2.25:8080   6m12s
```
可以看到，Service已经创建成功，并且和后端的两个Pod绑定起来。

然后，我们在Master节点上利用Service的Cluster IP:port来访问Pod
```
master@k8s-master:~$ curl 10.106.223.82:80
You've hit my-nginx-96d4cc4cc-kzzt4
master@k8s-master:~$ curl 10.106.223.82:80
You've hit my-nginx-96d4cc4cc-ll48t
```
随机选择后端的Pod，酷炫！！！

## 8.3 Service发现

K8s支持两种方式发现服务：环境变量和DNS。

### 8.3.1 环境变量

针对每一个有效的Service，kubelet在创建Pod时，向Pod添加一组环境变量，我们进入一个Pod中查看里面的环境变量
```
master@k8s-master:~$ sudo kubectl exec my-nginx-96d4cc4cc-kzzt4 -- printenv | grep SERVICE
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
```
可以看到其中并没有与my-nginx Service相关的环境变量，这是因为我们先创建了Pod的副本，后创建了Service，导致my-nginx Service相关的环境变量没有被注入进去。我们删除已有的两个Pod，Deployment将重新创建Pod以替代被删除的Pod。此时因为在创建Pod时，Service已经存在，所以可以在新的Pod中查看到Service的环境变量被正确设置，详情如下：
```
master@k8s-master:~$ sudo kubectl delete pods -l run=my-nginx
pod "my-nginx-96d4cc4cc-kzzt4" deleted
pod "my-nginx-96d4cc4cc-ll48t" deleted
master@k8s-master:~$ sudo kubectl get pods -l run=my-nginx -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
my-nginx-96d4cc4cc-28gpr   1/1     Running   0          75s   10.244.1.26   k8s-node1   <none>           <none>
my-nginx-96d4cc4cc-lsj9s   1/1     Running   0          75s   10.244.2.26   k8s-node2   <none>           <none>
master@k8s-master:~$ sudo kubectl exec my-nginx-96d4cc4cc-28gpr -- printenv | grep SERVICE
KUBERNETES_SERVICE_PORT_HTTPS=443
MY_NGINX_SERVICE_HOST=10.106.223.82
MY_NGINX_SERVICE_PORT=80
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
```
可以看到my-nginx Service相关的环境变量已经被正确注入。

### 8.3.2 DNS

K8s提供了一个DNS cluster addon，即core DNS，可自动为Service分配DNS name。与环境变量方式不同（Pod与Service的启动顺序有关），集群中任何Pod中按Service的名称访问该Service。

首先创建busyboxplus容器的命令行终端
```
master@k8s-master:~$ sudo kubectl run curl --image=radial/busyboxplus:curl -i --tty
If you don't see a command prompt, try pressing enter.
[ root@curl:/ ]$ 
```
然后执行命令 nslookup my-nginx，以及curl my-nginx:80，结果表明可获得Nginx的响应，最后exit退出容器。
```
[ root@curl:/ ]$ nslookup my-nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      my-nginx
Address 1: 10.106.223.82 my-nginx.default.svc.cluster.local
[ root@curl:/ ]$ curl my-nginx:80
You've hit my-nginx-96d4cc4cc-lsj9s
[ root@curl:/ ]$ exit
Session ended, resume using 'kubectl attach curl -c curl -i -t' command when the pod is running
```
删除刚刚创建的curl测试Pod
```
sudo kubectl delete pod curl
```

## 8.4 暴露Service

上述创建的Service类型是Cluster IP，也就是只能被集群内部的客户端访问。想要将Service暴露到外部的网络，K8s提供了NodePort、LoadBalancer、Ingress方式，其中LoadBalancer需要云环境支持，本文不做过多阐述。我们首先演示NodePort，然后后续再演示Ingress方式。

首先删除上面创建的my-nginx Service
```
master@k8s-master:~$ sudo kubectl delete svc my-nginx
service "my-nginx" deleted
master@k8s-master:~$ sudo kubectl get svc my-nginx
Error from server (NotFound): services "my-nginx" not found
```
然后创建NodePort类型的Service

*nginx-svc-nodeport.yaml*
```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-nodeport
  labels:
    run: my-nginx-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
    protocol: TCP
  selector:
    run: my-nginx
```
spec.type设置为NodePort，nodePort设置为30123，接下来创建Service

```
master@k8s-master:~$ sudo kubectl apply -f nginx-svc-nodeport.yaml 
service/my-nginx-nodeport created
master@k8s-master:~/k8s-learning/resources/service$ sudo kubectl get svc my-nginx-nodeport
NAME                TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
my-nginx-nodeport   NodePort   10.97.95.66   <none>        80:30123/TCP   28s
```

接下来我们在宿主机（Windows 7）的浏览器上使用访问虚拟机IP:30123，比如10.10.10.148:30123，查看结果，得到响应You've hit my-nginx-96d4cc4cc-lsj9s，同理使用其他集群节点的IP地址访问也都可以得到Pod的响应。这就证明NodePort Service奏效了。

# 9. Ingress演示

## 9.1 获得nginx-ingress controller yaml文件
下载地址如下
```
https://kuboard.cn/install-script/v1.20.x/nginx-ingress.yaml
```
传输到Master节点下

## 9.2 安装nginx-ingress controller
```
master@k8s-master:~$ sudo kubectl apply -f nginx-ingress.yaml 
namespace/nginx-ingress created
serviceaccount/nginx-ingress created
secret/default-server-secret created
configmap/nginx-config created
clusterrole.rbac.authorization.k8s.io/nginx-ingress created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress created
daemonset.apps/nginx-ingress created
```
创建命名空间nginx-ingress，创建的组件都放到这个命名空间中。

serviceaccount、secret、configmap等资源我们后续章节会介绍。

nginx-ingress controller是使用DaemonSet控制器来创建的，因此会在每一个worker节点上创建一个nginx-ingress controller Pod。

查看对应的Pod是否创建成功

```
master@k8s-master:~$ sudo kubectl get pods -n nginx-ingress
NAME                  READY   STATUS    RESTARTS   AGE
nginx-ingress-2f2n2   1/1     Running   0          8m48s
nginx-ingress-92jsp   1/1     Running   0          8m48s
```
没有问题，至此nginx-ingress controller安装完毕！

## 9.3 安装Deployment、Service和Ingress

**Deployment**

*run-my-nginx.yaml*
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: luksa/kubia:v1
        ports:
        - containerPort: 8080
          protocol: TCP
```

**Service**

*nginx-svc-nodeport.yaml*
```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-nodeport
  labels:
    run: my-nginx-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
    protocol: TCP
  selector:
    run: my-nginx
```

**Ingress**

*nginx-ingress.yaml*
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress-for-nginx  # Ingress 的名字，仅用于标识
spec:
  rules:                      # Ingress 中定义 L7 路由规则
  - host: xxx.k8s.example.cn   # 根据 virtual hostname 进行路由（请使用您自己的域名）
    http:
      paths:                  # 按路径进行路由
      - path: /
        backend:
          serviceName: my-nginx-nodeport  # 指定后端的Service为之前创建的my-nginx-nodeport
          servicePort: 80
```

执行命令

```
sudo kubectl apply -f run-my-nginx.yaml
sudo kubectl apply -f nginx-svc-nodeport.yaml
sudo kubectl apply -f nginx-ingress.yaml
```
查看Ingress
```
master@k8s-master:~$ sudo kubectl get ingress -o wide
NAME                   CLASS    HOSTS               ADDRESS   PORTS   AGE
my-ingress-for-nginx   <none>   a.demo.kuboard.cn             80      6m48s
```

## 9.4 配置DNS

本实例采用暴露单worker节点的方式来使外部机器访问集群内服务，我们将k8s-node1的IP地址10.10.10.148暴露给外网。我们使用宿主机来代表集群外的机器，我们的宿主机是win7，因此在C:\Windows\System32\drivers\etc\HOST文件中增加如下一行：
```
10.10.10.148 a.demo.kuboard.cn
```
保存退出，然后后宿主机的浏览器上访问http://a.demo.kuboard.cn/

查看结果，结果为得到集群中Pod的响应，如下图所示：

![结果1](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/ingress-res1.png)

![结果2](https://github.com/OucMan/MY-K8S-ROAD/blob/main/pic/ingress-res2.png)

至此，完成利用Ingress访问Service！！！
