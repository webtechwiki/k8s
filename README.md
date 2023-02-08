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

### 3.1 k8s基础概念

- [01.基础-使用kubeadm快速搭建k8s集群](./01.basic/01-build_in_virtual.md)
- [02.基础-k8s中的核心概念](./01.basic/02-conception.md)
- [03.基础-k8s中的核心组件](./01.basic/03-compoents.md)
- [04.基础-k8s资源配置清单](./01.basic/04-yaml.md)
- [05.基础-k8s名称空间](./01.basic/05-namespace.md)
- [06.基础-pod的基本操作](./01.basic/06-pod.md)
- [07.基础-k8s控制器](./01.basic/07-controller.md)
- [08.基础-k8s中的服务](./01.basic/08-service.md)
- [09.基础-使用Ingress将服务暴露到外网](./01.basic/09-ingress.md)
- [10.基础-k8s中的存储](./01.basic/10-storage.md)

### 3.2 在centos7中使用二进制包搭建k8s 1.15.2

- [01.进阶-k8s进阶知识概述](./02.forward/01-summary.md)
- [02.进阶-k8s二进制安装环境准备](./02.forward/02-prepare.md)
- [03.进阶-证书签发环境准备](./02.forward/03-sign-prepare.md)
- [04.进阶-通过二进制安装包安装docker](./02.forward/04-install-docker.md)
- [05.进阶-安装harhor服务](./02.forward/05-install-harbor.md)
- [06.进阶-安装etcd服务](./02.forward/06-install-etcd.md)
- [07.进阶-安装apiserver](./02.forward/07-install-apiserver.md)
- [08.进阶-安装L4反向代理服务](./02.forward/08-install-agent-server.md)
- [09.进阶-安装控制节点的其他组件](./02.forward/09-install-other-component.md)
- [10.进阶-安装kubectl](./02.forward/10-install-kubelet.md)
- [11.进阶-安装kube-proxy](./02.forward/11-install-kubeproxy.md)
- [12.进阶-cfssl证书工具介绍](./02.forward/12-cfssl-review.md)
- [13.进阶-声明式资源管理方法](./02.forward/13-kubectl-command.md)
- [14.进阶-陈述式资源管理方法](./02.forward/14-kubectl-yaml.md)
- [15.进阶-flannel网络插件](./02.forward/15-flannel-plugin.md)
- [16.进阶-flannel模型介绍](./02.forward/16-flannel-model.md)
- [17.进阶-flannel优化](./02.forward/17-flannel-optimize.md)
- [18.进阶-使用coredns实现服务发现](./02.forward/18-coredns.md)
- [19.进阶-服务暴露之nodePort型service](./02.forward/19-nodeport.md)
- [20.进阶-服务暴露之Ingress](./02.forward/20-ingress.md)

### 3.3 在debian中使用二进制包安装k8s 1.24.1

- [00.实战-k8s“微型集群”的实现方案](./03.actual_combat/README.md)
- [01.实战-k8s二进制安装环境准备](./03.actual_combat/01-prepare.md)
- [02.实战-安装containerd作为runtime](./03.actual_combat/02-install_containerd.md)
- [03.实战-签发SSL证书](./03.actual_combat/03-sign-prepare.md)
- [04.实战-安装etcd服务](./03.actual_combat/04-install-ectd.md)
- [05.实战-安装apiserver](./03.actual_combat/05-install-apiserver.md)
- [06.实战-搭建L4反向代理服务](./03.actual_combat/06-install-agent-server.md)
- [07.实战-安装controller-manager和scheduler](./03.actual_combat/07-install-other-component.md)
- [08.实战-安装kubectl](./03.actual_combat/08-install-kubelet.md)
- [09.实战-安装proxy](./03.actual_combat/09-install-kubeproxy.md)
- [10.实战-安装calico和coreDNS](./03.actual_combat/10-install-calico-coredns.md)
- [11.实战-安装traefik-ingress](./03.actual_combat/11-install-traefik.md)
- [12.实战-在k8s环境部署应用](./03.actual_combat/12-deploy-app.md)

### 3.4 扩展文章

- [01.k8s资源分类](./04.extend/01-k8s-resources.md)
- [02.使用二进制安装包安装docker](./04.extend/02-install-docker.md)
- [04.搭建harbor私有镜像仓库](./04.extend/04-install-harbor.md)
- [05.集群搭建IP网段规划建议](./04.extend/05-ip_suggestion.md)
- [06.解决centos6镜像启动容器失败的问题](./04.extend/06-run-centos6.md)