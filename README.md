# k8s入门与进阶实战

## 一、简介

### 1.1 作者信息

- **博客**：[https://jiker.dev](https://jiker.dev)
- **公众号**：极客开发者

### 1.2 文档概述

- **阅读对象**：后端工程师、运维工程师、Linux爱好者、容器技术爱好者、k8s爱好者
- **阅读条件**：熟悉Linux环境（越熟悉越好）、熟悉容器技术（如docker、containerd）

## 二、什么是k8s？

k8s本身涉及到大量的技术知识，包括操作系统、网络、存储、调度、分布式等方面的知识，这也正是技术人员学习与努力的方向。在这系列的文章，我们从了解Kubernetes的最基本的概念开始，先使用官方的`kubeadm`工具搭建一个简单的Kubernetes集群，再循序渐进地进入k8s的系统学习。

k8s是Kubernetes的简称，来自Google，是用于自动部署、扩展和管理“容器化应用程序”的开源系统。简单地说就是：k8s是一套服务器集群管理组件，k8s现在普遍用于管理集群节点上的容器。在学习k8s之前，我们应该具备一定的容器知识基础，在本系列文章中特指`docker`。

下面这张图展示了一个Kubernetes的一个典型的架构，你可能看不懂，但完全没关系，我们这里只是个了解，后面再介绍其中包含的技术点。

![Kubernetes](img/01-kubernetes.png)

## 三、k8s有哪些功能？

- 自我修复

- 弹性伸缩：实时根据服务器并发情况，实现自动增加或缩减容器数量

- 自动部署

- 回滚

- 服务发现和负载均衡

- 文件共享

......

## 三、章节

### 3.1 k8s基础概念

- [使用kubeadm快速搭建k8s集群](basic/01-build_in_virtual.md)
- [k8s中的核心概念](basic/02-conception.md)
- [k8s中的核心组件](basic/03-compoents.md)
- [k8s资源配置清单](basic/04-yaml.md)
- [k8s名称空间](basic/05-namespace.md)
- [pod的基本操作](basic/06-pod.md)
- [k8s控制器](basic/07-controller.md)
- [k8s中的服务](basic/08-service.md)
- [使用Ingress将服务暴露到外网](basic/09-ingress.md)
- [k8s中的存储](basic/10-storage.md)

### 3.2 在centos7中使用二进制包搭建k8s 1.15.2

- [k8s进阶知识概述](enhancement/01-summary.md)
- [k8s二进制安装环境准备](enhancement/02-prepare.md)
- [证书签发环境准备](enhancement/03-sign-prepare.md)
- [通过二进制安装包安装docker](enhancement/04-install-docker.md)
- [安装harhor服务](enhancement/05-install-harbor.md)
- [安装etcd服务](enhancement/06-install-etcd.md)
- [安装apiserver](enhancement/07-install-apiserver.md)
- [安装L4反向代理服务](enhancement/08-install-agent-server.md)
- [安装控制节点的其他组件](enhancement/09-install-other-component.md)
- [安装kubectl](enhancement/10-install-kubelet.md)
- [安装kube-proxy](enhancement/11-install-kubeproxy.md)
- [cfssl证书工具介绍](enhancement/12-cfssl-review.md)
- [声明式资源管理方法](enhancement/13-kubectl-command.md)
- [陈述式资源管理方法](enhancement/14-kubectl-yaml.md)
- [flannel网络插件](enhancement/15-flannel-plugin.md)
- [flannel模型介绍](enhancement/16-flannel-model.md)
- [flannel优化](enhancement/17-flannel-optimize.md)
- [使用coredns实现服务发现](enhancement/18-coredns.md)
- [服务暴露之nodePort型service](enhancement/19-nodeport.md)
- [服务暴露之Ingress](enhancement/20-ingress.md)

### 3.3 在debian中使用二进制包安装k8s 1.24.1

- [k8s“微型集群”的实现方案](ultimate/README.md)
- [k8s二进制安装环境准备](ultimate/01-prepare.md)
- [安装containerd作为runtime](ultimate/02-install_containerd.md)
- [签发SSL证书](ultimate/03-sign-prepare.md)
- [安装etcd服务](ultimate/04-install-ectd.md)
- [安装apiserver](ultimate/05-install-apiserver.md)
- [搭建L4反向代理服务](ultimate/06-install-agent-server.md)
- [安装controller-manager和scheduler](ultimate/07-install-other-component.md)
- [安装kubectl](ultimate/08-install-kubelet.md)
- [安装proxy](ultimate/09-install-kubeproxy.md)
- [安装calico和coreDNS](ultimate/10-install-calico-coredns.md)
- [安装traefik-ingress](ultimate/11-install-traefik.md)
- [在k8s环境部署应用](ultimate/12-deploy-app.md)

### 3.4 扩展文章

- [k8s资源分类](extend/01-k8s-resources.md)
- [使用二进制安装包安装docker](extend/02-install-docker.md)
- [搭建harbor私有镜像仓库](extend/04-install-harbor.md)
- [集群搭建IP网段规划建议](extend/05-ip_suggestion.md)
- [解决centos6镜像启动容器失败的问题](extend/06-run-centos6.md)
