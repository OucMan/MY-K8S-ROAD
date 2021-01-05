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

```


### 2.1.2 指定目录创建

当DATA中使用的是目录，则其中的每一个文件都对应一个key:value，



### 2.1.3 指定键值对创建



## 2.2 ConfigMap的使用

# 3. Secret

# 4. ServiceAccount

# 5. Resource

# 6. SecurityContext

# 7. InitContainer


# 关于ens33网卡消失的小插曲

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


