# Controller

> 当我们删除Pod时是可以直接删除的，如果生产过程中误操作，Pod同样也会被轻易删除，因此我们需要在k8s集群中引入另一种概念：控制器，用于在k8s 集群中以loop方式监视Pod状态，如果发现Pod被删除，将重新拉起一个Pod，以让Pod一直保持在用户期望状态

本节目标
* 了解Controller
* 了解Controller分类
* 了解Deployment控制器作用
* 掌握创建Deloypment控制器型应用方法
* 掌握删除Deployment控制器型应用方法

## 一、相关概念
### 1. 介绍
* Controller是k8s集群中的控制器组件
* 用于对应用运行的资源对象进行监控
* 当Pod出现问题时，会把Pod重新拉起，以达到用户期望的状态

### 2. 分类
常见的Pod控制器分类
**Deployment**：声明式更新控制器，用于发布无状态应用

**ReplicaSet**：副本集控制器，用于对Pod进行副本规模扩大或剪裁

**StatefulSet**：有状态副本集，用于发布有状态应用

**DaemonSet**：在k8s集群每一个Node上运行一个副本，用于发布监控或日志收集类等应用

**Job**：运行一次性作业任务

**CronJob**：运行周期性作业任务

## 二、相关命令
### 1. 创建
新版本k8s已经启用命令创建，我们使用资源文件创建，新建`create-deployment-nginx-app2.yaml`文件，写入如下内容：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app2
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginxapp
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        ports: 
        - containerPort: 80
```
使用以下命令进行创建
```shell
kubectl apply -f create-deployment-nginx-app2.yaml
```

### 2. 查看
查看创建的deployment
```shell
kubectl get deployment.apps
```
或者查看replicas
```shell
kubectl get rs
```
或通过查看pod详情
```shell
kubectl get pods -o wide
```

### 3. 删除
查看运行的deployment控制器类型的应用
```shell
kubectl get deployment.apps
```

通过命令删除deployment控制器类型的应用
```shell
kubectl delete deployment.apps nginx-app2
```

通过资源清单删除，在二中创建的应用，可以使用以下命令删除
```shell
kubectl delete -f create-deployment-nginx-app2.yaml
```