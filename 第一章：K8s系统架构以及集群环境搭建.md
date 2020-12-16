# 1、K8s系统架构


# 2、K8s集群环境搭建

## 2.1 节点信息

本环境中包含3台机器（其中一台master,两台node）,详情如下，并使用kubeadm构建K8s集群环境。

```bash
     IP        Role    Hostname        OS  
10.10.10.147  Master  k8s-master  Ubuntu 16.04
10.10.10.148  Node    k8s-node1   Ubuntu 16.04
10.10.10.149  Node    k8s-node2   Ubuntu 16.04
```
## 2.2 系统配置更改（Master和Node）

### 2.2.1 禁用swap
```bash
sudo swapoff -a
```
同时把/etc/fstab包含swap那行记录注释掉。

### 2.2.2 关闭防火墙
```bash
sudo systemctl stop firewalld
sudo disable firewalld
```

### 2.2.3 禁用Selinux
```bash
sudo apt install selinux-utils
sudo setenforce 0
```

### 2.2.4 主机名更改

修改/etc/hostname，将里面的内容换为机器的主机名,如master节点
```bash
master@k8s-master:~$ vi /etc/hostname 
k8s-master
...
```

### 2.2.5 配置hosts文件

/etc/hosts存放的是域名与ip的对应关系，将三个机器的主机名和ip添加进入，如master节点
```bash
master@k8s-master:~$ vi /etc/hosts

127.0.0.1       localhost
127.0.1.1       k8s-master
10.10.10.147    k8s-master
10.10.10.148    k8s-node1
10.10.10.149    k8s-node2

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
...        
```

### 2.2.6 重启

完成上述操作后，重启机器

```bash
sudo reboot 
```

## 2.3 安装docker（Master和Node）

### 2.3.1 安装相关工具

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
```

### 2.3.2 添加密钥

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

### 2.3.3 安装docker

```bash
sudo apt-get install docker.io -y
```

### 2.3.4 查看docker版本

```bash
sudo docker version
```

### 2.3.5 启动docker服务

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

### 2.3.6 更换阿里云镜像仓库

创建/etc/docker/daemon.json，并添加如下内容
```bash
{
    "registry-mirrors": ["https://alzgoonw.mirror.aliyuncs.com"],
    "live-restore": true
}
```

### 2.3.7 重起docker服务

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 2.4 安装kubectl，kubelet，kubeadm（Master和Node）

### 2.4.1 添加秘钥
```bash
sudo curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
```
### 2.4.2 添加阿里云源
```bash
sudo tee /etc/apt/sources.list.d/kubernetes.list <<-'EOF'
deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
EOF
sudo apt-get update
```
### 2.4.3 安装kubectl，kubelet，kubeadm
```bash
sudo apt-get install -y kubelet kubeadm kubectl
```

## 2.5 配置Master

现在执行kubeadm init命令会失败，因为k8s.gcr.io被墙了，导致拉取镜像失败，因此选择在阿里云镜像仓库首先将相应版本的镜像下载下来。

### 2.5.1 查看所需的镜像的版本

```bash
master@k8s-master:~$ sudo kubeadm  config images list
[sudo] password for master: 
k8s.gcr.io/kube-apiserver:v1.20.0
k8s.gcr.io/kube-controller-manager:v1.20.0
k8s.gcr.io/kube-scheduler:v1.20.0
k8s.gcr.io/kube-proxy:v1.20.0
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns:1.7.0
```

### 2.5.2 利用阿里云镜像仓库拉取镜像

```bash
sudo docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.0
sudo docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.0
sudo docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.0
sudo docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0
sudo docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0
sudo docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
sudo docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.0
```

### 2.5.3 Tag镜像

通过Tag来自阿里云的镜像使得与kubeadm init命令拉取的镜像名字相同

```bash
sudo docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.0 k8s.gcr.io/kube-controller-manager:v1.20.0
sudo docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.0 k8s.gcr.io/kube-scheduler:v1.20.0
sudo docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.0 k8s.gcr.io/kube-proxy:v1.20.0
sudo docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2
sudo docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0
sudo docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0
sudo docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.0 k8s.gcr.io/kube-apiserver:v1.20.0
```

### 2.5.4 kubeadm init
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=10.10.10.147 --kubernetes-version=v1.20.0 --ignore-preflight-errors=Swap
```
执行成功后，会得到使用集群之前需要执行的命令，以及节点加入集群的命令，现在只需要执行使用集群之前需要执行的命令，如下

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
节点加入集群的命令在配置Node时使用

### 2.5.5 安装网络插件

网络插件选择flannel（其机制后续章节会介绍）。首先将kube-flannel.yml文件下载到本地，因为quay.io镜像地址在国内访问不了，还是选择在阿里云镜像仓库将yml中需要的镜像提前下载到本地，然后改变标签。

#### 2.5.5.1 查看yaml需要的镜像

```bash
master@k8s-master:~$ cat kube-flannel.yml |grep image|uniq
        image: quay.io/coreos/flannel:v0.12.0-amd64
        image: quay.io/coreos/flannel:v0.12.0-arm64
        image: quay.io/coreos/flannel:v0.12.0-arm
        image: quay.io/coreos/flannel:v0.12.0-ppc64le
        image: quay.io/coreos/flannel:v0.12.0-s390x
```

#### 2.5.5.2 通过阿里云仓库下载对应的镜像

```bash
sudo docker pull registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-amd64
sudo docker pull registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-arm64
sudo docker pull registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-arm
sudo docker pull registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-ppc64le
sudo docker pull registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-s390x
```

#### 2.5.5.3 Tag镜像

创建与yaml文件所需镜像一样的名字

```bash
sudo docker tag registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-amd64 quay.io/coreos/flannel:v0.12.0-amd64
sudo docker tag registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-arm64 quay.io/coreos/flannel:v0.12.0-arm64
sudo docker tag registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-arm quay.io/coreos/flannel:v0.12.0-arm
sudo docker tag registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-ppc64le quay.io/coreos/flannel:v0.12.0-ppc64le
sudo docker tag registry.cn-shanghai.aliyuncs.com/leozhanggg/flannel:v0.12.0-s390x quay.io/coreos/flannel:v0.12.0-s390x
```

#### 2.5.5.4 加载flannel

```bash
sudo kubectl apply -f kube-flannel.yml
```

## 2.6 配置Node






