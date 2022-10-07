# 在k8s环境部署应用

## 一、概述

部署k8s通常包含以下几个步骤

- 1. 准备项目镜像，运行创建应用容器
- 2. 通常创建Deployment资源的方式运行应用
- 3. 创建应用的Service网络，用于关联Deployment
- 4. 创建对应的Ingress资源，用于调度7层流量

在下面，我们使用声明式的管理方式（通常指通过yaml配置文件来管理集群）来创建各个集群资源，完成应用部署。

## 二、准备资源配置文件

我们通过部署tomcat来演示一个应用在k8s部署的流程

### 2.1 创建deploymeny

创建`03-whoami.yml`文件，添加以下内容

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: whoami
  labels:
    app: whoami

spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: tomcat:8.5-jre10-slim
          ports:
            - name: web
              containerPort: 8080
```

### 2.2 创建service

创建`03-whoami-services.yml`文件，添加以下内容

```yml
apiVersion: v1
kind: Service
metadata:
  name: whoami

spec:
  ports:
    - name: web
      port: 80
      targetPort: web

  selector:
    app: whoami
```

### 2.3 创建ingress

创建`04-whoami-ingress.yml`，添加以下内容

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami-ingress
spec:
  rules:
  - host: tomcat.jkdev.cn
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: whoami
            port:
              name: we
```

## 三、验证

创建各个资源

```bash
kubectl apply -f 03-whoami-services.yml \
              -f 03-whoami.yml \
              -f 04-whoami-ingress.yml
```

执行以上命令之后，k8s将会拉取tomcat的镜像，并按我们指定的配置去启动服务。

启动完成之后，我们在自己的桌面操作系统的电脑上将`tomcat.jkdev.cn`域名解析到在前面通过keepalived创建的虚拟IP `192.168.9.190`，再使用浏览器访问域名，就顺利访问到了tomcat。
