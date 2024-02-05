# 安装traefik-ingress

## 一、创建角色和账号

在运行traefik之前，我们需要为traefik特定的角色和账号，我们在`199-debian`操作即可。

### 1.1 创建角色

```bash
cat > /etc/kubernetes/traefik-role.yml <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-role

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
      - networking.k8s.io
    resources:
      - ingresses
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses/status
    verbs:
      - update
EOF
```

### 1.2 创建账号

```bash
cat > /etc/kubernetes/traefik-account.yml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-account
EOF
```

### 1.3 创建角色和账号绑定

```bash
cat > /etc/kubernetes/traefik-role-binding.yml <<EOF
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-role-binding

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-role
subjects:
  - kind: ServiceAccount
    name: traefik-account
    namespace: default
EOF
```

## 二、启动服务

### 2.1 启动traefik

```bash
cat > /etc/kubernetes/traefik.yml <<EOF
kind: Deployment
apiVersion: apps/v1
metadata:
  name: traefik-deployment
  labels:
    app: traefik

spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-account
      containers:
        - name: traefik
          image: traefik:v2.8
          args:
            - --api.insecure
            - --providers.kubernetesingress
          ports:
            - name: web
              containerPort: 80
            - name: dashboard
              containerPort: 8080
EOF
```

基于traefik镜像启动的pod将创建运行两个端口服务，`80`对应ingress本身核心服务，我们后续可以将流量都转发到ingress的`80`端口，让ingress做流量调度。`8080`是traefik的控制面板后台服务。

### 2.2 创建traefik的service

```bash
cat /etc/kubernetes/traefik-services.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: traefik-dashboard-service

spec:
  type: NodePort
  ports:
    - port: 8080
      name: dashboard
      protocol: TCP
      nodePort: 34808
      name: http
  selector:
    app: traefik
---
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-service

spec:
  type: NodePort
  ports:
    - port: 80
      name: web
      protocol: TCP
      nodePort: 34807
      name: http
  selector:
    app: traefik
EOF
```

我们创建两个`nodePort`类型的service，目的是为了在每个工作节点上提供一个服务入口。后续再将请求负载到各个节点上。

### 2.3 配置nginx

在`http`节点添加以下配置

```ini
upstream traefik_dashboard {
    server 192.168.9.192:34808    max_fails=3 fail_timeout=10s;
    server 192.168.9.160:34808    max_fails=3 fail_timeout=10s;
}

server {
    server_name traefik.k8s.com;

    location / {
        proxy_pass http://traefik_dashboard;
        proxy_set_header Host       $http_host;
        proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    }
}


upstream default_backend_traefik {
    server 192.168.9.192:34807    max_fails=3 fail_timeout=10s;
    server 192.168.9.160:34807    max_fails=3 fail_timeout=10s;
}

server {
    server_name *.jkdev.cn;

    location / {
        proxy_pass http://default_backend_traefik;
        proxy_set_header Host       $http_host;
        proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    }
}
```

在以上配置中，我们把`traefik.k8s.com`的请求转发到traefik的控制面板。同时，我们把`*.jkdev.cn`的所有流量转发到集群的traefik服务，由traefik来调度，后续发布应该我们只需要配置好ingress资源即可。

至此，集群的核心组件和核心插件已经全部安装完毕，集群已可以正常工作。我们将在最后一节发布一个k8应用，敬请期待。
