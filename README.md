# k8s入门与进阶实战

## 一、概述

- **博客**：[https://jiker.dev](https://jiker.dev)
- **公众号**：极客开发者

**阅读对象**：后端工程师、运维工程师、Linux爱好者、k8s爱好者

## 二、什么是k8s？

k8s本身涉及到大量的技术知识，包括操作系统、网络、存储、调度、分布式等方面的知识，这也正是技术人员学习与努力的方向。在这系列的文章，我们从了解Kubernetes的最基本的概念开始，先使用官方的`kubeadm`工具搭建一个简单的Kubernetes集群，再循序渐进地进入k8s的系统学习。

k8s是Kubernetes的简称，来自Google，是用于自动部署、扩展和管理“容器化应用程序”的开源系统。简单地说就是：k8s是一套服务器集群管理组件，k8s现在普遍用于管理集群节点上的容器。在学习k8s之前，我们应该具备一定的容器知识基础，在本系列文章中特指`docker`。

下面这张图展示了一个Kubernetes的一个典型的架构，你可能看不懂，但完全没关系，我们这里只是个了解，后面再介绍其中包含的技术点。

![Kubernetes](./img/01-kubernetes.png)

## 三、k8s有哪些功能？

- 自我修复

- 弹性伸缩：实时根据服务器并发情况，实现自动增加或缩减容器数量

- 自动部署

- 回滚

- 服务发现和负载均衡

- 文件共享

......

## 三、章节

### 3.1 第一章：k8s基础概念与学习环境快速搭建

- [01.在虚拟机中使用kubeadm快速搭建k8s集群](./01.basic/kb01-build_in_virtual.md)
- [02.k8s中的核心概念](./01.basic/kb02-conception.md)
- [03.k8s中的核心组件](./01.basic/kb03-compoents.md)
- [04.k8s资源清单文件](./01.basic/kb04-yaml.md)
- [05.k8s名称空间](./01.basic/kb05-namespace.md)
- [06.pod的相关操作](./01.basic/kb06-pod.md)
- [07.Controller](./01.basic/kb07-controller.md)
- [08.Service](./01.basic/kb08-service.md)
- [09.Ingress](./01.basic/kb09-ingress.md)
- [10.存储](./01.basic/kb10-storage.md)
- [11.IP网段规划建议](./01.basic/kb11-ip_suggestion.md)

### 3.2 第二章：在centos7中使用二进制包搭建k8s 1.15.2

- [01.k8s进阶知识概述](./02.forward/f01-summary.md)
- [02.k8s二进制安装环境准备](./02.forward/f02-prepare.md)
- [03.证书签发环境准备](./02.forward/f03-sign-prepare.md)
- [04.通过二进制安装包安装docker](./02.forward/f04-install-docker.md)
- [05.安装harhor服务](./02.forward/f05-install-harbor.md)
- [06.安装etcd服务](./02.forward/f06-install-etcd.md)
- [07.安装apiserver](./02.forward/f07-install-apiserver.md)
- [08.安装L4反向代理服务](./02.forward/f08-install-agent-server.md)
- [09.安装控制节点的其他组件](./02.forward/f09-install-other-component.md)
- [10.安装kubectl](./02.forward/f10-install-kubelet.md)
- [11.安装kube-proxy](./02.forward/f11-install-kubeproxy.md)
- [12.回顾cfssl证书工具](./02.forward/f12-cfssl-review.md)
- [13.声明式资源管理方法](./02.forward/f13-kubectl-command.md)
- [14.陈述式资源管理方法](./02.forward/f14-kubectl-yaml.md)
- [15.flannel网络插件](./02.forward/f15-flannel-plugin.md)
- [16.flannel模型介绍](./02.forward/f16-flannel-model.md)
- [17.flannel优化](./02.forward/f16-flannel-optimize.md)
- [18.使用coredns实现服务发现](./02.forward/f18-coredns.md)
- [19.服务暴露之nodePort型service](./02.forward/f19-nodeport.md)
- [20.服务暴露之Ingress](./02.forward/f20-ingress.md)

### 3.3 第三章：在debian 11中使用二进制包安装k8s 1.24.1

序言-[搭建k8s“微型集群”的实现方案](./03.real_install/README.md)

- [01.k8s二进制安装环境准备](./03.real_install/f01-prepare.md)
- [02.安装containerd作为runtime](./03.real_install/f02-install_containerd.md)
- [03.签发SSL证书](./03.real_install/f03-sign-prepare.md)
- [04.安装etcd服务](./03.real_install/f04-install-ectd.md)
- [05.安装apiserver](./03.real_install/f05-install-apiserver.md)
- [06.搭建L4反向代理服务](./03.real_install/f06-install-agent-server.md)
- [07.安装controller-manager和scheduler](./03.real_install/f07-install-other-component.md)
- [08.安装kubectl](./03.real_install/f08-install-kubelet.md)
- [09.安装proxy](./03.real_install/f09-install-kubeproxy.md)
- [10.安装calico和coreDNS](./03.real_install/f10-install-calico-coredns.md)
- [11.安装traefik-ingress](./03.real_install/f11-install-traefik.md)
- [12.在k8s环境部署应用](./03.real_install/f12-deploy-app.md)

- [13.扩展:使用二进制安装包安装docker](./03.real_install/f13-install-docker.md)
- [14.扩展:安装harbor](./03.real_install/f14-install-harbor.md)

### 3.4 第四章：k8s的资源

- [01.序言](./04.resource/f01-prepare.md)
