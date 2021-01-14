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

此外，如果想要再卷的定义中引入容器级的元数据，则需要指定容器的名字，如下
```
spec:
  volumes:
  - name: downward
    downwardAPI:
      items:
      - path: "containerCpuRequestMilliCores"
        resourceFieldRef:
          containerName: main
          resource: request.cpu
          divisor: 1m
```

# 4.通过K8s API服务器获得元数据

downwardAPI提供了一种简单的方式，将Pod和容器的元数据传递给它们内部运行的进程，但这种方式仅可以暴露一个Pod自身的元数据，而且只可以暴露部分元数据。某些情况下，应用需要知道其它Pod的信息，甚至是集群中其它资源的信息，这时候则需要直接与K8s API服务器进行交互。

## 4.1 连接K8s REST API

首先利用kubectl cluster-info命名来获取k8s API服务器的URL
```
master@k8s-master:~$ sudo kubectl cluster-info
Kubernetes control plane is running at https://10.10.10.147:6443
KubeDNS is running at https://10.10.10.147:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
可以看到k8s API服务器运行在10.10.10.147主机上，监听6443端口，尝试去连接服务器
```
master@k8s-master:~$ curl https://10.10.10.147:6443
curl: (60) server certificate verification failed. CAfile: /etc/ssl/certs/ca-certificates.crt CRLfile: none
More details here: http://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
master@k8s-master:~$ curl https://10.10.10.147:6443 -k
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {
    
  },
  "code": 403
}
```
观察到直接curl API服务器会失败，因为API服务器使用的是HTTPS协议，需要授权，然后尝试-k选项跳过验证也不能连接成功。

莫慌，K8s为我们提供个了代理服务，用来接收来自本机的HTTP连接并将其转发到API服务器，同时处理身份认证，开启代理
```
master@k8s-master:~$ sudo kubectl proxy
Starting to serve on 127.0.0.1:8001
```
启动后，代理服务器将在本地端口8001接收连接请求，尝试发送请求给代理，查看响应
```
master@k8s-master:~$ curl localhost:8001
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/admissionregistration.k8s.io/v1beta1",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1",
    "/apis/apiextensions.k8s.io/v1beta1",
    "/apis/apiregistration.k8s.io",
    "/apis/apiregistration.k8s.io/v1",
    "/apis/apiregistration.k8s.io/v1beta1",
    "/apis/apps",
    "/apis/apps/v1",
    "/apis/authentication.k8s.io",
    "/apis/authentication.k8s.io/v1",
    "/apis/authentication.k8s.io/v1beta1",
    "/apis/authorization.k8s.io",
    "/apis/authorization.k8s.io/v1",
    "/apis/authorization.k8s.io/v1beta1",
    "/apis/autoscaling",
    "/apis/autoscaling/v1",
    "/apis/autoscaling/v2beta1",
    "/apis/autoscaling/v2beta2",
    "/apis/batch",
    "/apis/batch/v1",
    "/apis/batch/v1beta1",
    "/apis/certificates.k8s.io",
    "/apis/certificates.k8s.io/v1",
    "/apis/certificates.k8s.io/v1beta1",
    "/apis/coordination.k8s.io",
    "/apis/coordination.k8s.io/v1",
    "/apis/coordination.k8s.io/v1beta1",
    "/apis/discovery.k8s.io",
    "/apis/discovery.k8s.io/v1beta1",
    "/apis/events.k8s.io",
    "/apis/events.k8s.io/v1",
    "/apis/events.k8s.io/v1beta1",
    "/apis/extensions",
    "/apis/extensions/v1beta1",
    "/apis/flowcontrol.apiserver.k8s.io",
    "/apis/flowcontrol.apiserver.k8s.io/v1beta1",
    "/apis/networking.k8s.io",
    "/apis/networking.k8s.io/v1",
    "/apis/networking.k8s.io/v1beta1",
    "/apis/node.k8s.io",
    "/apis/node.k8s.io/v1",
    "/apis/node.k8s.io/v1beta1",
    "/apis/policy",
    "/apis/policy/v1beta1",
    "/apis/rbac.authorization.k8s.io",
    "/apis/rbac.authorization.k8s.io/v1",
    "/apis/rbac.authorization.k8s.io/v1beta1",
    "/apis/scheduling.k8s.io",
    "/apis/scheduling.k8s.io/v1",
    "/apis/scheduling.k8s.io/v1beta1",
    "/apis/storage.k8s.io",
    "/apis/storage.k8s.io/v1",
    "/apis/storage.k8s.io/v1beta1",
    "/healthz",
    "/healthz/autoregister-completion",
    "/healthz/etcd",
    "/healthz/log",
    "/healthz/ping",
    "/healthz/poststarthook/aggregator-reload-proxy-client-cert",
    "/healthz/poststarthook/apiservice-openapi-controller",
    "/healthz/poststarthook/apiservice-registration-controller",
    "/healthz/poststarthook/apiservice-status-available-controller",
    "/healthz/poststarthook/bootstrap-controller",
    "/healthz/poststarthook/crd-informer-synced",
    "/healthz/poststarthook/generic-apiserver-start-informers",
    "/healthz/poststarthook/kube-apiserver-autoregistration",
    "/healthz/poststarthook/priority-and-fairness-config-consumer",
    "/healthz/poststarthook/priority-and-fairness-config-producer",
    "/healthz/poststarthook/priority-and-fairness-filter",
    "/healthz/poststarthook/rbac/bootstrap-roles",
    "/healthz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/healthz/poststarthook/start-apiextensions-controllers",
    "/healthz/poststarthook/start-apiextensions-informers",
    "/healthz/poststarthook/start-cluster-authentication-info-controller",
    "/healthz/poststarthook/start-kube-aggregator-informers",
    "/healthz/poststarthook/start-kube-apiserver-admission-initializer",
    "/livez",
    "/livez/autoregister-completion",
    "/livez/etcd",
    "/livez/log",
    "/livez/ping",
    "/livez/poststarthook/aggregator-reload-proxy-client-cert",
    "/livez/poststarthook/apiservice-openapi-controller",
    "/livez/poststarthook/apiservice-registration-controller",
    "/livez/poststarthook/apiservice-status-available-controller",
    "/livez/poststarthook/bootstrap-controller",
    "/livez/poststarthook/crd-informer-synced",
    "/livez/poststarthook/generic-apiserver-start-informers",
    "/livez/poststarthook/kube-apiserver-autoregistration",
    "/livez/poststarthook/priority-and-fairness-config-consumer",
    "/livez/poststarthook/priority-and-fairness-config-producer",
    "/livez/poststarthook/priority-and-fairness-filter",
    "/livez/poststarthook/rbac/bootstrap-roles",
    "/livez/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/livez/poststarthook/start-apiextensions-controllers",
    "/livez/poststarthook/start-apiextensions-informers",
    "/livez/poststarthook/start-cluster-authentication-info-controller",
    "/livez/poststarthook/start-kube-aggregator-informers",
    "/livez/poststarthook/start-kube-apiserver-admission-initializer",
    "/logs",
    "/metrics",
    "/openapi/v2",
    "/openid/v1/jwks",
    "/readyz",
    "/readyz/autoregister-completion",
    "/readyz/etcd",
    "/readyz/informer-sync",
    "/readyz/log",
    "/readyz/ping",
    "/readyz/poststarthook/aggregator-reload-proxy-client-cert",
    "/readyz/poststarthook/apiservice-openapi-controller",
    "/readyz/poststarthook/apiservice-registration-controller",
    "/readyz/poststarthook/apiservice-status-available-controller",
    "/readyz/poststarthook/bootstrap-controller",
    "/readyz/poststarthook/crd-informer-synced",
    "/readyz/poststarthook/generic-apiserver-start-informers",
    "/readyz/poststarthook/kube-apiserver-autoregistration",
    "/readyz/poststarthook/priority-and-fairness-config-consumer",
    "/readyz/poststarthook/priority-and-fairness-config-producer",
    "/readyz/poststarthook/priority-and-fairness-filter",
    "/readyz/poststarthook/rbac/bootstrap-roles",
    "/readyz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/readyz/poststarthook/start-apiextensions-controllers",
    "/readyz/poststarthook/start-apiextensions-informers",
    "/readyz/poststarthook/start-cluster-authentication-info-controller",
    "/readyz/poststarthook/start-kube-aggregator-informers",
    "/readyz/poststarthook/start-kube-apiserver-admission-initializer",
    "/readyz/shutdown",
    "/version"
  ]
}
```
哇哦，帅呀，可以看到代理将我们的请求发到API服务器，然后API服务器返回了所有的路径，这些路径对应了我们创建Pod，Service这些资源时定义的API组和版本信息。

下面我们利用kubectl proxy来研究Pod资源API，我们首先创建一个Pod，yaml文件如下
*kubia-manual.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual
spec:
  containers:
  - image: luksa/kubia:v1
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```
创建Pod
```
master@k8s-master:~$ sudo kubectl apply -f kubia-manual.yaml
pod/kubia-manual created
master@k8s-master:~$ sudo kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
kubia-manual   1/1     Running   0          11s
```
Pod的apiVersion是v1，即我们需要从/api/v1路径下的内容开始，部分响应如下
```
{
      "name": "pods",
      "singularName": "",
      "namespaced": true,
      "kind": "Pod",
      "verbs": [
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "shortNames": [
        "po"
      ],
      "categories": [
        "all"
      ],
      "storageVersionHash": "xPOwRZ+Yhw8="
    },
```
返回的列表描述了在API服务器中暴露的REST资源。"name":"pods"表明API中包含了/api/v1/pods的endpoint，"verbs"数组表示Pods支持的动作，可以创建、删除、查询、更新Pod资源,"shortNames"表示Pod的简称为po，下面通过在/api/v1/pods路径上运行一个Get请求，来获得集群中所有Pod的清单，部分如下
```
master@k8s-master:~$ curl localhost:8001/api/v1/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "254279"
  },
  "items": [
  {
      "metadata": {
        "name": "kubia-manual",
        "namespace": "default",
        "uid": "1d591f87-c631-49c3-9398-aafcfd709636",
        "resourceVersion": "253384",
        "creationTimestamp": "2021-01-11T07:47:02Z",
        "annotations": {
          "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"name\":\"kubia-manual\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"image\":\"luksa/kubia:v1\",\"name\":\"kubia\",\"ports\":[{\"containerPort\":8080,\"protocol\":\"TCP\"}]}]}}\n"
        },
        "managedFields": [
          {
            "manager": "kubectl-client-side-apply",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2021-01-11T07:47:02Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"kubia\"}":{".":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:ports":{".":{},"k:{\"containerPort\":8080,\"protocol\":\"TCP\"}":{".":{},"f:containerPort":{},"f:protocol":{}}},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}
          },
          {
            "manager": "kubelet",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2021-01-11T07:47:04Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"10.244.2.35\"}":{".":{},"f:ip":{}}},"f:startTime":{}}}
          }
        ]
      },
      "spec": {
        "volumes": [
          {
            "name": "default-token-vs6vn",
            "secret": {
              "secretName": "default-token-vs6vn",
              "defaultMode": 420
            }
          }
        ],
        "containers": [
          {
            "name": "kubia",
            "image": "luksa/kubia:v1",
            "ports": [
              {
                "containerPort": 8080,
                "protocol": "TCP"
              }
            ],
            "resources": {
              
            },
            "volumeMounts": [
              {
                "name": "default-token-vs6vn",
                "readOnly": true,
                "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
              }
            ],
            "terminationMessagePath": "/dev/termination-log",
            "terminationMessagePolicy": "File",
            "imagePullPolicy": "IfNotPresent"
          }
        ],
        "restartPolicy": "Always",
        "terminationGracePeriodSeconds": 30,
        "dnsPolicy": "ClusterFirst",
        "serviceAccountName": "default",
        "serviceAccount": "default",
        "nodeName": "k8s-node2",
        "securityContext": {
          
        },
        "schedulerName": "default-scheduler",
        "tolerations": [
          {
            "key": "node.kubernetes.io/not-ready",
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          },
          {
            "key": "node.kubernetes.io/unreachable",
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          }
        ],
        "priority": 0,
        "enableServiceLinks": true,
        "preemptionPolicy": "PreemptLowerPriority"
      },
      "status": {
        "phase": "Running",
        "conditions": [
          {
            "type": "Initialized",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-01-11T07:47:03Z"
          },
          {
            "type": "Ready",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-01-11T07:47:04Z"
          },
          {
            "type": "ContainersReady",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-01-11T07:47:04Z"
          },
          {
            "type": "PodScheduled",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-01-11T07:47:03Z"
          }
        ],
        "hostIP": "10.10.10.149",
        "podIP": "10.244.2.35",
        "podIPs": [
          {
            "ip": "10.244.2.35"
          }
        ],
        "startTime": "2021-01-11T07:47:03Z",
        "containerStatuses": [
          {
            "name": "kubia",
            "state": {
              "running": {
                "startedAt": "2021-01-11T07:47:04Z"
              }
            },
            "lastState": {
              
            },
            "ready": true,
            "restartCount": 0,
            "image": "luksa/kubia:v1",
            "imageID": "docker://sha256:47f34b621acf54fb3cfeea1c17ade6a293b6bc2349a5c4c48277a4371a3185af",
            "containerID": "docker://0b9739ef89fca670c75f4c7c18ca4d3e6bc82a3a8e188065a545a6c4783b87b7",
            "started": true
          }
        ],
        "qosClass": "BestEffort"
      }
    },
  ...
  ]
```
上述信息有很多，因为集群中存在诸多的Pod，如果想要仅列出我们创建的那个kubia-manual Pod的信息，可以这样做：
```
master@k8s-master:~$ curl localhost:8001/api/v1/namespaces/default/pods/kubia-manual
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "kubia-manual",
    "namespace": "default",
    "uid": "1d591f87-c631-49c3-9398-aafcfd709636",
    "resourceVersion": "253384",
    "creationTimestamp": "2021-01-11T07:47:02Z",
    "annotations": {
      "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"name\":\"kubia-manual\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"image\":\"luksa/kubia:v1\",\"name\":\"kubia\",\"ports\":[{\"containerPort\":8080,\"protocol\":\"TCP\"}]}]}}\n"
    },
    "managedFields": [
      {
        "manager": "kubectl-client-side-apply",
        "operation": "Update",
        "apiVersion": "v1",
        "time": "2021-01-11T07:47:02Z",
        "fieldsType": "FieldsV1",
        "fieldsV1": {"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"kubia\"}":{".":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:ports":{".":{},"k:{\"containerPort\":8080,\"protocol\":\"TCP\"}":{".":{},"f:containerPort":{},"f:protocol":{}}},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}
      },
      {
        "manager": "kubelet",
        "operation": "Update",
        "apiVersion": "v1",
        "time": "2021-01-11T07:47:04Z",
        "fieldsType": "FieldsV1",
        "fieldsV1": {"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"10.244.2.35\"}":{".":{},"f:ip":{}}},"f:startTime":{}}}
      }
    ]
  },
  "spec": {
    "volumes": [
      {
        "name": "default-token-vs6vn",
        "secret": {
          "secretName": "default-token-vs6vn",
          "defaultMode": 420
        }
      }
    ],
    "containers": [
      {
        "name": "kubia",
        "image": "luksa/kubia:v1",
        "ports": [
          {
            "containerPort": 8080,
            "protocol": "TCP"
          }
        ],
        "resources": {
          
        },
        "volumeMounts": [
          {
            "name": "default-token-vs6vn",
            "readOnly": true,
            "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
          }
        ],
        "terminationMessagePath": "/dev/termination-log",
        "terminationMessagePolicy": "File",
        "imagePullPolicy": "IfNotPresent"
      }
    ],
    "restartPolicy": "Always",
    "terminationGracePeriodSeconds": 30,
    "dnsPolicy": "ClusterFirst",
    "serviceAccountName": "default",
    "serviceAccount": "default",
    "nodeName": "k8s-node2",
    "securityContext": {
      
    },
    "schedulerName": "default-scheduler",
    "tolerations": [
      {
        "key": "node.kubernetes.io/not-ready",
        "operator": "Exists",
        "effect": "NoExecute",
        "tolerationSeconds": 300
      },
      {
        "key": "node.kubernetes.io/unreachable",
        "operator": "Exists",
        "effect": "NoExecute",
        "tolerationSeconds": 300
      }
    ],
    "priority": 0,
    "enableServiceLinks": true,
    "preemptionPolicy": "PreemptLowerPriority"
  },
  "status": {
    "phase": "Running",
    "conditions": [
      {
        "type": "Initialized",
        "status": "True",
        "lastProbeTime": null,
        "lastTransitionTime": "2021-01-11T07:47:03Z"
      },
      {
        "type": "Ready",
        "status": "True",
        "lastProbeTime": null,
        "lastTransitionTime": "2021-01-11T07:47:04Z"
      },
      {
        "type": "ContainersReady",
        "status": "True",
        "lastProbeTime": null,
        "lastTransitionTime": "2021-01-11T07:47:04Z"
      },
      {
        "type": "PodScheduled",
        "status": "True",
        "lastProbeTime": null,
        "lastTransitionTime": "2021-01-11T07:47:03Z"
      }
    ],
    "hostIP": "10.10.10.149",
    "podIP": "10.244.2.35",
    "podIPs": [
      {
        "ip": "10.244.2.35"
      }
    ],
    "startTime": "2021-01-11T07:47:03Z",
    "containerStatuses": [
      {
        "name": "kubia",
        "state": {
          "running": {
            "startedAt": "2021-01-11T07:47:04Z"
          }
        },
        "lastState": {
          
        },
        "ready": true,
        "restartCount": 0,
        "image": "luksa/kubia:v1",
        "imageID": "docker://sha256:47f34b621acf54fb3cfeea1c17ade6a293b6bc2349a5c4c48277a4371a3185af",
        "containerID": "docker://0b9739ef89fca670c75f4c7c18ca4d3e6bc82a3a8e188065a545a6c4783b87b7",
        "started": true
      }
    ],
    "qosClass": "BestEffort"
  }
}
```
这样就获得了指定Pod的信息

