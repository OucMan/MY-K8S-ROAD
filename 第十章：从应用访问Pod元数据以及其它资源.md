# 1. 概述

容器中应用往往需要获取所运行环境的一些信息，包括应用自身以及集群中其他组件的元数据，比如Pod的名称、Pod的IP、Pod所在的命名空间、Pod运行节点的名称、Pod运行所归属的Sevice Account的名称、每个容器请求的CPU和内存使用率、每个容器可以使用的CPU和内存的限制、Pod的标签和注解等。本章将介绍通过环境变量、Dowanward API以及K8s API服务器三种方式使得应用获得元数据。

# 2. 通过环境变量暴露元数据

首先演示利用环境变量的方式将Pod和容器的元数据传递到容器中。

*env-metadata.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: env-metadata
spec:
  containers:
  - name: main
    image: busybox:latest
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName
    - name: CONTAINER_CPU_REQUEST_MILLICORES
      valueFrom:
        resourceFieldRef:
          resource: requests.cpu
          divisor: 1m
    - name: CONTAINER_MEMORY_LIMIT_KIBIBYTES
      valueFrom:
        resourceFieldRef:
          resource: limits.memory
          divisor: 1Ki
```
 文件中创建了一个名为env-metadata的Pod，其中有一个容器，在容器中定义了env字段，在env字段下定义了诸多的环境变量，这些环境变量的值来源于Pod、节点的元数据。
 
 创建Pod，然后查看该容器中的环境变量
 
```
master@k8s-master:~$ sudo kubectl apply -f env-metadata.yaml 
pod/env-metadata created
master@k8s-master:~$ sudo kubectl exec env-metadata -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=env-metadata
POD_NAMESPACE=default
POD_IP=10.244.1.38
NODE_NAME=k8s-node1
SERVICE_ACCOUNT=default
CONTAINER_CPU_REQUEST_MILLICORES=15
CONTAINER_MEMORY_LIMIT_KIBIBYTES=4096
POD_NAME=env-metadata
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
HOME=/root
```
可以看到通过环境变量的方式可以将Pod或者容器的一些元数据暴露给容器。

**注意**：通过环境变量的方式不能暴露标签和注解，因为这两个元数据在Pod启动后可以被修改，但是修改后的数据不会导致环境变量更新。


# 3.通过Dowanward API暴露元数据

除了环境变量的方式，还可以利用文件的方式来暴露元数据，即通过挂载downwardAPI卷的方式来暴露元数据，其中包括标签和注解。

示例

*downward-api-volume.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: downward
  labels:
    foo: bar
  annotations:
    key1: value1
    key2: |
      multi
      line
      value
spec:
  containers:
  - name: main
    image: busybox:latest
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    volumeMounts:
    - name: downward
      mountPath: /etc/downward
  volumes:
  - name: downward
    downwardAPI:
      items:
      - path: "podName"
        fieldRef:
          fieldPath: metadata.name
      - path: "podNamespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "containerCpuRequestMilliCores"
        resourceFieldRef:
          containerName: main
          resource: requests.cpu
          divisor: 1m
      - path: "containerMemoryLimitBytes"
        resourceFieldRef:
          containerName: main
          resource: limits.memory
          divisor: 1
```

文件解析：定义了一个叫做downward的卷，并且通过/etc/downward目录挂载到容器中，卷中包含的文件通过卷定义中的downwardAPI.items属性来定义，对于想要在文件中保存的Pod级或者容器资源字段，都分别在downwardAPI.items中说明了元数据被保存和应用的path，其结构如下图所示，


创建Pod，查看downwardAPI卷中的文件
```
master@k8s-master:~$ sudo kubectl apply -f downward-api-volume.yaml 
pod/downward created
master@k8s-master:~$ sudo kubectl exec downward -- ls /etc/downward
annotations
containerCpuRequestMilliCores
containerMemoryLimitBytes
labels
podName
podNamespace
```
可以看到每个文件对应卷定义的意向，打开标签和注解的文件看看
```
master@k8s-master:~$ sudo kubectl exec downward -- cat /etc/downward/labels
foo="bar"
master@k8s-master:~$ sudo kubectl exec downward -- cat /etc/downward/annotations
key1="value1"
key2="multi\nline\nvalue\n"
kubectl.kubernetes.io/last-applied-configuration="{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{\"key1\":\"value1\",\"key2\":\"multi\\nline\\nvalue\\n\"},\"labels\":{\"foo\":\"bar\"},\"name\":\"downward\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"sleep\",\"9999999\"],\"image\":\"busybox:latest\",\"name\":\"main\",\"resources\":{\"limits\":{\"cpu\":\"100m\",\"memory\":\"4Mi\"},\"requests\":{\"cpu\":\"15m\",\"memory\":\"100Ki\"}},\"volumeMounts\":[{\"mountPath\":\"/etc/downward\",\"name\":\"downward\"}]}],\"volumes\":[{\"downwardAPI\":{\"items\":[{\"fieldRef\":{\"fieldPath\":\"metadata.name\"},\"path\":\"podName\"},{\"fieldRef\":{\"fieldPath\":\"metadata.namespace\"},\"path\":\"podNamespace\"},{\"fieldRef\":{\"fieldPath\":\"metadata.labels\"},\"path\":\"labels\"},{\"fieldRef\":{\"fieldPath\":\"metadata.annotations\"},\"path\":\"annotations\"},{\"path\":\"containerCpuRequestMilliCores\",\"resourceFieldRef\":{\"containerName\":\"main\",\"divisor\":\"1m\",\"resource\":\"requests.cpu\"}},{\"path\":\"containerMemoryLimitBytes\",\"resourceFieldRef\":{\"containerName\":\"main\",\"divisor\":1,\"resource\":\"limits.memory\"}}]},\"name\":\"downward\"}]}}\n"
kubernetes.io/config.seen="2021-01-10T23:13:40.094361262-08:00"
kubernetes.io/config.source="api"
```

当修改标签或者注解时，卷中的文件也会相应的修改，以修改标签为例，将foo=bar，改为foo=test
```
master@k8s-master:~$ sudo kubectl label pod downward foo=test --overwrite
pod/downward labeled
master@k8s-master:~$ sudo kubectl exec downward -- cat /etc/downward/labelsfoo="test"master@k8s-master:~/k8s-learning/resources/pod$ 
```
一起更新，没有问题！



# 4.通过K8s API服务器获得元数据
