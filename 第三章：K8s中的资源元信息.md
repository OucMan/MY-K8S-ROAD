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
注意：**查看标签时，命令后要加上--show-labels选项**

如果想要查看自己感兴趣的标签，可以使用-L选项指定它们，并将其显示在列中，比如

```
master@k8s-master:~$ sudo kubectl get pods -o wide -L env
NAME              READY   STATUS    RESTARTS   AGE     IP           NODE        NOMINATED NODE   READINESS GATES   ENV
kubia-manual-v2   1/1     Running   0          9m39s   10.244.2.6   k8s-node2   <none>           <none>            prod
```

另一种创建标签的方式，就是先创建好资源，然后将标签帖附上去，比如上文创建的kubia-manual-v2 Pod，我们利用下面的命令来新增标签
```
master@k8s-master:~$ sudo kubectl label pod kubia-manual-v2 app=nginx
pod/kubia-manual-v2 labeled
master@k8s-master:~$ sudo kubectl get pods -o wide --show-labels
NAME              READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES   LABELS
kubia-manual-v2   1/1     Running   0          17m   10.244.2.6   k8s-node2   <none>           <none>            app=nginx,creation_method=manual,env=prod
```

### 1.1.2 修改标签

修改现有标签时，需要使用--overwrite选项
```Bash
master@k8s-master:~$ sudo kubectl label pod kubia-manual-v2 env=debug --overwrite
pod/kubia-manual-v2 labeled
master@k8s-master:~$ sudo kubectl get pods -o wide --show-labels
NAME              READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES   LABELS
kubia-manual-v2   1/1     Running   0          23m   10.244.2.6   k8s-node2   <none>           <none>            app=nginx,creation_method=manual,env=debug
```

### 1.1.3 利用标签实现资源查询

上面的内容，我们将标签附加到资源上，其目的就是帮助用户在后续使用标签选择器对资源进行查询和组合。标签选择器使用-l这个选项来进行指定的。上文说道标签选择器有两种类型，一种是相等型，对应的运算就是=和!=，另一种是集合型，对应的运算是in和not in。

```
master@k8s-master:~$ sudo kubectl get pods --show-labels -l env=debug
NAME              READY   STATUS    RESTARTS   AGE   LABELS
kubia-manual-v2   1/1     Running   0          50m   creation_method=manual,env=debug
```

```
master@k8s-master:~$ sudo kubectl get pods --show-labels -l env!=debug
No resources found in default namespace.
```

```
master@k8s-master:~$ sudo kubectl get pods --show-labels -l 'env in (debug, proc)'
NAME              READY   STATUS    RESTARTS   AGE   LABELS
kubia-manual-v2   1/1     Running   0          51m   creation_method=manual,env=debug
```

```
master@k8s-master:~$ sudo kubectl get pods --show-labels -l 'env notin (debug, proc)'
No resources found in default namespace.
```
当有多个筛选条件时，条件之间使用逗号隔开

```
master@k8s-master:~$ sudo kubectl get pods --show-labels -l 'env in (debug, proc)',creation_method=manual 
NAME              READY   STATUS    RESTARTS   AGE   LABELS
kubia-manual-v2   1/1     Running   0          53m   creation_method=manual,env=debug
```

还可以直接筛选出具有某一标签的资源，如下命令展示了包含env标签的所有pod
```
master@k8s-master:~$ sudo kubectl get pods --show-labels -l env
NAME              READY   STATUS    RESTARTS   AGE   LABELS
kubia-manual-v2   1/1     Running   0          55m   creation_method=manual,env=debug
```

注：发现规律，**当使用集合型标签选择器时，条件要使用引号括起来，而当使用相等型标签选择器时，则不需要引号。**

### 1.1.4 删除标签
```
master@k8s-master:~$ sudo kubectl label pod kubia-manual-v2 app-
pod/kubia-manual-v2 labeled
master@k8s-master:~$ sudo kubectl get pods -o wide --show-labels
NAME              READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES   LABELS
kubia-manual-v2   1/1     Running   0          29m   10.244.2.6   k8s-node2   <none>           <none>            creation_method=manual,env=debug
```

### 1.1.5 标签总结

标签本质上就是键值对，用来标记K8s资源，结合标签选择器可以灵活地实现资源的选择和组合。

## 1.2 Annotations

注解Annotations，一般是系统或者工具用来存储资源的非标示性信息，可以用来扩展资源的spec/status的描述，本质上Annotations也是键值对，只是与标签不同的是，注解不是为了保存标识信息而存在的，它不能像标签一样用于对资源进行分组，因此不存在注解选择器。还有一个显著的区别就是，在创建注解的时候，值那部分使用引号括起来，可以当做是字符串，该字符串中包含空格，逗号等等，这在标签中是不可以的。操作注解的命令时kubectl annotate，如

```
sudo kubectl annotate pod kubia-manual-v2 my-annotate=‘my annotate,ok’
```
其余的添加、修改和删除，和标签的命令格式相同。


## 1.3 OwnerReference

资源的OwnerReference用来记录该资源的创建者，比如Pod控制器会创建对应的归属Pod，例replicaset控制器（后面章节会阐述）在操作中会创建Pod，被创建Pod的Ownereference就指向了创建Pod的replicaset，Ownereference使得用户可以方便地查找一个创建资源的对象，另外，还可以用来实现级联删除的效果。

总结：OwnerReference的作用是方便反向查找创建资源的对象以及方便进行级联删除。

## 1.4 总结

Kubernetes资源对象中的元数据部分，主要包括了用来识别资源的标签：Labels， 用来描述资源的注解：Annotations， 用来描述多个资源之间相互关系的OwnerReference。这些元数据在K8s运行中有非常重要的作用。
