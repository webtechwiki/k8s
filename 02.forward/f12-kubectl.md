# kubectl的使用

在k8s中，各种服务可以被称为资源，管理k8s资源，无非就是对k8s的各种资源进行增删改查。在本文我们将介绍常见的资源管理命令。


## 1. 陈述式资源管理方法


**查看名称空间**

查看k8s集群中所有的名称空间使用以下命令

```shell
kubectl get namespaces
```

或者

```shell
kubectl get ns
```


**创建名称空间**

创建名称为`app`的名称空间

```shell
kubectl create namespace app
```

**删除名称空间**


```shell
kubectl delete namespace app
```

**创建deployment**

使用`nginx:alpine`在`kube-public`这个名称空间创建deployment，如下命令

```shell
kubectl create deployment nginx-dp --image=nginx:alpine -n kube-public
```


**查看pod**


创建deployment的时候，会创建对应的pod，使用以下命令查看所有名称空间的pod


```shell-script
kubectl get pod -o wide --all-namespaces
```


或者使用以下命令指定查看`kube-public`名称空间的pod

```shell
kubectl get pod -o wide -n kube-public
```


**查看deployment**

查看 kube-public 名称空间的的 deployment

```shell
kubectl get deploy -n kube-public
```


查看 kube-public 名称空间的 deployment 并附带扩展信息

```shell
kubectl get deployment -o wide -n kube-public
```


查看deployment详细信息


```shell-script
kubectl describe deployment nginx-dp -n kube-public
```





**进入pod**


进入`nginx-dp-f585689bf-prp27`这个pod，如下命令


```shell
kubectl exec -it nginx-dp-f585689bf-prp27 sh -n kube-public
```





**删除pod**

删除`nginx-dp-f585689bf-prp27`这个pod，如下命令

```shell
kubectl delete pod nginx-dp-f585689bf-prp27 -n kube-public
```

不过，我们删除之后k8s会重新启动一个新的pod，因为我们的前面创建的资源就是期望它正常运行


**删除deployment**

删除我们上上面创建的deployment，如下命令

```shell
kubectl delete deploy nginx-dp -n kube-public
```

执行deployment删除命令之后，pod也跟着被删除了



**弹性伸缩**

为了模拟向deployment向外望暴露端口，我们想使用创建 deployment，如下命令

```shell-script
kubectl create deployment nginx-dp --image=nginx:alpine -n kube-public
```

我们使用以下命令创建将deployment 设置为两个副本

```shell
kubectl scale deployment nginx-dp --replicas=2 -n kube-public
```


**暴露端口**

向外暴露80端口，如下命令

```shell
kubectl expose deployment nginx-dp --port 80 -n kube-public
```

此时创建了对应的service，使用如下命令查看

```shell
kubectl describe svc nginx-dp -n kube-public
```

可以看到返回如下内容

```shell
[root@kb21 vagrant]# kubectl describe svc nginx-dp -n kube-public
Name:              nginx-dp
Namespace:         kube-public
Labels:            app=nginx-dp
Annotations:       <none>
Selector:          app=nginx-dp
Type:              ClusterIP
IP:                192.168.11.76
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         172.7.21.2:80,172.7.22.2:80
Session Affinity:  None
Events:            <none>
```

