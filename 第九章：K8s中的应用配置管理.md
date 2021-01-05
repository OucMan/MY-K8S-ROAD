# 1. 背景

K8s中提供了许多的资源实现应用的灵活配置，这里面的思想就是将配置和应用分离（解耦），各种各样的配置在K8s中以资源的形式存在，常见的配置资源类型如下：

* ConfigMap：ConfigMap存放容器中使用的可变配置（如环境变量），从而避免配置更改时重新编译容器镜像；
* Secret：Secret用来实现一些敏感信息的存储和使用，比如密码，token等；
* ServiceAccount：容器要访问集群自身，使用ServiceAccount实现身份认证；
* Resource：节点上运行的容器可能对资源（CPU\内存等）有一定的需求，通过Resource来控制；
* SecurityContext：节点上的容器是共享内核的，利用SecurityContext实现安全管控；
* InitContainer：业务容器启动之前的需要一个前置条件检验，比如容器启动前检测网络是否联通？这是可使用InitContainer；

# 2. ConfigMap

ConfigMap主要是管理一些可变配置信息，比如说应用的一些配置文件，或者说它里面的一些环境变量，或者一些命令行参数。它的好处在于它可以让一些可变配置和容器镜像进行解耦，这样也保证了容器的可移植性。假如把一些可变的配置写到镜像里面，每次这个配置需要变化的时候，都需要重新编译一次镜像，这肯定是不好的。

ConfigMap主要包括两部分：一个是ConfigMap元信息，如name和namespace；一个是data，data本质上就是一map，其中是键值对key:value，其中key是一个文件名，value是这个文件的内容。

## 2.1 ConfigMap的创建

推荐用kubectl这个命令来创建，它带的参数主要有两个：一个是指定name，第二个是DATA。其中DATA可以通过指定文件或者指定目录，以及直接指定键值对。
```
sudo kubectl create configmap [Name] [DATA]
```

### 2.1.1 指定文件创建

如果DATA使用的文件，则会创建一个key:value，文件名是key，文件内容就是value。

假设我们有一个json文件，里面的内容如下：

*cni-conf.json*
```
{
    "name": "cbr0",
    "type": "flannel",
    "delegate": {
      "isDefaultGateway": true
    }
}
```
使用如下命令创建名称为flannel-cfg的ConfigMap
```
sudo kubectl create configmap flannel-cfg --from-file=cni-conf.json
```
查看创建的ConfigMap的内容
```
master@k8s-master:~$ sudo kubectl get configmap flannel-cfg -o yaml
apiVersion: v1
data:
  cni-conf.json: |
    {
        "name": "cbr0",
        "type": "flannel",
        "delegate": {
          "isDefaultGateway": true
        }
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2021-01-05T06:38:26Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:cni-conf.json: {}
    manager: kubectl-create
    operation: Update
    time: "2021-01-05T06:38:26Z"
  name: flannel-cfg
  namespace: default
  resourceVersion: "232422"
  uid: 783a957b-cfd6-4f47-a72d-6e3339c72d31
```


### 2.1.2 指定目录创建

当DATA中使用的是目录，则其中的每一个文件都对应一个key:value

假设我们有一个文件夹my-nginx，里面有两个文件，分别是
*sleep-interval*
```
25
```
*nginx.conf*
```
server {
  listen    80;
  server_name   www.haha.com
}
```

使用如下命令创建名称为flannel-cfg的ConfigMap
```
sudo kubectl create configmap nginx-cfg --from-file=my-nginx
```
查看创建的ConfigMap的内容
```
master@k8s-master:~$ sudo kubectl get configmap nginx-cfg -o yaml
apiVersion: v1
data:
  nginx.conf: |
    server {
      listen    80;
      server_name   www.haha.com
    }
  sleep-interval: |
    25
kind: ConfigMap
metadata:
  creationTimestamp: "2021-01-05T06:46:31Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:nginx.conf: {}
        f:sleep-interval: {}
    manager: kubectl-create
    operation: Update
    time: "2021-01-05T06:46:31Z"
  name: nginx-cfg
  namespace: default
  resourceVersion: "233102"
  uid: 88eb54d5-1d04-4fdc-b123-a0e1af6b0b46
```

### 2.1.3 指定键值对创建

```
master@k8s-master:~$ sudo kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm
configmap/special-config created
master@k8s-master:~$ sudo kubectl get configmap special-config -o yaml
apiVersion: v1
data:
  special.how: very
  special.type: charm
kind: ConfigMap
metadata:
  creationTimestamp: "2021-01-05T06:48:20Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:special.how: {}
        f:special.type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-01-05T06:48:20Z"
  name: special-config
  namespace: default
  resourceVersion: "233257"
  uid: 8cfae80f-2d79-42e4-be44-7a87374f4de7
```

## 2.2 ConfigMap的使用

ConfigMap主要被Pod使用，一般用于挂载Pod用的配置文件，环境变量 ，命令行参数等。

### 2.2.1 用ConfigMap配置环境变量

创建Pod

*configmap-env-pod.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: cm-env-test
spec:
  restartPolicy: OnFailure
  containers:
  - image: luksa/kubia:v1
    name: test-container
    command: ["/bin/sh","-c","env"]
    env:
      - name: SPECILA_LEVEL_KEY
        valueFrom:
          configMapKeyRef:
            name: special-config
            key: special.how
