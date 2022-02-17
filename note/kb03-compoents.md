# k8s中的核心组件


## 1. k8s包含哪些组件？

**主控制节点(master node):**

- apiserver: 用于接收客户端操作k8s的指令

- schduler: 从多个woker节点组件中选举一个来启动服务

- controller manger: 向worker节点的kubelet组件发送指令


**工作节点(worker node):**

- kubenet：负责向docker发送指令管理docker容器

- kube-proxy：调度docker容器的网络


**CLI客户端:**

- kubectl：kubectl是一个用于操作kubernetes集群的命令行接口，通过利用kubectl的各种命令可以实现各种功能



**核心附件:**

- CNI网络插件：flannel/calico
- 服务发现插件：coredns
- 服务暴露插件：traefik
- GUI管理插件：Dashboard


## 2. 核心组件的相关功能


**apiserver**

- 提供了REST API接口，包括鉴权、数据校验、集群状态变更等

- 负责各个模块之间的数据交互，承担通信枢纽功能

- 资源配额控制的入口

- 提供完整的集群安全机制


**controller-manager**

- 由一组控制器组成，通过apiserver监控整个集群的状态，并确保集群处于预期的工作状态，常见的控制器如下：

1. Node Controller
2. Deployment Controller
3. Service Controller
4. Volume Controller
5. Endpoint Controller
6. Barbage Controller
7. Namespace Controller
8. Job Controller
9. Resurce quta Controller

.....


**scheluder***

- 主要是调度pod到适合的运算节点上，通过预算策略（predict）和 优选策略（priorities）等调度最合适的节点去运行 pod


**kubelet**

- kubelet 的主要功能就是定时从某个地方获取节点上Pod的期望状态（运行的容器、运行的副本数、网络配置、存储配置等），并调用对应的容器平台接口达到这个壮态。


- 负责定时汇报当前节点的状态给 `apiserver` ，以供调度的时候使用

- 负责镜像和容器的清理工作，保证节点上的镜像不会占满磁盘空间，退出的容器不会占用太多资源


**kube-proxy**

- k8s在每个节点上运行网络代理，service资源的载体

- 建立了Pod网络和集群网络的关系（clusterip -> podip）

- 常用的三种流量调度模式：Userspace（已废弃）、Iptables（濒临飞起）、Ipvs（推荐）


- 负责建立和删除包括更新调度规则、通知apiserver自己的更新，或者从apiserver那里获取其他kube-proxy的调度规则变化来更新自己的规则










