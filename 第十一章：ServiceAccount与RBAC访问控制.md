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

### 2.2.1 Role与RoleBinding演示

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


### 2.2.2 ClusterRole与ClusterRoleBinding演示

Role和RoleBinding都是命名空间的资源，这意味着他们属于和应用在一个单一的命名空间资源上，但是RoleBinding可以应用来自其他命名空间的ServiceAccount。

K8s中还有两个集群级别的RBAC资源：ClusterRole和ClusterRoleBinding。ClusterRole允许访问没有命名空间的资源（Node\PV\Namespace等）和非资源型URL（/healthz等），或者作为单个命名空间内部绑定的公共角色，从而避免必须在每个命名空间中重新定义相同的角色。

#### 2.2.2.1 访问集群级别的资源

下面演示如何允许Pod列出集群中的PV

首先创建一个ClusterRole，名叫pv-reader
```
sudo kubectl create clusterrole pv-reader --verb=get,list --resource=persistentvolumes
```
查看该ClusterRole的详情
```
master@k8s-master:~$ sudo kubectl get clusterrole pv-reader -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: "2021-01-15T01:53:21Z"
  managedFields:
  - apiVersion: rbac.authorization.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:rules: {}
    manager: kubectl-create
    operation: Update
    time: "2021-01-15T01:53:21Z"
  name: pv-reader
  resourceVersion: "301312"
  uid: ebc052e1-3e18-4b28-8d81-cfca64331881
rules:
- apiGroups:
  - ""
  resources:
  - persistentvolumes
  verbs:
  - get
  - list
```

在将ClusterRole绑定到Pod之前，先来验证一下能够列出PV，我们进入foo命名空间下的Pod中，尝试访问PV
```
master@k8s-master:~$ sudo kubectl exec -it test-7648b6b45f-tsg9g -n foo -- sh
/ # curl localhost:8001/api/v1/persistentvolumes
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "persistentvolumes is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"persistentvolumes\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "persistentvolumes"
  },
  "code": 403
}
```
目前看还是不可以的！

下面创建ClusterRoleBinding，将ClusterRole与Pod的ServiceAccount绑定在一起，注意当想要授权Pod访问集群级资源时，不能使用RoleBinding绑定ClusterRole与Pod的ServiceAccount。

创建ClusterRoleBinding
```
sudo kubectl create clusterrolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default
```
然后再次进入foo命名空间下的Pod中，访问PV
```
master@k8s-master:~$ sudo kubectl exec -it test-7648b6b45f-tsg9g -n foo -- sh
/ # curl localhost:8001/api/v1/persistentvolumes
{
  "kind": "PersistentVolumeList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "302597"
  },
  "items": []
}
```
没有问题，现在Pod已经获得了访问PV的权限了

#### 2.2.2.2 访问非资源型的URL

API服务器也会外暴露非资源型的URL，访问这些URL也必须要显示地授予权限，否则API服务器会拒绝客户端的请求。通常这个会通过system:discovery ClusterRole和相同命名的ClusterRoleBinding来自动完成。

查看一下默认的system:discovery ClusterRole
```
master@k8s-master:~$ sudo kubectl get clusterrole system:discovery -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2020-12-15T13:12:19Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  managedFields:
  - apiVersion: rbac.authorization.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:rbac.authorization.kubernetes.io/autoupdate: {}
        f:labels:
          .: {}
          f:kubernetes.io/bootstrapping: {}
      f:rules: {}
    manager: kube-apiserver
    operation: Update
    time: "2020-12-15T13:12:19Z"
  name: system:discovery
  resourceVersion: "83"
  uid: c75aa89c-d5c1-46ee-a26e-44947e145c03
rules:
- nonResourceURLs:
  - /api
  - /api/*
  - /apis
  - /apis/*
  - /healthz
  - /livez
  - /openapi
  - /openapi/*
  - /readyz
  - /version
  - /version/
  verbs:
  - get
```
可以看出，该ClusterRole引用的是URL路径而不是自愿，verbs字段只允许使用HTTP GET方法

和集群级别的资源一样，非资源型的URL ClusterRole必须与ClusterRoleBinding结合使用，与RoleBinding绑定没有任何效果。

与system:discovery ClusterRole对应的还有一个system:discovery ClusterRoleBinding，查看一下
```
master@k8s-master:~$ sudo kubectl get clusterrolebinding system:discovery -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2020-12-15T13:12:19Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  managedFields:
  - apiVersion: rbac.authorization.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:rbac.authorization.kubernetes.io/autoupdate: {}
        f:labels:
          .: {}
          f:kubernetes.io/bootstrapping: {}
      f:roleRef:
        f:apiGroup: {}
        f:kind: {}
        f:name: {}
      f:subjects: {}
    manager: kube-apiserver
    operation: Update
    time: "2020-12-15T13:12:19Z"
  name: system:discovery
  resourceVersion: "144"
  uid: 48ee6a60-ecde-419a-a061-13f5cf86089a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:discovery
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
```
ClusterRoleBinding 绑定了system:discovery ClusterRole和system:authenticated组，也就是经过认证的用户都可以Get非资源型URL。我们来确认一下

