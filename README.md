# Kubernetes 基础学习笔记


> 该仓库是我自己学习 k8s 的笔记、总结、以及一些想法。
> 

## 1. k8s概述

k8s 本身涉及到大量的技术知识，包括操作系统、网络、存储、调度、分布式等方面的知识，这也正是技术人员学习与努力的方向。在学习之初，本系列文章不会着重讲解 Kubernetes的详细知识。而是尝试去了解Kubernetes的最基本的概念，并引导你基于官方的kubeadmin 工具搭建一个简单的Kubernetes集群。后续再循序渐进地进入k8s的系统学习。


k8s是Kubernetes的简称，来自Google，是用于自动部署、扩展和管理“容器化应用程序”的开源系统。简单地说就是：k8s 是一套服务器集群管理组件，k8s现在普遍用于管理集群节点上的容器。在学习k8s之前，我们应该具备一定的docker容器基础。


下面这张图展示了一个Kubernetes的一个典型的架构，你可能看不懂，但完全没关系，我们这里只是个了解，后面再介绍其中包含的技术点。

![Kubernetes](./img/Kubernetes.png)


## 2. k8s的功能

- 自我修复

- 弹性伸缩：实时根据服务器并发情况，实现自动增加或缩减容器数量

- 自动部署

- 回滚

- 服务发现和负载均衡

- 文件共享

......


## 3. k8s包含哪些组件？

**主控制节点(master node):**

- master节点需要安装以下组件：

- apiserver: 用于接收客户端操作k8s的指令

- schduler: 从多个woker节点组件中选举一个来启动服务

- controller manger: 向worker节点的kubelet组件发送指令

**工作节点(worker node):**

工作节点需要安装以下组件:

- kubenet：向docker发送指令管理docker容器

- kubeproxy：管理docker容器的网络

## 4. 学习前提

在正式学习使用kubernetes来管理你的容器应用之前，应当拥有一个Kubernetes集群环境。可以根据下面的笔记链接，我们第一步先用自己的电脑，搭建一个虚拟的Kubernetes集群。

## 5. 笔记目录

- [1.搭建k8s集群](./note/kb1-build.md)
- [2.k8s声明样式资源清单（YAML）文件](./note/kb2-yaml.md)
- [3.namespace](./note/kb3-namespace.md)
- [4.pod相关操作](./note/kb4-pod.md)
- [5.Controller](./note/kb5-controller.md)
- [6.Service](./note/kb6-service.md)
- [7.Ingress](./note/kb7-ingress.md)
- [8.存储](./note/kb8-storage.md)