## 4.2 Pod内部与K8s API交互（常规方式）

上面的实验我们是从本机通过kubectl proxy与API服务器进行交互，下面开始描述Pod内部与API服务器交互的流程。

### 4.2.1 发现API服务器地址

首先创建Pod
```
apiVersion: v1
kind: Pod
metadata:
  name: curl
spec:
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "9999999"]
```

集群中有一个kubernetes服务在默认的命名空间被暴露，该服务被配置为指向API服务器
```
master@k8s-master:~$ sudo kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   26d
```
每个服务都被配置了对应的环境变量，因此可以进入容器中，查看环境变量
```
master@k8s-master:~$ sudo kubectl exec -it curl -- bash
root@kubia-manual:/# env | grep KUBERNETES_SERVICE
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT_HTTPS=443
root@kubia-manual:/# 
```
另一种方式就是DNS，直接curl https://kubernetes来指向API 服务器

#### 4.2.2 验证API服务器的身份

连接时通过HTTPS协议，因此需要身份验证。容器在与服务器交互之前，验证服务器身份是必须的，同时API服务器也得认证容器的身份，从而进行授权。

在前面讲述Secret时，集群中会自动创建一个名为default-token-xxxx的Secret，并挂载到每一个容器的/var/run/secrets/kubernetes.io/serviceaccount目录中，查看一下目录下的文件
```
root@kubia-manual:/var/run/secrets/kubernetes.io/serviceaccount# ls
ca.crt	namespace  token
```
有三个文件，包括CA证书，它用来验证API服务器的身份，token用来证明容器自己的身份，还有一个命名空间文件。

