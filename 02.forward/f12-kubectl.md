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


**查看deployment**

使用以下命令查看所有名称空间的deployment

```shell-script
kubectl get pod -o wide --all-namespaces
```


或者使用以下命令指定查看`kube-public`名称空间的deployment

```shell
kubectl get pod -o wide -n kube-public
```