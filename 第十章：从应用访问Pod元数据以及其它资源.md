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
 
 ```


# 3.通过Dowanward API暴露元数据


# 4.通过K8s API服务器获得元数据
