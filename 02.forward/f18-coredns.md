# coredns

## 1. 概述

简单来说，服务发现就是服务（应用）之间相互定位的过程。


服务发现并非云计算时代独有，传统的单架构时代也会用到。在以下应用场景下，更需要服务发现

- 服务（应用）的动态性强

- 服务（应用）更新发布频繁

- 服务（应用）支持自动伸缩


在跨k8s集群里，pod的IP是不段变化的，如何“以不变应万变”呢？

- 在k8s里，抽象出了Service资源，通过标签选择器，关联一组Pod
- 抽象出了集群网络，通过相对固定的“集群IP”，使服务接入点固定

那么如何自定关联Service资源的“名称”和“集群网络IP”，从而达到服务被集群自动发现的目的呢？

在传统的DNS模型里，我们可以通过`hdss7-21.host.com`解析道`192.168.14.21`，实际上，在集群里我们也可以建立类似的模型，让 `nginx-ds`自动关联到一个虚拟的Service IP。在k8s里，实现服务发现的方式就是通过DNS。以下工具可以实现k8s的dns服务

`kube-dns`: kubernetes-v1.2至kubernetes-v1.10
`coredns`: kubernetes-v1.11至今

不过要注意的是，k8s里的dns不是万能的！它只负责自动维护“服务名”和“集群网络IP”之间的关系。在本文，我们将使用coredns实现服务的发现的过程。在下文中，我们将以k8s资源的形式，来搭建coredns服务。


## 2. 搭建yaml配置文件的http服务

**搭建http服务**

为了更好的复用yaml配置文件，我们在`kb200`这台主机上配置一个yaml文件的在线http服务。创建nginx配置文件`/etc/nginx/conf.d/k8s-yaml.od.com.conf`，并添加以下内容

```shell
server {
	listen 80;
	server_name k8s-yaml.od.com

	location / {
		autoindex on;
		default_type text/plain;
		root /data/k8s-yaml;
	}
}
```

重新加载nginx

```shell
nginx -s reload
```


现在我们回到`kb11`这台主机上，添加一条解析，修改dns配置文件`/var/named/od.com.zone`，如下内容

```shell
$ORIGIN od.com.
$TTL 600 ; 10 minutes
@ IN SOA dns.od.com. dnsadmin.od.com. (
    2022012005 ; serial
    10800      ; refresh (3 hours)
    900        ; retry (15 minutes)
    604800     ; expire (1 week)
    86400      ; minium (1 day)
    ) 
    NS dns.od.com.

$TTL 60    ;   1 minute
dns    A   192.168.14.11
harbor    A   192.168.14.200
k8s-yaml    A   192.168.14.200
```

修改的内容为

- `2022012005 ; serial` : 增加序列号 
- `k8s-yaml    A   192.168.14.200` : 添加A记录，将`k8s-yaml`解析到`192.168.14.200`，即解析到运维主机上


重启域名解析服务

```shell
systemctl restart named
```

在使用以下命令检查是否已经正常解析

```shell
dig -t A kb21.host.com @192.168.14.11 +short
```

如果有返回，则代表已经设置成功。


**创建coredns的资源配置清单**



回到`kb200`这台主机，使用以下命令创建coredns配置清单的目录

```shell
mkdir -p /data/k8s-yaml/coredns
```

创建coredns的资源配置清单`/data/k8s-yaml/coredns/rbac.yaml`

```shell
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
```


创建configmap配置清单`/data/k8s-yaml/coredns/cm.yaml`


```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        log
        health
        ready
        kubernetes cluster.local 192.168.0.0/16  #service资源cluster地址
        forward . 10.4.7.11   #上级DNS地址
        cache 30
        loop
        reload
        loadbalance
       }
```


创建depoly控制器清单`/data/k8s-yaml/coredns/dp.yaml`

```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: coredns
  template:
    metadata:
      labels:
        k8s-app: coredns
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      containers:
      - name: coredns
        image: harbor.zq.com/public/coredns:v1.6.1
        args:
        - -conf
        - /etc/coredns/Corefile
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
```

创建service资源清单`/data/k8s-yaml/coredns/svc.yaml`


```shell
apiVersion: v1
kind: Service
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: coredns
  clusterIP: 192.168.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
  - name: metrics
    port: 9153
    protocol: TCP
```


## 3. 启动coredns服务

我们再回到`kb21`这台集群节点主机上，使用以下命令将资源挨个创建出来

```shell
kubectl create -f http://k8s-yaml.zq.com/coredns/rbac.yaml
kubectl create -f http://k8s-yaml.zq.com/coredns/cm.yaml
kubectl create -f http://k8s-yaml.zq.com/coredns/dp.yaml
kubectl create -f http://k8s-yaml.zq.com/coredns/svc.yaml
```

验证服务

```shell
# 查看所有资源
kubectl get all -n kube-system
# 查看kube-system名称空间的资源
kubectl get svc -o wide -n kube-system
```

验证域名解析

```shell-script
# 验证公网解析
dig -t A www.baidu.com @192.168.0.2 +short
# 验证自建解析
dig -t A harbor.zq.com @192.168.0.2 +short
```

coredns已经能解析外网域名了,因为coredns的配置中,写了他的上级DNS为10.4.7.11,如果它自己解析不出来域名,会通过递归查询一级级查找
但coredns我们不是用来做外网解析的,而是用来做service名和serviceIP的解析


## 4. 验证服务


我们先将集群中自己创建的pod都删除掉，再重新创建一个`deployment`类型的pod

```shell
kubectl create deployment nginx-dp --image=nginx:alpine -n kube-public
```

给pod创建一个service

```shell
kubectl expose deployment nginx-dp --port=80 -n kube-public
```

我们先使用`dig -t A nginx-dp @192.168.0.2 +short`命令进行验证，发现无返回数据，其实是需要service的完整域名:`服务名.名称空间.svc.cluster.local.`，再使用以下命令进行验证，数据就正常了

```shell
dig -t A nginx-dp.kube-public.svc.cluster.local. @192.168.0.2 +short
```

> 可以看到我们没有手动添加任何解析记录，我们nginx-dp的service资源的IP，已经被解析了，这就是coredns帮我们做的


如果我们进入pod里面，使用`ping nginx-dp`进行测试，发现是通的，原因是我们启动的pod，dns地址是我们前面设置的coredns地址，以及搜索域中已经添加了搜索域：`kube-public.svc.cluster.local`。


这样，我们就解决了集群内部服务相互访问的问题，在后续，我们将介绍如何将我们的内部服务暴露在外网。





