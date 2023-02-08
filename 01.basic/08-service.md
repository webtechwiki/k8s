# Service

## 一、 概述

我们可以通过Controller创建应用，可是当我们访问应用时，发现一个问题，Pod的状态不是认为控制的，Pod IP是在创建的时候分配的。假设Pod被误删除，被Controller重新拉起一个新的Pod时，我们发现Pod IP是变化的。如果访问必须更换IP地址，这样对于大量Pod运行应用来说，我们对Pod完全无法控制，因此在k8s集群中我们引入另一个新的概念：Service

本期目标

- 了解Service的作用
- 了解Serivce的类型
- Serivce参数
- Service创建方法
- Service删除方法

## 二、相关概念

### 2.1 Service概念

- Serivice不是一个实体服务，Service是一条iptables或ipvs的转发规则
- 通过Service为Pod客户端提供访问Pod方法（即客户端访问Pod入口），Service通过Pod标签与Pod进行关联

### 2.2 Service类型

`ClusterIP`: 默认，分配一个集群内部可以访问的虚拟IP

`NodePort`: 在每个Node上分配一个端口作为外部访问入口

`LoadBalancer`: 工作在待定的Cloud Provider上，例如 Google Cloud，AWS，OpenStack

`ExternalName`: 表示把集群外部的服务引入到集群内部中来，即实现集群内部Pod和集群外部的服务进行通信

### 2.3 Service参数

常用端口参数

- port: 访问Service使用的端口
- targetPort: Pod中容器端口
- NodePort: 通过Node实现外网用户访问k8s集群内部Service（30000-32767）

## 三、Service的创建

### 3.1 通过命令创建Service

先创建deployment应用，然后创建Service与Deployment应用关联，查看deployment应用：`kubectl get deployment.apps`，返回以下内容

```bash
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app2   1/1     1            1           7m43s
```

创建Service

```bash
kubectl expose deployment.apps nginx-app2 --type=ClusterIP --target-port=80 --port=80
```

查看创建的Service

```bash
kubectl get service
# 或
kubectl get svc
```

返回以下内容

```bash
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   24h
nginx-app2   ClusterIP   10.104.130.50   <none>        80/TCP    6m58s
```

查看Service与Pod的端点关系，使用以下命令

```bash
kubectl get endpoints
```

返回以下内容

```bash
NAME         ENDPOINTS           AGE
kubernetes   192.168.64.4:6443   24h
nginx-app2   10.244.1.5:80       7m10s
```

### 3.2 通过资源清单创建Service

我们先在资源清单里定义Deployment应用，再定义Service，创建`create-deployment-nginx-app-service.yaml`资源清单文件，内容如下

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      apps: nginx
  template:
    metadata:
      labels:
        apps: nginx
    spec:
      containers:
      - name: nginxapp
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-app2-svc
spec:
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 80
  selector:
   apps: nginx
```

通过以下命令来创建

```bash
kubectl apply -f create-deployment-nginx-app-service.yaml
```

### 3.3 通过资源清单创建NodePort类型的Service

创建`create-deployment-nginx-app3-service.yaml`，定义内容如下

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app3
spec:
  replicas: 2
  selector:
    matchLabels:
      apps: nginx-app3
  template:
    metadata:
      labels:
        apps: nginx-app3
    spec:
      containers:
      - name: nginxapp3
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-app3-svc
spec:
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30001
  selector:
   apps: nginx-app3
```

通过以下命令来创建

```bash
kubectl apply -f create-deployment-nginx-app3-service.yaml
```

### 3.4 删除Service

通过命令行删除

```bash
kubectl delete service nginx-app2
```

通过资源清单文件创建的可以通过资源清单文件删除，如下

```bash
kubectl delete -f create-deployment-nginx-app-service.yaml
```
