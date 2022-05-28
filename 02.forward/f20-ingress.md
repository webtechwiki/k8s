# Ingress


## 1. 概述

Ingress是k8s API的标准资源类型之一，也是一种核心资源，它其实就是一组基于域名和URL路径，把用户请求转发至指定Service资源的规则。可以将集群外部的流量请求转发至集群内部，从而实现“服务暴露”。Ingress控制器是能够为Ingress资源监听某套接字，然后根据Ingress规则匹配机制路由调度流量的一个组件。我们可以把Ingress理解为一个实现了 nginx 功能go语言程序。


常用的Ingress控制器的实现软件：

- Ingress-nginx
- HAProxy
- Trasefik

......



本文我们使用 Trasefik 作为Ingress控制器来实现服务暴露



## 2. 安装Trasefik（Ingress控制器）


### 2.1. 准备

我们先在集群节点下载 trasefik 的镜像

```shell
docker pull trasefik:v1.7.2-alpine
```


再登录`kb200`准备trasefik资源文件，创建资源目录

```shell
mkdir -p /data/k8s-yaml/traefik
```


### 2.2. 授权清单

创建`rbac.yaml`授权资源，写入如下内容

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
```


### 2.3. delepoly资源清单

创建`ds.yaml`文件，写入如下内容：

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: traefik-ingress
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress
        name: traefik-ingress
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik:v1.7.2
        name: traefik-ingress
        ports:
        - name: controller
          containerPort: 80
          hostPort: 81
        - name: admin-web
          containerPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
        - --insecureskipverify=true
        - --kubernetes.endpoint=https://192.168.14.10:7443
        - --accesslog
        - --accesslog.filepath=/var/log/traefik_access.log
        - --traefiklog
        - --traefiklog.filepath=/var/log/traefik.log
        - --metrics.prometheus
```


### 2.4. service清单


创建`svc.yaml`，写入如下内容


```yaml
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress
  ports:
    - protocol: TCP
      port: 80
      name: controller
    - protocol: TCP
      port: 8080
      name: admin-web
```


### 2.5. ingress清单

创建文件`ingress.yaml`，写入以下内容

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: traefik.od.com
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-ingress-service
          servicePort: 8080
```

### 2.6. 创建资源

在任意集群节点上执行以下命令

```shell
kubectl create -f http://k8s-yaml.od.com/traefik/rbac.yaml
kubectl create -f http://k8s-yaml.od.com/traefik/ds.yaml
kubectl create -f http://k8s-yaml.od.com/traefik/svc.yaml
kubectl create -f http://k8s-yaml.od.com/traefik/ingress.yaml
```

创建好资源之后，使用如下命令查看是否正常

```shell
kubectl get pods -n kube-system -o wide
```


如果看到traefik处于running状态中，则代表服务正常。我们继续使用以下命令查看81端口的服务

```shell
netstat -luntp | grep 81
```


如果返回类似如下内容，代表traefik正常服务

```shell
[root@kb21 vagrant]# netstat -luntp | grep 81
tcp6       0      0 :::81                   :::*                    LISTEN      16694/docker-proxy
```


## 3. 在往外入口做nginx方向代理

### 3.1. 反向代理


我们在`kb11`和`kb12`上做反向代理，将泛域名的解析都转发到trafik上，创建文件`/etc/nginx/conf.d/od.com.conf`文件，写入如下内容：

```shell
upstream default_backend_traefik {
    server 192.168.14.21:81    max_fails=3 fail_timeout=10s;
    server 192.168.14.22:81    max_fails=3 fail_timeout=10s;
}
server {
    server_name *.od.com;
  
    location / {
        proxy_pass http://default_backend_traefik;
        proxy_set_header Host       $http_host;
        proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    }
}
```

重启nginx服务
```shell
nginx -t
nginx -s reload
```


### 3.2. 测试服务

需要将traefik 服务的解析记录添加的DNS解析中,注意是绑定到VIP上，编写`/var/named/od.com.zone`，添加`traefik`子域的解析，序列号往前进位，如下


```shell
traefik            A    192.168.14.10
```


重启named服务

```shell
systemctl restart named
```

验证域名解析

```shell
dig -t A traefik.od.com +short
```


如果正常有内容返回，代表解析正常

### 3.3. 集群验证

在集群外,访问`http://traefik.od.com`,如果能正常显示web页面.说明我们已经暴露服务成功




















