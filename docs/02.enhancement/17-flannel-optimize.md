# flannel优化


## 1. 需优化的原因


在我运行服务的时候，`kb22`这台主机上启动了一个名称`nginx-dp-f585689bf-649n7`的pod，可以使用以下命令监控它的日志，如下命令

```shell
kubectl logs nginx-dp-f585689bf-649n7 -n kube-public
```

在`kb21`这台主机启动了名称`nginx-dp-f585689bf-649n7`的pod，我们进去进入容器
```shell
kubectl exec -it nginx-dp-f585689bf-649n7 sh -n kube-public
```


在容器里使用以下命令请求`kb22`上的容器

```shell
curl 172.7.22.2
```


但我们在`kb22`的容器上看到的请求日志，来源地址是`kb21`这台宿主机的IP，原因是docker在访问其他IP时，做了SNAT源地址转换，使用其宿主机IP去访问其他IP。而实际上，我们希望看到容器的IP，并且在集群中，也必须是容器IP。


在`kb21`这台主机，我们可以使用`iptables-save`指令可以将内核中当前的iptables配置导出到标准输出。SNET转换是在iptables的`net`表的`postrouting`上做的。我们可以使用以下命令查看到相关信息

```shell
iptables-save | grep -i postrouting
```

可以看到返回如下内容

```shell
:POSTROUTING ACCEPT [0:0]
:KUBE-POSTROUTING - [0:0]
-A POSTROUTING -s 172.7.21.0/24 ! -o docker0 -j MASQUERADE
-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
-A KUBE-POSTROUTING -m comment --comment "Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose" -m set --match-set KUBE-LOOP-BACK dst,dst,src -j MASQUERADE
```

从以上信息可以看出

- 来源地址`172.7.21.0/24`，如果出网不是`docker0`这个网卡，将做原地址NAT转换
- 来源地址`172.17.0.0/16`，如果出网不是`docker0`这个网卡，将做原地址NAT转换


也就是说，以上两个IP段，在访问外网的时候，那么原地址一定会被NAT转换，也就是说会以宿主机的网络IP访问外网。

在docker集群中，这样的配置显然是不对的，我们希望docker访问其他主机的docker时，也不要进行NAT网络转换。在我们这个集群中，也就是希望`172.7.0.0/16`这个网段不被NAT地址转换。


## 2. 优化过程

**安装iptables**

我们需要在所有`kb21`和`kb22`执行以下操作，先安装`iptables-services`这个组件，如下命令

```shell
yum install -y iptables-services
```

装上`iptables-services`之后也会默认将`iptables`装上，我们将启动`iptables`并设置为开机自启，如下命令

```shell
# 启动iptables
systemctl start iptables
# 设置服务为开机自启
systemctl enable iptables
```


**删除默认的规则**

iptables默认的规则比较严格，我们需要删除默认的规则，我们先放通所有网络

```shell
# 放通所有网络
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -F
# 写入到系统配置中
iptables-save > /etc/sysconfig/iptables
```

删除掉规则，使用以下命令



**优化规则**


在1中，以`kb21`这台主机为例，我们看到有两个网段在访问外网时会被NAT转换，一是`172.17.0.0/16`，这是docker的容器默认网段，另外是`172.7.21.0/24`，这个我们设置的在`kb21`这台主机上容器启动默认的网段。所以，我们只需要优化`172.7.21.0/24`这个网段就可以了。我们先删除SANT转换规则，如下命令

```shell
iptables -t nat -D POSTROUTING -s 172.7.21.0/24 ! -o docker0 -j MASQUERADE
```

再添加以下规则

```shell
iptables -t nat -I POSTROUTING -s 172.7.21.0/24 ! -d 172.7.0.0/16 ! -o docker0 -j MASQUERADE
```

添加的规则代表的是：再本机上，针对源地址为`172.7.21.0/24`的网段，访问地址不是`172.7.0.0/16`网段时（该网段是集群中docker容器的网段），并且出口网卡不是`docker0`，才会进行SNAT网络转换。也就是说，如果我们在`kb21`这台主机上访问其他主机的容器IP，不会进行SNAT网络转换。访问外网如**百度的IP**才会进行网络转换。


执行以上操作之后，容器之间的网络通信就是容器的真实IP了。