下面利用--cacert选项来执行CA证书，然后在Http头部添加Token信息，为了简洁，设置两个环境变量
```
export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

然后尝试去访问API服务器
```
curl -H "Authorization: Bearer $TOKEN" https://10.96.0.1
```
不幸地是，还是访问失败，其原因便是我们的K8s集群使用的基于角色的访问控制（RBAC），这个我们在后续章节会重点讲述，因此默认的服务账户没有被授权访问API服务器，最简单的，我们通过为所有的服务账户赋予集群管理员权限来绕过RBAC机制，执行下面的命令
```
master@k8s-master:~$ sudo kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin --group=system:serviceaccounts
[sudo] password for master: 
clusterrolebinding.rbac.authorization.k8s.io/permissive-binding created
```
这次再在Pod中去访问API服务器
```
root@curl:/# curl -H "Authorization: Bearer $TOKEN" https://10.96.0.1
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/admissionregistration.k8s.io/v1beta1",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1",
    "/apis/apiextensions.k8s.io/v1beta1",
    "/apis/apiregistration.k8s.io",
    "/apis/apiregistration.k8s.io/v1",
    "/apis/apiregistration.k8s.io/v1beta1",
    "/apis/apps",
    "/apis/apps/v1",
    "/apis/authentication.k8s.io",
    "/apis/authentication.k8s.io/v1",
