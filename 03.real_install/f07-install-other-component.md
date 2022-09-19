# 安装controller-manager和kube-scheduler

## 一、创建kubectl链接

我们使用`kubectl`作为k8s的管理工具，创建一个软链接，如下

```shell
ln -s /opt/kubernetes/server/bin/kubectl /usr/local/bin/kubectl
```

## 二、安装controller-manager

初始化配置

```bash
# 设置一个集群项
kubectl config set-cluster kubernetes \
     --certificate-authority=/opt/certs/ca.pem \
     --embed-certs=true \
     --server=https://192.168.9.190:7443 \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

# 设置一个环境项，一个上下文
kubectl config set-context system:kube-controller-manager@kubernetes \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

# 设置一个用户项
kubectl config set-credentials system:kube-controller-manager \
     --client-certificate=/opt/certs/controller-manager/controller-manager.pem \
     --client-key=/opt/certs/controller-manager/controller-manager-key.pem \
     --embed-certs=true \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

# 设置默认环境
kubectl config use-context system:kube-controller-manager@kubernetes \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
```

`/etc/kubernetes/controller-manager.kubeconfig`配置文件只需生成一次，再传到其他主机即可。

创建文件`/opt/kubernetes/server/bin/kube-controller-manager.sh`，添加以下内容

```shell
#!/bin/bash
./kube-controller-manager \
    --v=2 \
    --logtostderr=true \
    --bind-address=127.0.0.1 \
    --root-ca-file=/opt/certs/ca.pem \
    --cluster-signing-cert-file=/opt/certs/ca.pem \
    --cluster-signing-key-file=/opt/certs/ca-key.pem \
    --service-account-private-key-file=/opt/certs/sa.key \
    --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig \
    --leader-elect=true \
    --use-service-account-credentials=true \
    --node-monitor-grace-period=40s \
    --node-monitor-period=5s \
    --pod-eviction-timeout=2m0s \
    --controllers=*,bootstrapsigner,tokencleaner \
    --allocate-node-cidrs=true \
    --feature-gates=IPv6DualStack=true \
    --service-cluster-ip-range=10.96.0.0/12 \
    --cluster-cidr=172.16.0.0/12,fc00::/48 \
    --node-cidr-mask-size-ipv4=24 \
    --node-cidr-mask-size-ipv6=64 \
    --requestheader-client-ca-file=/opt/certs/ca.pem
```

添加执行权限与创建日志目录

```shell
# 添加可执行权限
chmod +x kube-controller-manager.sh
# 创建日志目录
mkdir -p /data/logs/kubernetes/kube-controller-manager
```

创建supervisor脚本启动管理文件`/etc/supervisor/conf.d/kube-controller-manager.conf`，添加以下内容

```ini
[program:kube-controller-manager-199]
directory=/opt/kubernetes/server/bin
command=/opt/kubernetes/server/bin/kube-controller-manager.sh
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
stdout_logfile=/data/logs/kubernetes/kube-controller-manager/controller.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_event_enabled=false
```

更新supervisor

```shell
supervisorctl update
```

## 三、部署scheduler

创建配置

```bash
# 设置一个集群项
kubectl config set-cluster kubernetes \
     --certificate-authority=/opt/certs/ca.pem \
     --embed-certs=true \
     --server=https://192.168.9.190:7443 \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

# 设置一个环境项，一个上下文
kubectl config set-credentials system:kube-scheduler \
     --client-certificate=/opt/certs/scheduler/scheduler.pem \
     --client-key=/opt/certs/scheduler/scheduler-key.pem \
     --embed-certs=true \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

# 设置一个用户项
kubectl config set-context system:kube-scheduler@kubernetes \
     --cluster=kubernetes \
     --user=system:kube-scheduler \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

# 设置默认环境
kubectl config use-context system:kube-scheduler@kubernetes \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
```

`/etc/kubernetes/scheduler.kubeconfig`配置文件也只需生成一次，再传到其他主机即可。

创建scheluder启动脚本文件`/opt/kubernetes/server/bin/kube-scheduler.sh`文件，添加以下内容

```shell
#!/bin/bash
./kube-scheduler \
    --v=2 \
    --logtostderr=true \
    --bind-address=127.0.0.1 \
    --leader-elect=true \
    --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
```

添加脚本执行权限与创建日志目录

```shell
# 添加脚本的可执行权限
chmod +x kube-scheduler.sh
# 创建日志目录
mkdir -p /data/logs/kubernetes/kube-scheduler
```

创建进程管理配置文件`/etc/supervisor/conf.d/kube-scheduler.conf`文件，添加以下内容

```ini
[program:kube-scheduler-199]
directory=/opt/kubernetes/server/bin
command=/opt/kubernetes/server/bin/kube-scheduler.sh
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
stdout_logfile=/data/logs/kubernetes/kube-scheduler/scheduler.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_event_enabled=false
```

更新supervisor

```shell
supervisorctl update
```

## 四、集群验证

先创建一个管理员的配置项

```bash
# 设置一个集群项
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/certs/ca.pem \
  --embed-certs=true \
  --server=https://192.168.9.190:7443 \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig

# 设置一个环境项，一个上下文
kubectl config set-credentials kubernetes-admin \
  --client-certificate=/opt/certs/admin/admin.pem \
  --client-key=/opt/certs/admin/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig

# 设置一个用户项
kubectl config set-context kubernetes-admin@kubernetes \
  --cluster=kubernetes \
  --user=kubernetes-admin \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig

# 设置默认环境
kubectl config use-context kubernetes-admin@kubernetes  --kubeconfig=/etc/kubernetes/admin.kubeconfig
```

拷贝配置文件到用户目录

```bash
mkdir -p ~/.kube
cp /etc/kubernetes/admin.kubeconfig ~/.kube/config
```

再使用`kubectl get cs`检查集群状态，如下返回如下内容，则代表正常服务

```shell
[root@kb22 bin]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
[root@kb22 bin]#
```
