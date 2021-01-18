# 1. StatefulSet

## 1.1 概述

ReplicaSet通过一个模板创建多个Pod，这些副本除了名字和IP地址不同，其它的没有差别，同时这些副本共享持久卷。当某一个Pod由于一些原因被删除，然后被新的Pod替代，那么新的Pod将会有新的主机名和IP。对于有状态应用存在一种需求，即新的Pod和被替换的Pod具有相同的主机名和网络标识，并且具有专属的存储，这时候ReplicaSet就不能满足要求了，为此K8s中提供了StatefulSet控制器来管理Pod。

## 1.2 稳定的网络标识

StatefulSet创建的每一个Pod都有一个从零开始的顺序索引，这会体现在Pod的名称和主机名上，同样也会体现在Pod对应的固定存储上，比如StatefulSet A有三个Pod副本，那么Pod的名字分别是Pod A-0，Pod A-1，和Pod A-2，因此Pod的名字是可以预知的。

有状态的Pod和无状态的Pod还有一个差异性：无状态的Pod都是一样的，在需要的时候随便选一个就好，但是有状态的Pod，因为它们彼此是不同的，通常希望操作的是其中特定的一个，因此需要使用主机名来定位。基于这个原因，一个StatefulSet通常需要创建一个用来记录每个Pod网络标识的headless Service。通过这个Service，每个Pod将拥有独立的DNS记录，这样集群里它的伙伴或者客户端就可以通过主机名找到它。比如一个属于default命名空间，名为foo的控制服务，它的一个Pod名称为A-0，那么可以通过完整域名来访问它：a-0.foo.default.svc.cluster.local，也可以通过DNS服务，查找域名foo.default.svc.cluster.local对应的所有SRV记录，获取一个Statefulset中所有Pod的名称。

### 1.2.1 替换Pod

当StatefulSet管理的一个Pod实例消失后，StatefulSet会保证重启一个新的Pod实例替换它，新的Pod会拥有与之前Pod完全一致的名称和主机名（对于新的Pod运行在哪一个节点并不重要）。

### 1.2.2 扩缩容StatefulSet

扩容一个StatefulSet会使用下一个还没有用到的顺序索引值创建一个新的Pod实例。

缩容一个StatefulSet会最先删除最高索引值的实例，并且缩容在任何时候只会操作一个Pod实例。

注意，StatefulSet在有实例不健康的情况下是不允许缩容的。


## 1.3 稳定的专属存储

一个有状态的Pod需要拥有自己的存储，即使该有状态的Pod被重新调度，新的实例也必须挂载着相同的存储。StatefulSet采用的办法就是使用卷声明模板为每一个Pod都创建持久卷声明，这些持久卷声明会在创建Pod之前创建出来，绑定到一个Pod实例上。持久卷声明对应的持久卷可以是管理员提前创建出来，也可以是由持久卷的动态供应机制实时创建出来。

### 1.3.1 持久卷的创建和删除

扩容StatefulSet增加一个副本数时，会创建一个Pod和与之相关的一个或者多个持久卷声明

缩容StatefulSet时，只会删除一个Pod，而留下之前创建的声明。

### 1.3.2 重新将持久卷挂载到相同Pod的新实例上

因为缩容StatefulSet时会保留持久卷声明，所以在随后的扩容操作时，新的Pod实例会使用绑定在持久卷上的相同声明和其上的数据。


# 2. 演示

## 2.1 创建持久卷

对于每一个Pod实例，Statefulset都会创建一个绑定到一个持久卷上的持久卷声明，因为我们的集群中不支持持久卷的动态供应，因此手动创建。实验中Statefulset共有三个副本，因此需要创建三个持久卷。
*statefuleset-pv.yaml*
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-a
spec:
  capacity: 
    storage: 1Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/store1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-b
spec:
  capacity: 
    storage: 1Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/store2
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-c
spec:
  capacity: 
    storage: 1Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/store3
```

创建
```
master@k8s-master:~$ sudo kubectl apply -f statefuleset-pv.yaml 
persistentvolume/pv-a created
persistentvolume/pv-b created
persistentvolume/pv-c created
master@k8s-master:~$ sudo kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-a   1Mi        RWO            Retain           Available                                   25s
pv-b   1Mi        RWO            Retain           Available                                   24s
pv-c   1Mi        RWO            Retain           Available                                   24s
```

## 2.2 创建Service

在部署Statefulset之前，需要创建一个在有状态Pod之间提供网络标识的Headless Service。
*kubia-service-headless.yaml*
```
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  clusterIP: None
  selector:
    app: kubia
  ports:
  - name: http
    port: 80
    targetPort: 8080
```
创建
```
master@k8s-master:~$ sudo kubectl apply -f kubia-service-headless.yaml 
service/kubia created
master@k8s-master:~$ sudo kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   33d
kubia        ClusterIP   None         <none>        80/TCP    13s
```
## 2.3 创建应用容器镜像

我们重新创建一下由Statefulset管理的Pod中的容器镜像。首先应用如下
*app.js*
```
const http = require('http');
const os = require('os');
const fs = require('fs');

const dataFile = "/var/data/kubia.txt";

