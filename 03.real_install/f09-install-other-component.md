# 安装主控制节点控制器和调度器

## 一、安装controller-manager

```bash
# 设置一个集群项
kubectl config set-cluster kubernetes \
     --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
     --embed-certs=true \
     --server=https://192.168.9.199:6443 \
     --kubeconfig=/opt/kubernetes/server/bin/conf/controller-manager.kubeconfig

# 设置一个环境项，一个上下文

kubectl config set-context system:kube-controller-manager@kubernetes \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=/opt/kubernetes/server/bin/conf/controller-manager.kubeconfig

# 设置一个用户项
kubectl config set-credentials system:kube-controller-manager \
     --client-certificate=/opt/kubernetes/server/bin/certs/client.pem \
     --client-key=/opt/kubernetes/server/bin/certs/client-key.pem \
     --embed-certs=true \
     --kubeconfig=/opt/kubernetes/server/bin/conf/controller-manager.kubeconfig

# 设置默认环境
kubectl config use-context system:kube-controller-manager@kubernetes \
     --kubeconfig=/opt/kubernetes/server/bin/conf/controller-manager.kubeconfig
```

创建文件`/opt/kubernetes/server/bin/kube-controller-manager.sh`，添加以下内容

```shell
#!/bin/sh
./kube-controller-manager \
  --v=2 \
  --logtostderr=true \
  --root-ca-file=./certs/ca.pem \
  --cluster-signing-cert-file=./certs/ca.pem \
  --cluster-signing-key-file=./certs/ca-key.pem \
  --service-account-private-key-file=/opt/kubernetes/server/bin/certs/sa.key \
  --kubeconfig=/opt/kubernetes/server/bin/conf/controller-manager.kubeconfig \
  --leader-elect=true \
  --use-service-account-credentials=true \
  --node-monitor-grace-period=40s \
  --node-monitor-period=5s \
  --pod-eviction-timeout=2m0s \
  --controllers=*,bootstrapsigner,tokencleaner \
  --allocate-node-cidrs=true \
  --cluster-cidr=172.16.0.0/12 \
  --requestheader-client-ca-file=/opt/kubernetes/server/bin/certs/ca.pem \
  --node-cidr-mask-size=24
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

## 二、部署scheduler

创建配置

```bash
kubectl config set-cluster kubernetes \
     --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
     --embed-certs=true \
     --server=https://192.168.9.199:6443 \
     --kubeconfig=/opt/kubernetes/server/bin/conf/scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
     --client-certificate=/opt/kubernetes/server/bin/certs/client.pem \
     --client-key=/opt/kubernetes/server/bin/certs/client-key.pem \
     --embed-certs=true \
     --kubeconfig=/opt/kubernetes/server/bin/conf/scheduler.kubeconfig

kubectl config set-context system:kube-scheduler@kubernetes \
     --cluster=kubernetes \
     --user=system:kube-scheduler \
     --kubeconfig=/opt/kubernetes/server/bin/conf/scheduler.kubeconfig

kubectl config use-context system:kube-scheduler@kubernetes \
     --kubeconfig=/opt/kubernetes/server/bin/conf/scheduler.kubeconfig
```

创建scheluder启动脚本文件`/opt/kubernetes/server/bin/kube-scheduler.sh`文件，添加以下内容

```shell
#!/bin/bash
./kube-scheduler \
  --logtostderr=true \
  --leader-elect \
  --kubeconfig=/opt/kubernetes/server/bin/conf/scheduler.kubeconfig \
  --v 2
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

## 三、集群验证

安装完成基本的组件之后，我们接下来使用`kubectl`作为k8s的管理工具，创建一个软链接，如下

```shell
ln -s /opt/kubernetes/server/bin/kubectl /usr/local/bin/kubectl
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
