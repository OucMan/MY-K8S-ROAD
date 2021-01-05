# 1. 背景

K8s中提供了许多的资源实现应用的灵活配置，这里面的思想就是将配置和应用分离（解耦），各种各样的配置在K8s中以资源的形式存在，常见的配置资源类型如下：

* ConfigMap：ConfigMap存放容器中使用的可变配置（如环境变量），从而避免配置更改时重新编译容器镜像；
* Secret：Secret用来实现一些敏感信息的存储和使用，比如密码，token等；
* ServiceAccount：容器要访问集群自身，使用ServiceAccount实现身份认证；
* Resource：节点上运行的容器可能对资源（CPU\内存等）有一定的需求，通过Resource来控制；
* SecurityContext：节点上的容器是共享内核的，利用SecurityContext实现安全管控；
* InitContainer：业务容器启动之前的需要一个前置条件检验，比如容器启动前检测网络是否联通？这是可使用InitContainer；

# 2. ConfigMap

# 3. Secret

# 4. ServiceAccount

# 5. Resource

# 6. SecurityContext

# 7. InitContainer
