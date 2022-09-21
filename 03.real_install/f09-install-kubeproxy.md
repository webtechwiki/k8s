# 安装kube-proxy

## 一、创建kubeconfig配置文件

在证书服务器上执行，如下命令

```bash
# 进入证书目录
cd /etc/kubernetes/pki/
# 创建集群信息
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.9.190:7443 --kubeconfig=kube-proxy.kubeconfig

# 创建用户信息
kubectl config set-credentials kube-proxy --client-certificate=proxy.pem --client-key=proxy-key.pem --embed-certs=true --kubeconfig=kube-proxy.kubeconfig

# 创建上下文
kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig

# 应用上下文
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

# 剪切到/etc/kubernetes/
mv kube-proxy.kubeconfig /etc/kubernetes/
```

创建完成后，同步到工作节点`192-debian`和`160-debian`。

## 二、创建kube-proxy配置文件

在工作节点创建`/etc/kubernetes/kube-proxy.yaml`，内容如下

```bash
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
clusterCIDR: 10.244.0.0/16 
kind: KubeProxyConfiguration
metricsBindAddress: 0.0.0.0:10249
mode: "ipvs"
```

## 三、启动服务

### 3.1 创建启动脚本

在两台主机执行以上脚本之后，我们创建`kube-proxy`的启动脚本文件`/opt/kubernetes/server/bin/kube-proxy.sh`

```shell
#!/bin/bash
./kube-proxy \
  --config=/etc/kubernetes/kube-proxy.yaml \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
```

添加可执行权限

```shell
chmod +x /opt/kubernetes/server/bin/kube-proxy.sh
```

### 3.2 创建服务配置

创建supervisor的配置文件`/etc/supervisor/conf.d/kube-proxy.conf`文件，添加以下内容

```ini
[program:kube-proxy-192]
directory=/opt/kubernetes/server/bin
command=/opt/kubernetes/server/bin/kube-proxy.sh
numprocs=1
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/supervisor/kube-proxy.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_event_enabled=false
```

更新supervisor

```shell
supervisorctl update
```

## 四、创建必要的权限

### 4.1 创建权限配置文件

在`199-debian`上创建`/etc/kubernetes/rbac.yaml`，写入如下内容

```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kubernetes-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kubernetes
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kubernetes-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
```

### 4.2 创建角色及授权

```bash

```bash
kubectl apply -f /etc/kubernetes/rbac.yaml
```

## 五、集群的验证

在两个节点都启动好kube-proxy服务之后，至此，集群的基本组件已经安装完成，下面我们来验证集群。在任意管理节点创建一个Pod类型的资源，添加`nginx-pod.yaml`文件，添加以下内容

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: nginx-pod
    image: nginx:latest
    imagePullPolicy: IfNotPresent
    ports:
    - name: nginxport
      containerPort: 80
```

执行资源创建命令

```shell
kubectl create -f nginx-ds.yaml
```

使用以下命令验证pod是否正常运行

```shell
kubectl get pod -o wide
```

如果返回如下内容，代表集群正常

```shell
root@199-debian:~# kubectl get pod -o wide
NAME   READY   STATUS    RESTARTS   AGE     IP          NODE       NOMINATED NODE   READINESS GATES
pod1   1/1     Running   0          2m52s   10.88.0.2   node-192   <none>           <none>
```

创建的pod运行在`node-192`这台主机上，在这台机使用`curl 10.88.0.2`命令能正常访问到nginx服务。但是如果我们在另一个节点`node-160`上执行`curl curl 10.88.0.2`会发现访问不到。原因是这两个节点上的容器在各自的虚拟网络内，我们将到后续的章节安装通过安装k8s网络插件的方式，继续解决不同节点的容器网络互相访问的问题！
