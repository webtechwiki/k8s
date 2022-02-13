# Kubernetes 基础学习笔记


> 该仓库是我自己学习 k8s 的笔记、总结、以及一些想法。
> 

## 1. 什么是k8s？

k8s 本身涉及到大量的技术知识，包括操作系统、网络、存储、调度、分布式等方面的知识，这也正是技术人员学习与努力的方向。在这系列的文章，我们从了解Kubernetes的最基本的概念开始，先基于官方的kubeadmin 工具搭建一个简单的Kubernetes集群，再循序渐进地进入k8s的系统学习。


k8s是Kubernetes的简称，来自Google，是用于自动部署、扩展和管理“容器化应用程序”的开源系统。简单地说就是：k8s 是一套服务器集群管理组件，k8s现在普遍用于管理集群节点上的容器。在学习k8s之前，我们应该具备一定的docker容器基础。


下面这张图展示了一个Kubernetes的一个典型的架构，你可能看不懂，但完全没关系，我们这里只是个了解，后面再介绍其中包含的技术点。

![Kubernetes](./img/Kubernetes.png)


## 2. k8s有哪些功能？

- 自我修复

- 弹性伸缩：实时根据服务器并发情况，实现自动增加或缩减容器数量

- 自动部署

- 回滚

- 服务发现和负载均衡

- 文件共享

......


## 3. 笔记目录

- [01.从快速搭建k8s集群开始学习k8s](./note/kb01-build.md)
- [02.k8s中的基础核心概念](./note/kb02-conception.md)
- [03.k8s中的核心组件](./note/kb03-compoents.md)
- [04.k8s声明样式资源清单（YAML）文件](./note/kb04-yaml.md)
- [05.namespace](./note/kb04-namespace.md)
- [06.pod相关操作](./note/kb06-pod.md)
- [07.Controller](./note/kb07-controller.md)
- [08.Service](./note/kb08-service.md)
- [09.Ingress](./note/kb09-ingress.md)
- [10.存储](./note/kb10-storage.md)




