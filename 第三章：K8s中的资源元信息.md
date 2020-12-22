# 1. K8s中的资源元信息

K8s的资源对象组成主要包括了Metedate、Spec、以及Status三部分。其中Metedate为资源的元数据部分，用来保存资源的某些固有属性，Spec部分用来描述期望的状态，Status部分用来描述观测到的状态。

Metedate主要包括了用于识别资源的标签（Label）、用来描述资源的注解（Annotation）、以及用来描述多个资源之间相互关系的OwnerReference。

本章主要对Metedate中的三种类型的数据进行总结并对常见的一些操作进行描述。

## 1.1 Label

标签是一种简单却功能强大的K8s特性，不仅可以用来组织Pod，也可以用来组织所有其它的K8s资源。本质上，**标签是可以附加到资源上的任意键值对**，比如Pod上具有标签app=nginx，它的主要作用就是用来筛选和组合资源。在筛选和组合过程中，需要使用到标签选择器Seletor，最常见的Selector就是相等型Selector和集合型Selector，多个筛选条件使用“,”来连接，相当于逻辑与的关系。

### 1.1.1 创建标签

在创建资源时指定标签，比如下面的yaml文件中，在创建Pod时，创建了两个标签，分别是creation_method和env

```
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual-v2
  labels:
    creation_method: manual
    env: prod
spec:
  containers:
  - image: luksa/kubia:v1
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```

运行命令
```Bash
sudo kubectl create -f kubia-manual-with-labels.yaml
```
然后查看Pod和标签的创建情况
```
master@k8s-master:~$ sudo kubectl get pods -o wide --show-labels
NAME              READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES   LABELS
kubia-manual-v2   1/1     Running   0          52s   10.244.2.6   k8s-node2   <none>           <none>            creation_method=manual,env=prod
```
注意：##查看标签时，命令后要加上--show-labels选项##


### 1.1.2 修改标签

### 1.1.3 查询标签

### 1.1.4 删除标签

### 1.1.5 标签总结

## 1.2 Annotations


## 1.3 OwnerReference


