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

### 2.1.4 YAML文件创建
*my-configmap.yaml*
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfigmap
data:
  sex: man
  name: Jeff
  age: '15'
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
      volumeMounts:
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

Secret是一个主要用来存储密码token等一些敏感信息的资源对象。其中，敏感信息是采用base-64编码保存起来的。查看Secret的定义：
```
apiVersion: v1
kind: Secret # Secret元数据
metadata: 
  name: mysecret
  namespace: kube-system
type: Opaque
data:
  username: YYUYYYLKSD
  password: sjdfksdklflkll=
```
元数据主要包括name、namespace，type是指Secret的类型,常用的四种类型有：(1)Opaque，普通的Secret文件；(2)service-account-token，用于service-account身份认证用的Secret；(3)dockerconfigjson，是拉取私有仓库镜像的用的一种Secret；(4)bootstrap.token，用于节点接入集群校验用的Secret。再接下来是data，存储的Secret的数据，和ConfigMap一样，它也是key-value的形式存储。

## 3.1 Secret的创建

K8s为每一个namespace的默认用户（default ServiceAccount）创建Secret，用户也可以手动创建Secret，推荐kubectl这个命令行工具，它相对ConfigMap会多一个type参数，其中data也是一样，它也是可以指定文件和键值对的，type的话，要是不指定的话，默认是Opaque类型。
```
sudo kubectl create secrt generic [NAME] [DATA] [TYPE]
```
### 3.1.1 指定键值对创建

通过--from-literal选项，每个--from-literal对应一个信息条目。
```
sudo kubectl create secret generic mysecret --from-literal=username=admin --from-literal=password=123456
```
查看创建的mysecret的具体内容
```
master@k8s-master:~$ sudo kubectl get secret mysecret -o yaml
apiVersion: v1
data:
  password: MTIzNDU2
  username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: "2021-01-06T01:29:22Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:password: {}
        f:username: {}
      f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-01-06T01:29:22Z"
  name: mysecret
  namespace: default
  resourceVersion: "241220"
  uid: 04979966-d6df-4081-8f35-a1d8f2a4b79d
type: Opaque
```
可以看到username和password分别被表示为YWRtaW4=和MTIzNDU2。我们查看一下admin和123456的base64编码
```
master@k8s-master:~$ echo -n admin | base64
YWRtaW4=
master@k8s-master:~$ echo -n 123456 | base64
MTIzNDU2
```
可以验证，Secret中确实使用base64编码来存储value

### 3.1.2 指定独立文件创建

通过--from-file选项，每个文件内容对应一个信息条目。

假设在当前目录下我们创建了两个文件：username和password，其中username文件中的内容为admin，password文件中的内容为123456。使用如下命令创建Secret mysecret1
```
sudo kubectl create secret generic mysecret1 --from-file=username --from-file=password
```
查看创建的mysecret1的具体内容
```
master@k8s-master:~$ sudo kubectl get secret mysecret1 -o yaml
apiVersion: v1
data:
  password: MTIzNDU2Cg==
  username: YWRtaW4K
kind: Secret
metadata:
  creationTimestamp: "2021-01-06T01:39:57Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:password: {}
        f:username: {}
      f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-01-06T01:39:57Z"
  name: mysecret1
  namespace: default
  resourceVersion: "242117"
  uid: f9975230-aa09-43ba-88b1-b44564ab24d1
type: Opaque
```
没有问题！！！

### 3.1.3 指定文件创建

我们还可以将Secret中的每一个key:value放到一个文件中（一行一个），使用--from-env-file来创建Secret，这样文件中每一行的内容对应一个信息条目。

假如我们有一个文件user.txt，其中的内容如下：
*user.txt*
```
username=admin
password=123456
```

创建命令如下：
```
sudo kubectl create secret generic mysecret2 --from-env-file=user.txt
```
查看创建的mysecret2的具体内容
```
master@k8s-master:~$ sudo kubectl get secret mysecret2 -o yaml
apiVersion: v1
data:
  password: MTIzNDU2
  username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: "2021-01-06T01:47:22Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:password: {}
        f:username: {}
      f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-01-06T01:47:22Z"
  name: mysecret2
  namespace: default
  resourceVersion: "242748"
  uid: bf3fbf9f-8994-42fa-b561-92c0c8fc35f0
type: Opaque
```
也是OK的！！！

### 3.1.4 YAML文件

*mysecret3.yaml*
```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret3
data:
  username: admin
  password: 123456
```
使用如下命令来创建
```
master@k8s-master:~$ sudo kubectl apply -f mysecret3.yaml 
Error from server (BadRequest): error when creating "mysecret3.yaml": Secret in version "v1" cannot be handled as a Secret: v1.Secret.Data: base64Codec: invalid input, error found in #10 byte of ...|password":123456,"us|..., bigger context ...|{"apiVersion":"v1","data":{"password":123456,"username":"admin"},"kind":"Secret","metada|...
```
呀，报错了，根据报错信息，发现在YAML文件中，value得使用base64编码，因此修改
```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret3
data:
  username: YWRtaW4=
  password: MTIzNDU2
```
再次创建
```
master@k8s-master:~$ sudo kubectl apply -f mysecret3.yaml 
secret/mysecret3 created
master@k8s-master:~$ sudo kubectl get secret mysecret3 -o yaml
apiVersion: v1
data:
  password: MTIzNDU2
  username: YWRtaW4=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"MTIzNDU2","username":"YWRtaW4="},"kind":"Secret","metadata":{"annotations":{},"name":"mysecret3","namespace":"default"}}
  creationTimestamp: "2021-01-06T01:56:45Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:password: {}
        f:username: {}
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
      f:type: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2021-01-06T01:56:45Z"
  name: mysecret3
  namespace: default
  resourceVersion: "243540"
  uid: faf695f4-5d5a-44cd-a8d7-4c8c199cada1
type: Opaque
```
齐活儿~~

