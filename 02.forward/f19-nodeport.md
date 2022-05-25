# 服务暴露之nodePort型service

## 1. 概述

k8s的dns实现了服务在集群内被自动发现，那如何使得服务在k8s集群外被使用和访问呢？实现方案如下：


- 使用NodePort型的Service：无法使用kube-proxy的ipvs模型，只能使用iptables模型


- 使用Ingress资源：只能调度并暴露7层应用，特指http和https协议


Ingress是k8s API的标准资源类型之一，也是一种核心资源，它其实就是一组基于域名和URL路径，把用户请求转发至指定Service资源的规则。可以将集群外部的流量请求转发至集群内部，从而实现“服务暴露”。Ingress控制器是能够为Ingress资源监听某套接字，然后根据Ingress规则匹配机制路由调度流量的一个组件。我们可以把Ingress理解为一个 nginx 加上一段 go语言代码。


常用的Ingress控制器的实现软件：

- Ingress-nginx
- HAProxy
- Trasefik

......



## 2. Nodeport 演示