function fileExists(file) {
  try {
    fs.statSync(file);
    return true;
  } catch (e) {
    return false;
  }
}

var handler = function(request, response) {
  if (request.method == 'POST') {
    var file = fs.createWriteStream(dataFile);
    file.on('open', function (fd) {
      request.pipe(file);
      console.log("New data has been received and stored.");
      response.writeHead(200);
      response.end("Data stored on pod " + os.hostname() + "\n");
    });
  } else {
    var data = fileExists(dataFile) ? fs.readFileSync(dataFile, 'utf8') : "No data posted yet";
    response.writeHead(200);
    response.write("You've hit " + os.hostname() + "\n");
    response.end("Data stored on this pod: " + data + "\n");
  }
};

var www = http.createServer(handler);
www.listen(8080);
```
当应用接收到POST请求，将请求中body数据写入/var/data/kubia.txt文件中。当收到GET请求，返回主机名和/var/data/kubia.txt文件中的内容。

镜像信息如下
*Dockerfike*
```
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```
在两个Node节点上创建镜像
```
sudo docker build -f ./Dockerfile . -t luksa/kubectl-pet:v1
```

## 2.4 创建Statefulset

上述的资源创建后，就可以创建Statefulset了
*kubia-statefulset.yaml*
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kubia
spec:
  serviceName: kubia
  replicas: 2
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia-pet:v1
        ports:
        - name: http
          containerPort: 8080
        volumeMounts:
        - name: data
          mountPath: /var/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 1Mi
      accessModes:
      - ReadWriteOnce
```
主要注意的就是spec.serviceName设置为上面创建的Headless Service，即kubia，然后使用一个新的组件volumeClaimTemplates，定义了一个名为data的卷声明，Statefulset会依据这个模板为每一个Pod创建一个持久卷声明，做到每一个Pod有一个专属的存储。

```
master@k8s-master:~$ sudo kubectl create -f kubia-statefulset.yaml 
statefulset.apps/kubia created
master@k8s-master:~$ sudo kubectl get pod
NAME      READY   STATUS              RESTARTS   AGE
kubia-0   1/1     Running             0          4s
kubia-1   0/1     ContainerCreating   0          1s
master@k8s-master:~$ sudo kubectl get pod
NAME      READY   STATUS    RESTARTS   AGE
kubia-0   1/1     Running   0          9s
kubia-1   1/1     Running   0          6s
```
一个细节，ReplicationController或ReplicaSet会同时创建所有的Pod实例，而Statefulset会一次创建，即只有第一个Pod创建好，才开始创建第二个。

## 2.5 检查生成的有状态的Pod

```
master@k8s-master:~$ sudo kubectl get pod kubia-0 -o yaml
apiVersion: v1
kind: Pod
metadata:
...
spec:
  containers:
  - image: luksa/kubia-pet:v1
    imagePullPolicy: IfNotPresent
    name: kubia
    ports:
    - containerPort: 8080
      name: http
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/data
      name: data
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-vs6vn
      readOnly: true
  ...
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-kubia-0
  ...
```
可以看到Statefuleset根据卷声明模板创建持久卷声明，对于名为kubia的Statefuleset，第一个Pod名称为kubia-0（Statefulset名+序号），其对应的持久卷声明名为data-kubia-0（卷模板名+Pod名）。

继续查看持久卷声明
```
master@k8s-master:~$ sudo kubectl get pvc
NAME           STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-kubia-0   Bound    pv-a     1Mi        RWO                           16m
data-kubia-1   Bound    pv-b     1Mi        RWO                           12m
```
可以看到PVC已经与提前创建好的PV绑定

## 2.6 通过API 服务器与Pod通信

直接访问Pod，可以借助另一个Pod，然后在里面运行curl命令，或者使用端口转发。同时还可以借助API服务器通过代理的方式连接到指定的Pod。比如，如果想通过API服务器请求当前的kubia-0 Pod，可通过如下URL：
```
<apiServerHost>:<port>/api/v1/namespaces/default/pods/kubia-0/proxy/<path>
```

因为API服务器具有安全保证，即每次访问都需要证书和Token，因此可以借助kubectl proxy来与API服务器通信，这时使用localhost:8001代替上面的API服务器的IP和端口。

通信架构如下图



来，操作一下

首先打开kubectl proxy
```
master@k8s-master:~$ sudo kubectl proxy
Starting to serve on 127.0.0.1:8001
```

发送GET请求到kubia-1 Pod
```
master@k8s-master:~$ curl localhost:8001/api/v1/namespaces/default/pods/kubia-1/proxy/
You've hit kubia-1
Data stored on this pod: No data posted yet
```
返回的消息表明请求被正确收到，并在kubia-1 Pod中的应用中被正确处理。

