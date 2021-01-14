# 1. ServiceAccount

## 1.1 ServiceAccount概述

Pod与K8s API服务器交互时，会发送/var/run/secrets/kubernetes.io/serviceaccount/token文件内容来进行身份认证，这个文件是运行在Pod中应用程序的身份证明，被包含在ServiceAccount中。因此，每个Pod都与一个ServiceAccount相关联，当Pod中的应用程序与K8s API服务器交互时，会使用ServiceAccount中token连接K8s API服务器，身份认证插件会对ServiceAccount进行身份认证，并将ServiceAccount的用户名传回给API服务器内部，然后API服务器将这个用户名传给已配置好的授权插件，来决定该应用程序所尝试的操作是否被允许。

ServiceAccount和Pod，Secret等一样都是集群中的资源，它们作用在单独的命名空间中，每个命名空间中都有一个被自动创建的默认的ServiceAccount，如下
```
master@k8s-master:~$ sudo kubectl get sa
NAME      SECRETS   AGE
default   1         29d
```

当前命名空间只包含default ServiceAccount，其它相关的ServiceAccount可以在使用时添加。每个Pod都与一个ServiceAccount相关联，但是多个Pod可以使用同一个ServiceAccount，并且Pod只能使用同一个命名空间内的ServiceAccount。

## 1.2 ServiceAccount演示

每个命名空间都拥有一个默认的ServiceAccount，但是为了对Pod的资源访问进行权限限定，有时候需要创建额外的ServiceAccount，并与Pod进行绑定。

### 1.2.1 创建ServiceAccount

创建ServiceAccount十分容易，直接使用kubectl create serviceaccount name命令即可，比如
```
master@k8s-master:~$ sudo kubectl create serviceaccount foo
serviceaccount/foo created
```
创建了名为foo的ServiceAccount，使用kubectl describe命令查看ServiceAccount
```
master@k8s-master:~$ sudo kubectl describe serviceaccount foo
Name:                foo
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   foo-token-b6vr7
Tokens:              foo-token-b6vr7
Events:              <none>
```
可以看到已经创建了Token密钥，并且已经和ServiceAccount绑定。可以通过kubectl describe secret foo-token-b6vr7查看密钥里面的数据
```
master@k8s-master:~$ sudo kubectl describe secret foo-token-b6vr7
Name:         foo-token-b6vr7
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: foo
              kubernetes.io/service-account.uid: d1d689c9-33d8-4248-b06e-678724b48149

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  7 bytes
token:      eyJhbGci...
```
可以看到它包含了和默认的ServiceAccount相同的条目，即CA证书、命名空间和token，内容肯定是不同的。

### 1.2.2 绑定ServiceAccount与Pod

创建ServiceAccount后，需要将他们赋值给Pod，这是通过Pod定义文件中的spec.serviceAccountName字段上设置ServiceAccount的名称来进行绑定。

注意：Pod的ServiceAccount必须在Pod创建时进行设置，后续不能更改

Pod的定义文件如下
*curl-custom-sa.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: curl-custom-sa
spec:
  serviceAccountName: foo
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "9999999"]
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
```
在该Pod中，我们不使用默认的ServiceAccount，而是换成上面创建的foo ServiceAccount。

创建Pod并查看Pod容器内token的值
```
master@k8s-master:~$ sudo kubectl apply -f curl-custom-sa.yaml 
pod/curl-custom-sa created
master@k8s-master:~$ sudo kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
curl-custom-sa   2/2     Running   0          56s
master@k8s-master:~$ sudo kubectl exec -it curl-custom-sa -c main -- bash
root@curl-custom-sa:/# cat /var/run/secrets/kubernetes.io/serviceaccount/token 
eyJhbGciOi...
```
通过对比token的内容，果然Pod挂载的是创建的foo ServiceAccount。

### 1.2.3 利用自定义ServiceAccount与K8s API服务器交互

进入容器，访问ambassador容器监听的8001端口，
```
root@curl-custom-sa:/# curl localhost:8001/api/v1/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "279339"
  },
  "items": [
    {
      "metadata": {
        "name": "curl-custom-sa",
        "namespace": "default",
        "uid": "d8813720-dca9-402d-88d4-fc6c41df8a2e",
        "resourceVersion": "278630",
        "creationTimestamp": "2021-01-14T07:45:15Z",
        "annotations": {
        ...
```
没有问题，说明利用创建的foo ServiceAccount，Pod仍然可以和K8s API服务器交互。

注意：其实这是因为我们给了所有的ServiceAccount全部的权限，下面就来实践一下RBAC授权插件，来控制Pod的操作。

# 2. RBAC授权机制

## 2.1 RBAC概述

K8s API服务器可以配置一个授权插件来检查是否允许用户请求的动作执行，其中RBAC插件是标准。RBAC是基于角色的访问控制机制，简单的说就是一个用户可以拥有一个或者多个角色，每一个角色又对应着一个权限集合（即对哪些资源可以执行哪些操作）。在K8s中，资源可以是Namespace级别的资源（Pod, Service等），也可以是集群级别的资源（Node，PV等），操作类型为增删改查等，对应资源类型，还有两种类型的角色，Role和ClusterRole，Role对应的是命名空间的资源，即对命名空间的资源可以执行的动作集合，ClusterRole对应的是集群的资源，即对集群的资源可以执行的动作集合。定义好了角色外，还需要将用户与角色绑定，根据角色类型，K8s中有两种类型的绑定，即RoleBinding和ClusterRoleBinding，使用它们可以将上述的角色绑定到特定的用户、组或者ServiceAccounts上。

综上，角色定义了可以对哪些资源执行哪些操作，而绑定定义了谁可以做这些操作。

## 2.2 RABC演示

在进行第十章Pod与K8s API服务器交互的实验时，我们通过下面的命令将所有的ServiceAccount赋予集群管理员权限来绕过RBAC机制
```
sudo kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin --group=system:serviceaccounts
```
现在为了体现出RBAC的效果，我们先执行下面的命令来恢复RBAC
```
sudo kubectl delete clusterrolebinding permissive-binding
```
上面的两个语句，看完本章后就会门儿清了。

## 2.2.1 Role与RoleBinding演示



## 2.2.2 ClusterRole与ClusterRoleBinding演示






