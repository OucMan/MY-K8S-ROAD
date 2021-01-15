# 1. StatefulSet

## 1.1 概述

ReplicaSet通过一个模板创建多个Pod，这些副本除了名字和IP地址不同，其它的没有差别，同时这些副本共享持久卷。当某一个Pod由于一些原因被删除，然后被新的Pod替代，那么新的Pod将会有新的主机名和IP。对于有状态应用存在一种需求，即新的Pod和被替换的Pod具有相同的主机名和网络标识，并且具有专属的存储，这时候ReplicaSet就不能满足要求了，为此K8s中提供了StatefulSet控制器来管理Pod。

## 1.2 稳定的网络标识

StatefulSet创建的每一个Pod都有一个从零开始的顺序索引，这会体现在Pod的名称和主机名上，同样也会体现在Pod对应的固定存储上，比如StatefulSet A有三个Pod副本，那么Pod的名字分别是Pod A-0，Pod A-1，和Pod A-2，因此Pod的名字是可以预知的。


## 1.3 稳定的专属存储



# 2. 演示
