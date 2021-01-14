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

创建两个命名空间foo和bar
```
sudo kubectl create ns foo
sudo kubectl create ns bar
```
创建Pod
```
master@k8s-master:~$ sudo kubectl apply -f kubectl-proxy.yaml -n foo
deployment.apps/test created
master@k8s-master:~$ sudo kubectl get pods -n foo
NAME                    READY   STATUS    RESTARTS   AGE
test-7648b6b45f-tsg9g   1/1     Running   0          5s
master@k8s-master:~$ sudo kubectl apply -f kubectl-proxy.yaml -n bar
deployment.apps/test created
master@k8s-master:~$ sudo kubectl get pods -n bar
NAME                    READY   STATUS    RESTARTS   AGE
test-7648b6b45f-lzz7k   1/1     Running   0          7s
```
其中kubectl-proxy.yaml的内容为
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
spec:
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: test
        image: luksa/kubectl-proxy:1.6.2
```
进入foo命名空间下的test-7648b6b45f-tsg9g Pod，并尝试访问K8s API服务器
```
master@k8s-master:~$ sudo kubectl exec -it test-7648b6b45f-tsg9g -n foo -- sh
/ # curl localhost:8001/api/v1/namespaces/foo/services
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "services is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"services\" in API group \"\" in the namespace \"foo\"",
  "reason": "Forbidden",
  "details": {
    "kind": "services"
  },
  "code": 403
}
```
根据响应结果可以看到，请求被拒绝，RBAC插件起了作用，同理bar命名空间下的Pod也是一样的。

上述实验表明，在RBAC插件的限制下，Pod使用默认的ServiceAccount不允许列出同一命名空间下的Service列表，下面我们通过创建Role和RoleBinding来向Pod开放这个权限。

Role定义如下
*service-reader.yaml*
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: foo
  name: service-reader
rules:
- apiGroups: [""]
  verbs: ["get", "list"]
  resources: ["services"]
```

Role和RoleBinding都需要在命名空间中生效，因此需要指定命名空间，因为资源是在foo命名空间中，因此Role的命名空间也应该是foo，资源设置为Services，注意在指定资源时必须使用复数的形式，允许的动作有get和list，因为Service是核心apiGroup的资源，所以没有apiGroup名，就是“”；可以看到rules是一个复数，所以可以设定多种规则。本例里允许访问所有服务资源，还可以利用resourceNames字段指定服务实例的名称来限制对服务实例的访问。

创建角色
```
master@k8s-master:~$ sudo kubectl create -f service-reader.yaml -n foo
role.rbac.authorization.k8s.io/service-reader created
```
除了使用YAML文件的方式创建Role，还可以直接使用kubectl create role命令来创建，使用这种方式创建bar命名空间中的Role
```
master@k8s-master:~$ sudo kubectl create role service-reader --verb=get --verb=list --resource=services -n bar
role.rbac.authorization.k8s.io/service-reader created
```

创建完Role，下面就要将每个角色绑定到各自命名空间中的ServiceAccount上。通过创建一个RoleBinding资源来实现将角色绑定到主体，运行下面的命令，我们将service-reader角色绑定到default ServiceAccount:
```
master@k8s-master:~$ sudo kubectl create rolebinding test --role=service-reader --serviceaccount=foo:default -n foo
rolebinding.rbac.authorization.k8s.io/test created
```
这个RoleBinding资源也被创建在foo命名空间中。

注意，如果要绑定一个角色到一个用户，则使用--user作为参数来指定用户名，如果要绑定角色到组，则使用--group参数。

至此，已经将service-reader角色绑定到foo空间的默认ServiceAccount上，我们来查看一下这个RoleBinding的信息

```
master@k8s-master:~/$ sudo kubectl get rolebinding test -n foo -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2021-01-14T11:30:38Z"
  managedFields:
  - apiVersion: rbac.authorization.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:roleRef:
        f:apiGroup: {}
        f:kind: {}
        f:name: {}
      f:subjects: {}
    manager: kubectl-create
    operation: Update
    time: "2021-01-14T11:30:38Z"
  name: test
  namespace: foo
  resourceVersion: "297847"
  uid: a6869bc9-9878-4496-a983-77bcfe65b53a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: service-reader
subjects:
- kind: ServiceAccount
  name: default
  namespace: foo
```
能够很清晰地看到哪个角色绑定到哪个主体。

接下来尝试一个foo命名空间中的Pod内的应用是否能够从API获得服务列表
```
master@k8s-master:~$ sudo kubectl exec -it test-7648b6b45f-tsg9g -n foo -- sh
/ # curl localhost:8001/api/v1/namespaces/foo/services
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "298410"
  },
  "items": []
}
```
没有问题，已经能够正常的访问到，只是现在foo命名空间内没有服务存在所以items为空。

现在在来看bar命名空间中的Pod，现在该Pod中的进程还不能从API 服务器中得到服务列表，因为还没有将它的ServiceAccount与角色绑定，按照上述的步骤同样可以完成，这个工作就不在重复。现在想要修改foo命名空间中的RoleBinding，添加bar命名空间下的ServiceAccount，使bar命名空间下的Pod能够得到foo命名空间下服务列表。
```
sudo kubectl edit rolebinding test -n foo
```
在subjects下添加如下内容
```
- kind: ServiceAccount
  name: default
  namespace: bar
```
进入bar命名空间中的Pod，尝试访问foo命名空间下的服务列表
```
master@k8s-master:~/$ sudo kubectl exec -it test-7648b6b45f-lzz7k -n bar -- sh
/ # curl localhost:8001/api/v1/namespaces/foo/services
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "299769"
  },
  "items": []
}
```
没有问题，经过上述操作，目前集群中在foo命名空间下有一个RoleBinding，它引用了foo命名空间中的service-reader角色，并且绑定了foo和bar命名空间中的default ServiceAccount。



## 2.2.2 ClusterRole与ClusterRoleBinding演示