...
```
搞定，然后就可以在Pod中访问集群中的所有资源了。我们注意到容器的/var/run/secrets/kubernetes.io/serviceaccount目录中还有一个namespace文件，这个文件包含了当前Pod所在的命名空间，因此可以通过读取这个文件来获得命名空间信息，我们查看一下该命名空间下所有的Pod，如下
```
root@curl:/# cat /var/run/secrets/kubernetes.io/serviceaccount/namespace 
default
root@curl:/# NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace) 
root@curl:/# curl -H "Authorization: Bearer $TOKEN" https://10.96.0.1/api/v1/namespaces/$NS/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "262776"
  },
  "items": [
    {
      "metadata": {
        "name": "curl",
        "namespace": "default",
        "uid": "ca4b2fb1-736a-4fea-ae87-0ea954c9fdc5",
        "resourceVersion": "261760",
        "creationTimestamp": "2021-01-14T02:09:58Z",
        "annotations": {
          "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"name\":\"curl\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"sleep\",\"9999999\"],\"image\":\"tutum/curl\",\"name\":\"main\"}]}}\n"
        },
        "managedFields": [
          {
            "manager": "kubectl-client-side-apply",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2021-01-14T02:09:58Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"main\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}
          },
          {
            "manager": "kubelet",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2021-01-14T02:10:02Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"10.244.1.42\"}":{".":{},"f:ip":{}}},"f:startTime":{}}}
          }
        ]
      },
      "spec": {
        "volumes": [
          {
            "name": "default-token-vs6vn",
            "secret": {
              "secretName": "default-token-vs6vn",
              "defaultMode": 420
            }
          }
        ],
        "containers": [
          {
            "name": "main",
            "image": "tutum/curl",
            "command": [
              "sleep",
              "9999999"
            ],
            "resources": {
              
            },
            "volumeMounts": [
              {
                "name": "default-token-vs6vn",
                "readOnly": true,
                "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
              }
            ],
            "terminationMessagePath": "/dev/termination-log",
            "terminationMessagePolicy": "File",
            "imagePullPolicy": "Always"
          }
        ],
        "restartPolicy": "Always",
        "terminationGracePeriodSeconds": 30,
        "dnsPolicy": "ClusterFirst",
        "serviceAccountName": "default",
        "serviceAccount": "default",
        "nodeName": "k8s-node1",
        "securityContext": {
          
        },
        "schedulerName": "default-scheduler",
        "tolerations": [
          {
            "key": "node.kubernetes.io/not-ready",
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          },
          {
            "key": "node.kubernetes.io/unreachable",
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          }
        ],
        "priority": 0,
        "enableServiceLinks": true,
        "preemptionPolicy": "PreemptLowerPriority"
      },
      "status": {
        "phase": "Running",
        "conditions": [
          {
            "type": "Initialized",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-01-14T02:09:58Z"
          },
          {
            "type": "Ready",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-01-14T02:10:02Z"
          },
          {
            "type": "ContainersReady",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-01-14T02:10:02Z"
          },
          {
            "type": "PodScheduled",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-01-14T02:09:58Z"
          }
        ],
        "hostIP": "10.10.10.148",
        "podIP": "10.244.1.42",
        "podIPs": [
          {
            "ip": "10.244.1.42"
          }
        ],
        "startTime": "2021-01-14T02:09:58Z",
        "containerStatuses": [
          {
            "name": "main",
            "state": {
              "running": {
                "startedAt": "2021-01-14T02:10:02Z"
              }
            },
            "lastState": {
              
            },
            "ready": true,
            "restartCount": 0,
            "image": "tutum/curl:latest",
            "imageID": "docker-pullable://tutum/curl@sha256:b6f16e88387acd4e6326176b212b3dae63f5b2134e69560d0b0673cfb0fb976f",
            "containerID": "docker://39be59377212bcc75ff51bc585d79e225cb639b286d03c8f737440594f1d70b6",
            "started": true
          }
        ],
        "qosClass": "BestEffort"
      }
    }
  ]
}
```
总结一下，Pod中运行的应用访问K8s API服务器的流程：
* 应用验证API服务器的证书是否为整数机构所办法，这个证书就是ca.crt
* 应用凭借在token文件中持有的凭证来向API服务器证明自己的身份，并且通过Authorization标头来获得API服务器的授权；
* 当对Pod所在命名空间的API对象进行CRUD操作时，应该使用namespace文件来传递命名空间信息到API服务器。

## 4.3 Pod内部与K8s API交互（ambassador容器方式）

本机上可以通过kubectl proxy命令来使用代理，帮助完成证书和授权操作，减轻了用户的操作，同样的如果一个应用需要访问API服务器，除了直接交互外，可以在主容器运行的同时，启动一个ambassador容器，并在其中运行kubectl proxy命令，通过它来实现与API服务器的交互。

这种模式呀，运行在主容器中的应用不是直接与API服务器进行交互，而是通过HTTP协议与ambassador容器连接，并且由ambassador容器通过HTTPS协议来连接API服务器，这对于应用来说是透明的，本质上ambassador容器也是使用的默认凭证Secret卷中的文件。


## 4.3.1 演示过程

创建Pod
*curl-with-ambassador.yaml*
```
apiVersion: v1
kind: Pod
metadata:
  name: curl-with-ambassador