首先未通过认证的用户不能Get非资源型URL
```
master@k8s-master:~$ curl https://10.10.10.147:6443/api -k
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/api\"",
  "reason": "Forbidden",
  "details": {
    
  },
  "code": 403
}
```

然后通过认证的用户能Get非资源型URL

开启kubectl proxy
```
master@k8s-master:~$ sudo kubectl proxy
Starting to serve on 127.0.0.1:8001
```
访问
```
master@k8s-master:~$ curl localhost:8001/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.10.10.147:6443"
    }
  ]
}
```

#### 2.2.2.3 使用ClusterRole来授权访问指定命名空间中的资源

ClusterRole不是一定和集群级别的ClusterRoleBinding绑定使用，它们也可以和常规的有命名空间的RoleBinding进行绑定。

集群中有一个名为view的ClusterRole，其信息如下
```
master@k8s-master:~$ sudo kubectl get clusterrole view -o yaml
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-view: "true"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2020-12-15T13:12:19Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
  managedFields:
  - apiVersion: rbac.authorization.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:aggregationRule:
        .: {}
        f:clusterRoleSelectors: {}
      f:metadata:
        f:annotations:
          .: {}
          f:rbac.authorization.kubernetes.io/autoupdate: {}
        f:labels:
          .: {}
          f:kubernetes.io/bootstrapping: {}
          f:rbac.authorization.k8s.io/aggregate-to-edit: {}
    manager: kube-apiserver
    operation: Update
    time: "2020-12-15T13:12:19Z"
  - apiVersion: rbac.authorization.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:rules: {}
    manager: kube-controller-manager
    operation: Update
    time: "2020-12-15T13:12:36Z"
  name: view
  resourceVersion: "416"
  uid: b5576df0-538b-4c12-97f8-c65b5a73589a
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - persistentvolumeclaims/status
  - pods
  - replicationcontrollers
  - replicationcontrollers/scale
  - serviceaccounts
  - services
  - services/status
  verbs:
  - get
  - list
  - watch
...
```
这个ClusterRole有很多规则，我们显示了第一条，可以看到在该规则中包括资源ConfigMap、Endpoint、PVC等，这些资源都是由命名空间的，允许的操作为get、list和watch。

下面好玩的地方来了，假如这个ClusterRole与ClusterRoleBinding绑定，那么绑定中的主体可以在所有的命名空间中查看指定的资源。假如这个ClusterRole与RoleBinding绑定，那么绑定中的主体只能查看在RoleBinding命名空间中查看指定的资源。

分来来看看，首先创建一个ClusterRoleBinding，将view ClusterRole与foo命名空间下的defult ServiceAccount绑定。
```
master@k8s-master:~$ sudo kubectl create clusterrolebinding view-test --clusterrole=view --serviceaccount=foo:default
clusterrolebinding.rbac.authorization.k8s.io/view-test created
```
进入foo命名空间中的testPod中，尝试查看foo命名空间中的Pod列表和bar命名空间中的Pod列表
```
master@k8s-master:~$ sudo kubectl exec -it test-7648b6b45f-tsg9g -n foo -- sh
/ # curl localhost:8001/api/v1/namespaces/foo/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
...

/ # curl localhost:8001/api/v1/namespaces/bar/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
...

/ # curl localhost:8001/api/v1/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
...

```
没问题都能够访问，证明了ClusterRole与ClusterRoleBinding绑定，绑定中的主体可以在所有的命名空间中查看指定的资源。

然后我们删除ClusterRoleBinding，并创建一个RoleBinding
```
master@k8s-master:~$ sudo kubectl delete clusterrolebinding view-test
clusterrolebinding.rbac.authorization.k8s.io "view-test" deleted
master@k8s-master:~$ sudo kubectl create rolebinding view-test --clusterrole=view --serviceaccount=foo:default -n foo
rolebinding.rbac.authorization.k8s.io/view-test created
```

现在foo命名空间中有一个RoleBinding，它将同一命名空间中的default ServiceAccount绑定到view ClusterRole。

现在来看一下foo命名空间中的Pod可以访问哪些资源呢

```
master@k8s-master:~$ sudo kubectl exec -it test-7648b6b45f-tsg9g -n foo -- sh
/ # curl localhost:8001/api/v1/namespaces/foo/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "308171"
  },
  "items": [
    ...
  ]
/ # curl localhost:8001/api/v1/namespaces/bar/pods
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"pods\" in API group \"\" in the namespace \"bar\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
/ # curl localhost:8001/api/v1/pods
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"pods\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```
可以看到，这个Pod只能列出foo命名空间中的Pod，这也证明了指向一个ClusterRole的RoleBinding只授权获取在RoleBinding命名空间中的资源。

# 3. 总结

Pod使用ServiceAccount来向API服务器表明自己的身份，授权插件，如RBAC插件根据Pod的身份来检测Pod是否具有执行当前操作的权限。