```
创建Pod，并查看容器输出
```
master@k8s-master:~$ sudo kubectl logs cm-env-test 
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
NODE_VERSION=7.10.1
HOSTNAME=cm-env-test
YARN_VERSION=0.24.4
HOME=/root
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
NPM_CONFIG_LOGLEVEL=info
KUBERNETES_PORT_443_TCP_PROTO=tcp
SPECILA_LEVEL_KEY=very
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
```
可以看到*SPECILA_LEVEL_KEY=very*已经添加到容器的环境变量中。

### 2.2.2 一次传递ConfigMap的所有条目作为环境变量

如果ConfigMap有多个条目，为每一个条目单独设置环境变量有些麻烦，因此可以一次性为所有的ConfigMap条目设置环境变量

*configmap-env-once-pod.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: cm-env-test
spec:
  restartPolicy: OnFailure
  containers:
  - image: luksa/kubia:v1
    name: test-container
    command: ["/bin/sh","-c","env"]
    envFrom:
      - prefix: CONGIF_
        configMapRef:
          name: special-config
```

创建Pod，并查看容器日志
```
master@k8s-master:~$ sudo kubectl create -f configmap-env-once-pod.yaml 
pod/cm-env-test created
master@k8s-master:~$ sudo kubectl logs cm-env-test 
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
NODE_VERSION=7.10.1
HOSTNAME=cm-env-test
YARN_VERSION=0.24.4
HOME=/root
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
NPM_CONFIG_LOGLEVEL=info
KUBERNETES_PORT_443_TCP_PROTO=tcp
CONGIF_special.type=charm
CONGIF_special.how=very
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
```
可以看到special-config中的两个键值对special.type=charm和special.how=very都被添加进容器的环境变量中。

### 2.2.3 用ConfigMap配置管控命令行参数

创建Pod

*configmap-exec-pod.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: cm-env-test
spec:
  restartPolicy: OnFailure
  containers:
  - image: luksa/kubia:v1
    name: test-container
    command: ["/bin/sh","-c","echo ${SPECILA_LEVEL_KEY}"]
    env:
      - name: SPECILA_LEVEL_KEY
        valueFrom:
          configMapKeyRef:
            name: special-config
            key: special.how
```

执行创建Pod的命令，并查看容器的日志

```
master@k8s-master:~$ sudo kubectl create -f configmap-exec-pod.yaml 
pod/cm-exec-test created
master@k8s-master:~$ sudo kubectl logs cm-exec-test 
very
```

没有问题！

### 2.2.4 用ConfigMap挂载配置文件

创建Pod

*configmap-volume-pod.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: cm-volume-test
spec:
  containers:
    - name: test-container
      image: luksa/kubia:v1
      command: ["/bin/sh","-c","ls /etc/config"]
      valumeMonuts:
        - name: config-value
          mountPath: /etc/config
  volumes:
    - name: config-valume
      configMap:
        name: special-config
  restartPolicy: Never
```

执行创建Pod的命令，并且查看容器日志
```
master@k8s-master:~$ sudo kubectl create -f configmap-volume-pod.yaml 
pod/cm-volume-test created
master@k8s-master:~$ sudo kubectl logs cm-volume-test 
special.how
special.type
```
齐活儿

# 2.3 ConfigMap总结

* ConfigMap文件的大小。虽然说ConfigMap文件没有大小限制，但是在ETCD里面，数据的写入是有大小限制的，现在是限制在1MB以内；
* Pod引入ConfigMap的时候，必须是相同的Namespace中的ConfigMap;
* Pod引用ConfigMap时，假如这个ConfigMap不存在，那么这个Pod是无法创建成功的，其实这也表示在创建Pod前，必须先把要引用的ConfigMap创建好；
* 使用envFrom的方式把ConfigMap里面所有的信息导入成环境变量时，如果ConfigMap里有些key是无效的，比如key的名字里面带有数字，那么这个环境变量其实是不会注入容器的，它会被忽略。但是这个Pod本身是可以创建的；
* 只有通过K8s apiserver创建的Pod才能使用ConfigMap，比如说通过用命令行kubectl来创建的pod，但其他方式创建的Pod，比如说kubelet通过manifest创建的静态Pod，它是不能使用ConfigMap的。

**注意**：静态Pod就是直接在Node上使用kubelet使用本地文件系统或者WEB上的manifest创建的Pod，与普通的Pod，即通过K8s apiserver创建的Pod相比，它无法被RC、RS、Deployment、Job等数据对象关联起来，也不会被进行健康检查。当在某个Node需要长期运行一个应用，这时可以使用静态Pod。


# 3. Secret

# 4. ServiceAccount

# 5. Resource

# 6. SecurityContext

# 7. InitContainer


# 8. 关于ens33网卡消失的小插曲

今天打开集群环境，突然发现一个节点处于NotReady状态
```
master@k8s-master:~$ sudo kubectl get nodes
NAME         STATUS     ROLES                  AGE   VERSION
k8s-master   Ready      control-plane,master   20d   v1.20.0
k8s-node1    Ready      <none>                 20d   v1.20.0
k8s-node2    NotReady   <none>                 20d   v1.20.0
```
首先考虑的是网络的问题，就进入k8s-node2机器，去ping一下其它的机器，发现不通，判断就是网络的问题。然后使用ifconfig看看，发现ens33网卡不见了，这...，再搞回来吧，使用下面的命令
```
sudo dhclient ens33
```
再使用ifconfig命令看看，啊！大师兄回来了，
再看看集群节点的状态
```
master@k8s-master:~$ sudo kubectl get nodes
NAME         STATUS   ROLES                  AGE   VERSION
k8s-master   Ready    control-plane,master   20d   v1.20.0
k8s-node1    Ready    <none>                 20d   v1.20.0
k8s-node2    Ready    <none>                 20d   v1.20.0
```
啊~，全回来了！！！