spec:
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "9999999"]
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
```

镜像luksa/kubectl-proxy:1.6.2便是ambassador容器，负责运行kubectl-proxy

创建Pod，并进入main容器中
```
sudo kubectl apply -f curl-with-ambassador.yaml
```
```
master@k8s-master:~/k8s-learning/resources/pod$ sudo kubectl exec -it curl-with-ambassador -c main -- bash
root@curl-with-ambassador:/# curl localhost:8001
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/admissionregistration.k8s.io/v1beta1",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1",
    "/apis/apiextensions.k8s.io/v1beta1",
    "/apis/apiregistration.k8s.io",
    "/apis/apiregistration.k8s.io/v1",
    "/apis/apiregistration.k8s.io/v1beta1",
    "/apis/apps",
    "/apis/apps/v1",
    "/apis/authentication.k8s.io",
    "/apis/authentication.k8s.io/v1",
    "/apis/authentication.k8s.io/v1beta1",
    "/apis/authorization.k8s.io",
    "/apis/authorization.k8s.io/v1",
    "/apis/authorization.k8s.io/v1beta1",
    "/apis/autoscaling",
    "/apis/autoscaling/v1",
    "/apis/autoscaling/v2beta1",
    "/apis/autoscaling/v2beta2",
    "/apis/batch",
    "/apis/batch/v1",
    "/apis/batch/v1beta1",
    "/apis/certificates.k8s.io",
    "/apis/certificates.k8s.io/v1",
    "/apis/certificates.k8s.io/v1beta1",
    "/apis/coordination.k8s.io",
    "/apis/coordination.k8s.io/v1",
    "/apis/coordination.k8s.io/v1beta1",
    "/apis/discovery.k8s.io",
    "/apis/discovery.k8s.io/v1beta1",
    "/apis/events.k8s.io",
    "/apis/events.k8s.io/v1",
    "/apis/events.k8s.io/v1beta1",
    "/apis/extensions",
    "/apis/extensions/v1beta1",
    "/apis/flowcontrol.apiserver.k8s.io",
    "/apis/flowcontrol.apiserver.k8s.io/v1beta1",
    "/apis/networking.k8s.io",
    "/apis/networking.k8s.io/v1",
    "/apis/networking.k8s.io/v1beta1",
    "/apis/node.k8s.io",
    "/apis/node.k8s.io/v1",
    "/apis/node.k8s.io/v1beta1",
    "/apis/policy",
    "/apis/policy/v1beta1",
    "/apis/rbac.authorization.k8s.io",
    "/apis/rbac.authorization.k8s.io/v1",
    "/apis/rbac.authorization.k8s.io/v1beta1",
    "/apis/scheduling.k8s.io",
    "/apis/scheduling.k8s.io/v1",
    "/apis/scheduling.k8s.io/v1beta1",
    "/apis/storage.k8s.io",
    "/apis/storage.k8s.io/v1",
    "/apis/storage.k8s.io/v1beta1",
    "/healthz",
    "/healthz/autoregister-completion",
    "/healthz/etcd",
    "/healthz/log",
    "/healthz/ping",
    "/healthz/poststarthook/aggregator-reload-proxy-client-cert",
    "/healthz/poststarthook/apiservice-openapi-controller",
    "/healthz/poststarthook/apiservice-registration-controller",
    "/healthz/poststarthook/apiservice-status-available-controller",
    "/healthz/poststarthook/bootstrap-controller",
    "/healthz/poststarthook/crd-informer-synced",
    "/healthz/poststarthook/generic-apiserver-start-informers",
    "/healthz/poststarthook/kube-apiserver-autoregistration",
    "/healthz/poststarthook/priority-and-fairness-config-consumer",
    "/healthz/poststarthook/priority-and-fairness-config-producer",
    "/healthz/poststarthook/priority-and-fairness-filter",
    "/healthz/poststarthook/rbac/bootstrap-roles",
    "/healthz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/healthz/poststarthook/start-apiextensions-controllers",
    "/healthz/poststarthook/start-apiextensions-informers",
    "/healthz/poststarthook/start-cluster-authentication-info-controller",
    "/healthz/poststarthook/start-kube-aggregator-informers",
    "/healthz/poststarthook/start-kube-apiserver-admission-initializer",
    "/livez",
    "/livez/autoregister-completion",
    "/livez/etcd",
    "/livez/log",
    "/livez/ping",
    "/livez/poststarthook/aggregator-reload-proxy-client-cert",
    "/livez/poststarthook/apiservice-openapi-controller",
    "/livez/poststarthook/apiservice-registration-controller",
    "/livez/poststarthook/apiservice-status-available-controller",
    "/livez/poststarthook/bootstrap-controller",
    "/livez/poststarthook/crd-informer-synced",
    "/livez/poststarthook/generic-apiserver-start-informers",
    "/livez/poststarthook/kube-apiserver-autoregistration",
    "/livez/poststarthook/priority-and-fairness-config-consumer",
    "/livez/poststarthook/priority-and-fairness-config-producer",
    "/livez/poststarthook/priority-and-fairness-filter",
    "/livez/poststarthook/rbac/bootstrap-roles",
    "/livez/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/livez/poststarthook/start-apiextensions-controllers",
    "/livez/poststarthook/start-apiextensions-informers",
    "/livez/poststarthook/start-cluster-authentication-info-controller",
    "/livez/poststarthook/start-kube-aggregator-informers",
    "/livez/poststarthook/start-kube-apiserver-admission-initializer",
    "/logs",
    "/metrics",
    "/openapi/v2",
    "/openid/v1/jwks",
    "/readyz",
    "/readyz/autoregister-completion",
    "/readyz/etcd",
    "/readyz/informer-sync",
    "/readyz/log",
    "/readyz/ping",
    "/readyz/poststarthook/aggregator-reload-proxy-client-cert",
    "/readyz/poststarthook/apiservice-openapi-controller",
    "/readyz/poststarthook/apiservice-registration-controller",
    "/readyz/poststarthook/apiservice-status-available-controller",
    "/readyz/poststarthook/bootstrap-controller",
    "/readyz/poststarthook/crd-informer-synced",
    "/readyz/poststarthook/generic-apiserver-start-informers",
    "/readyz/poststarthook/kube-apiserver-autoregistration",
    "/readyz/poststarthook/priority-and-fairness-config-consumer",
    "/readyz/poststarthook/priority-and-fairness-config-producer",
    "/readyz/poststarthook/priority-and-fairness-filter",
    "/readyz/poststarthook/rbac/bootstrap-roles",
    "/readyz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/readyz/poststarthook/start-apiextensions-controllers",
    "/readyz/poststarthook/start-apiextensions-informers",
    "/readyz/poststarthook/start-cluster-authentication-info-controller",
    "/readyz/poststarthook/start-kube-aggregator-informers",
    "/readyz/poststarthook/start-kube-apiserver-admission-initializer",
    "/readyz/shutdown",
    "/version"
  ]
}
```
主容器中的应用通过本机的8001端口，进入ambassador容器，然后ambassador容器负责加密、授权、服务器验证工作后，与API服务器通信。

# 5 总结

本章描述了Pod中的应用通过环境变量、DownwardAPI卷、以及与K8s API交互的方式获得Pod、容器甚至集群资源。