## 3.2 查看Secret

首先看一下集群中目前存在的Secret
```
master@k8s-master:~$ sudo kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-vs6vn   kubernetes.io/service-account-token   3      21d
mysecret              Opaque                                2      40m
mysecret1             Opaque                                2      29m
mysecret2             Opaque                                2      22m
mysecret3             Opaque                                2      12m
```
以mysecret为例，通过kubectl describe secret mysecret查看条目的Key
```
master@k8s-master:~$ sudo kubectl describe secret mysecret
Name:         mysecret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  6 bytes
username:  5 bytes
```

接着来，使用kubectl edit secret mysecret继续查看value
```
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  password: MTIzNDU2
  username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: "2021-01-06T01:29:22Z"
  name: mysecret
  namespace: default
  resourceVersion: "241220"
  uid: 04979966-d6df-4081-8f35-a1d8f2a4b79d
type: Opaque
~
~
```
得到data如下
```
password: MTIzNDU2
username: YWRtaW4=
```
然后通过base64将Value反编码
```
master@k8s-master:~$ echo -n YWRtaW4= | base64 --decode
adminmaster@k8s-master:~$ 
master@k8s-master:~$ echo -n MTIzNDU2 | base64 --decode
123456master@k8s-master:~$ 
```
即可获得value值

## 3.3 Secret的使用

Pod可以通过Volume或者环境变量的方式使用Secret。

### 3.3.1 Volume方式

Pod的创建文件如下
*my-secret-pod.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: my-secret-pod
spec:
  containers:
    - name: test-container
      image: luksa/kubia:v1
      args:
        - /bin/sh
        - -c
        - sleep 30000 
      volumeMounts:
        - name: test-secret
          mountPath: /etc/secret
          readOnly: true
  volumes:
    - name: test-secret
      secret:
        secretName: mysecret
```
文件解析： 定义volume test-secret，来源为secret mysecret；将test-secret mount到容器路径/etc/secret，可指定读写权限为readOnly。

创建Pod
```
master@k8s-master:~$ sudo kubectl apply -f my-secret-pod.yaml 
pod/my-secret-pod created
```
进入容器，读取Secret
```
master@k8s-master:~$ sudo kubectl exec -it my-secret-pod -- sh
# ls /etc/secret             
password  username
# cat /etc/secret/username
admin# 
# cat /etc/secret/password
123456# 
```
可以看到，K8s会在指定的路径/etc/secret下为每条敏感数据创建一个文件，文件名就是数据条目的Key，这里是/etc/secret/username和/etc/foo/password，Value则以明文存放在文件中。

我们还可以自定义存放数据的文件名，比如将Pod配置文件改为

```
apiVersion: v1
kind: Pod
metadata:
  name: my-secret-pod
spec:
  containers:
    - name: test-container
      image: luksa/kubia:v1
      args:
        - /bin/sh
        - -c
        - sleep 30000 
      volumeMounts:
        - name: test-secret
          mountPath: /etc/secret
          readOnly: true
  volumes:
    - name: test-secret
      secret:
        secretName: mysecret
        items:
          - key: username
            path: my-group/my-username
          - key: password
            path: my-group/my-password
```
查看Pod
```
master@k8s-master:~$ sudo kubectl exec -it my-secret-pod -- sh
# ls /etc/secret
my-group
# cat /etc/secret/my-group/my-username
admin# cat etc/secret/my-group/my-password
123456# 
```
这时数据将分别存放在/etc/secret/my-group/my-username和/etc/secret/my-group/my-password中。

此外，以Volume方式使用的Secret支持动态更新：Secret更新后，容器中的数据也会更新。

### 3.3.2 环境变量方式

K8s还支持通过环境变量使用Secret。

创建Pod

*my-secret-exec-pod.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: my-secret-pod
spec:
  containers:
  - image: luksa/kubia:v1
    name: test-container
    args:
      - /bin/sh
      - -c
      - sleep 30000 
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
```
创建Pod，并且进入容器，查看环境变量
```
master@k8s-master:~$ sudo kubectl apply -f my-secret-exec-pod.yaml 
pod/my-secret-pod created
master@k8s-master:~$ sudo kubectl get pod
NAME            READY   STATUS    RESTARTS   AGE
my-secret-pod   1/1     Running   0          4s
master@k8s-master:~$ sudo kubectl exec -it my-secret-pod -- sh
# echo $SECRET_USERNAME
admin
# echo $SECRET_PASSWORD
123456
# exit
```
明明白白的~~

需要注意的是，环境变量读取Secret很方便，但无法支撑Secret动态更新。

### 3.4 总结

* Secret的文件大小限制和ConfigMap一样，也是 1MB；
* Secret采用了base64编码，但是它跟明文也没有太大区别。所以说，如果有一些机密信息要用Secret来存储的话，还是要很慎重考虑；


# 4. ServiceAccount

ServiceAccount是用于解决Pod在集群里面的身份认证问题，身份认证信息是存在于Secret里面，这时候Secret的type就是kubernetes.io/service-account-token。



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


