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


### 2.1. 修改kube-proxy代理模式

为了演示NodePort类型的service，我们先修改集群节点上 kube-proxy 的启动配置文件 `/opt/kubernetes/server/bin/kube-proxy.sh`，将 `--proxy-mode`从`ipvs`改为`iptables`，将`--ipvs-scheduler`改为`rr`，如下内容


```shell
#!/bin/bash
./kube-proxy \
  --cluster-cidr 172.7.0.0/16 \
  --hostname-override kb21 \
  --proxy-mode=iptables \
  --ipvs-scheduler=rr \
  --kubeconfig ./conf/kube-proxy.kubeconfig
```


修改好了kube-proxy的启动配置文件之后，我们重启kube-proxy即可。



我们再一下命令查看 kube-proxy 的关键日志

```shell
# /data/logs/kubernetes/kube-proxy/kube-proxy.stdout.log 为supervisord配置文件指定的日志文件路径
tail -fn 200 /data/logs/kubernetes/kube-proxy/kube-proxy.stdout.log
```

如果看到`Using iptables Proxier.`关键字，代表启动成功


### 2.2. 删除ipvs代理规则


因为我们在前面使用了ipvs作为集群的网络代理模式，所以ipvs创建了一些代理规则，我们先在`kb21`和`kb22`上删除掉ipvs茶u你感觉爱你的代理规则，使用如下命令查看代理规则列表

```shell
ipvsadm -Ln
```

可以看到类似以下的返回内容，显示了当前主机上的所有代理规则

```shell
[root@kb21 bin]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.0.1:443 nq
  -> 192.168.14.21:6443           Masq    1      0          0         
  -> 192.168.14.22:6443           Masq    1      1          0         
TCP  192.168.0.2:53 nq
  -> 172.7.21.3:53                Masq    1      0          0         
TCP  192.168.0.2:9153 nq
  -> 172.7.21.3:9153              Masq    1      0          0         
TCP  192.168.59.60:80 nq
  -> 172.7.21.4:80                Masq    1      0          0         
  -> 172.7.22.3:80                Masq    1      0          0         
TCP  192.168.61.100:80 nq
  -> 172.7.21.4:80                Masq    1      0          0         
  -> 172.7.22.3:80                Masq    1      0          0         
TCP  192.168.163.187:80 nq
  -> 172.7.21.4:80                Masq    1      0          0         
  -> 172.7.22.3:80                Masq    1      0          0         
UDP  192.168.0.2:53 nq
  -> 172.7.21.3:53                Masq    1      0          0
```

实际上`kb21`和`kb22`这两台主机的代理规则都一样，一起执行如下删除ipvs代理规则命令即可

```shell
# 删除tpc规则列表
ipvsadm -D -t 192.168.0.1:443
ipvsadm -D -t 192.168.0.2:53
ipvsadm -D -t 192.168.0.2:9153
ipvsadm -D -t 192.168.59.60:80
ipvsadm -D -t 192.168.61.100:80
ipvsadm -D -t 192.168.163.187:80
# 删除udp代理
ipvsadm -D -u 192.168.0.2:53
```


### 2.3. 使用NodePort做服务暴露

> p12  08:48






