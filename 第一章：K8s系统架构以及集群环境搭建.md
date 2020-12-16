# K8s系统架构


# K8s集群环境搭建

## 节点信息

本环境中包含3台机器（其中一台master,两台node）,详情如下，并使用kubeadm构建K8s集群环境。

```bash
     IP        Role    Hostname        OS  
10.10.10.147  Master  k8s-master  Ubuntu 16.04
10.10.10.148  Node    k8s-node1   Ubuntu 16.04
10.10.10.149  Node    k8s-node2   Ubuntu 16.04
```
## 系统配置更改（Master和Node）

### 禁用swap
```bash
sudo swapoff -a
```
同时把/etc/fstab包含swap那行记录注释掉。

### 关闭防火墙
```bash
sudo systemctl stop firewalld
sudo disable firewalld
```

### 禁用Selinux
```bash
sudo apt install selinux-utils
sudo setenforce 0
```

### 主机名更改

修改/etc/hostname，将里面的内容换为机器的主机名,如master节点
```bash
master@k8s-master:~$ vi /etc/hostname 
k8s-master
...
```

### 配置hosts文件

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

### 重启

完成上述操作后，重启机器

```bash
sudo reboot 
```

## 安装docker（Master和Node）

### 安装相关工具

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
```

### 添加密钥

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

### 安装docker

```bash
sudo apt-get install docker.io -y
```

### 查看docker版本

```bash
sudo docker version
```

### 启动docker服务

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

### 更换阿里云镜像仓库

创建/etc/docker/daemon.json，并添加如下内容
```bash
{
    "registry-mirrors": ["https://alzgoonw.mirror.aliyuncs.com"],
    "live-restore": true
}
```

### 重起docker服务

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```





