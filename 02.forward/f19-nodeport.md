# 服务暴露之nodePort型service

## 1. 概述

k8s的dns实现了服务在集群内被自动发现，那如何使得服务在k8s集群外被使用和访问呢？实现方案如下：


- 使用NodePort型的Service：无法使用kube-proxy的ipvs模型，只能使用iptables模型


- 使用Ingress资源：只能调度并暴露7层应用，特指http和https协议。Ingress在后面一节会介绍




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

创建资源文件`nginx-ds-svc-nodeport.yml`，并添加以下内容

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-ds
  name: nginx-ds
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
    nodePort: 8000
    name: http
  selector:
    app: nginx-ds
  sessionAffinity: None
  type: NodePort
```

上面的资源文件定义了NodePort类型的Service，将宿主机的`8000`端口绑定到名为`nginx-ds`的pod，使用以下命令创建service

```shell
kubectl create -f nginx-ds-svc-nodeport.yml
```


查看已创建service，如下命令

```shell
kubectl get svc -o wide
```

我们还可以k8s节点使用以下命令查看 8000 端口 的使用情况

```shell
netstat -luntp | grep 8000
```

可以看到如下的返回内容

```shell
[root@kb21 vagrant]# netstat -luntp | grep 8000
tcp6       0      0 :::8000                 :::*                    LISTEN      31190/./kube-proxy
```

此时我们宿主机的 8000 端口映射到 pod 上。我们可以通过宿主机8000端口成功访问到 nginx。如果我们使用如下命令删除nodeport，将不再能通过宿主机 8000 端口来访问服务。

```shell
kubectl delete -f nginx-ds-svc-nodeport.yml
```

### 2.4. 恢复kube-proxy的初始配置


使用Nodeport向外望暴露服务有一个严重的缺点就是，每当暴露一个服务就会占用一个宿主机端口。所以 这个服务暴露方式并不适合常规的需求。在后面一节我们会介绍 Ingress 来代替 Nodeport。我们需要在不同的集群节点的 kube-proxy 启动参数设置为初始值，以`kb21`节点为示例，启动脚本内容如下


```shell
#!/bin/bash
./kube-proxy \
  --cluster-cidr 172.7.0.0/16 \
  --hostname-override kb21 \
  --proxy-mode=ipvs \
  --ipvs-scheduler=nq \
  --kubeconfig ./conf/kube-proxy.kubeconfig
```


重启服务代理模式又回到初始状态

> 但要注意的是，修改代理模式又改回来之后，集群可能出现问题，我们可以重启一下 docker，并重启一下kubelet。