发送POST请求到kubia-1 Pod
```
master@k8s-master:~$ curl -X POST -d "good, you are so good!" localhost:8001/api/v1/namespaces/default/pods/kubia-1/proxy/
Data stored on pod kubia-1
master@k8s-master:~$ curl localhost:8001/api/v1/namespaces/default/pods/kubia-1/proxy/
You've hit kubia-1
Data stored on this pod: good, you are so good!
```
没有问题，数据已经被存储，接下来再看看kubia-0 Pod
```
master@k8s-master:~$ curl localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
You've hit kubia-0
Data stored on this pod: No data posted yet
```
kubia-0 Pod中还没有数据，因此证明了两个Pod不是共享存储的关系，而是有自己专属的存储。

下面删除kubia-1 Pod，使得Statefuleset重新拉起一个kubia-1 Pod，我们来看存储下的内容是否还在

一个窗口执行
```
master@k8s-master:~$ sudo kubectl delete pod kubia-1
```
另一个窗口查看
```
master@k8s-master:~$ sudo kubectl get pod
NAME      READY   STATUS        RESTARTS   AGE
kubia-0   1/1     Running       0          14m
kubia-1   1/1     Terminating   0          14m
```
删除完毕后，Statefuleset重新拉起一个kubia-1 Pod
```
master@k8s-master:~$ sudo kubectl get pod
NAME      READY   STATUS    RESTARTS   AGE
kubia-0   1/1     Running   0          14m
kubia-1   1/1     Running   0          5s
```
再次访问kubia-1 Pod
```
master@k8s-master:~$ curl localhost:8001/api/v1/namespaces/default/pods/kubia-1/proxy/
You've hit kubia-1
Data stored on this pod: good, you are so good!
```
没有问题，数据还在，因此可以证明Statefulset会重启拉起一个和原来删除的Pod一模一样的Pod。

接着创建一个常规的Service，用来管理Pod
*kubia-service-public.yaml*
```
apiVersion: v1
kind: Service
metadata:
  name: kubia-public
spec:
  selector:
    app: kubia
  ports:
  - port: 80
    targetPort: 8080
```
利用代理，通过API服务器访问集群中的服务
```
master@k8s-master:~$ curl localhost:8001/api/v1/namespaces/default/services/kubia-public/proxy/
You've hit kubia-0
Data stored on this pod: No data posted yet
```
集群内部的客户端可以通过服务来存储或者读取数据，每一个请求会随机分配给一个Pod。

## 2.7 在Statefulset中发现伙伴节点

DNS中有一个SRV记录，它用来指向提供指定服务的服务器的主机名和端口号，K8s通过一个Headless Service创建SRV记录来指向Pod的主机名，我们可以查一下当前Headless Service kubia中有哪些记录。

创建一个临时的Pod，在其中运行DNS查询工具dig

命令
```
sudo kubectl run -it look --image=tutum/dnsutils --rm --restart=Never -- dig SRV kubia.default.svc.cluster.local
```
临时Pod的名字为look，镜像为tutum/dnsutils， --rm表示退出即删除Pod， --restart=Never表示不重启， 在Pod中执行dig SRV kubia.default.svc.cluster.local命令。

```
master@k8s-master:~$ sudo kubectl run -it look --image=tutum/dnsutils --rm --restart=Never -- dig SRV kubia.default.svc.cluster.local
If you don't see a command prompt, try pressing enter.
Error attaching, falling back to logs: unable to upgrade connection: container look not found in pod look_default

; <<>> DiG 9.9.5-3ubuntu0.2-Ubuntu <<>> SRV kubia.default.svc.cluster.local
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49291
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 3
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;kubia.default.svc.cluster.local. IN	SRV

;; ANSWER SECTION:
kubia.default.svc.cluster.local. 30 IN	SRV	0 50 8080 kubia-1.kubia.default.svc.cluster.local.
kubia.default.svc.cluster.local. 30 IN	SRV	0 50 8080 kubia-0.kubia.default.svc.cluster.local.

;; ADDITIONAL SECTION:
kubia-1.kubia.default.svc.cluster.local. 30 IN A 10.244.1.57
kubia-0.kubia.default.svc.cluster.local. 30 IN A 10.244.2.50

;; Query time: 201 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Mon Jan 18 09:26:07 UTC 2021
;; MSG SIZE  rcvd: 350

pod "look" deleted
```
可以看到Statefulset有两个Pod，主机名和端口号也看到是kubia-1.kubia.default.svc.cluster.local和kubia-0.kubia.default.svc.cluster.local， 8080和8080。根据主机名、IP地址以及端口号，然后Pod就可以根据这些信息找到伙伴Pod，进而通信。

## 2.8 更新Statefulset

更新Statefulset中的Pod镜像时，不会改变原有Pod的执行，如果想要使用新镜像，需要删除掉就Pod。

### 3. Statefulset处理失效节点

只有Statefulset明确地得到Pod已不在运行才会重新创建和原来一模一样的Pod，这样来保证集群中不会出现两个一模一样的Pod。但是如果节点和Master的网络断开了连接，这时候Master会将Pod状态设置为Unknown，超过一段时间后，Master节点会在节点上驱逐这个Pod,但是此时节点上的kubelet无法与Master节点通信，因此kubelet无法将Pod删除的消息告诉Master，从而导致在Master节点看来Pod一直是Unknown状态。

解决：强制删除Pod
```
sudo kubectl delete pod kubia-0 --force --grace-period 0
```

