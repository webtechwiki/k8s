# 安装主控制节点控制器和调度器

## 一、安装controller-manager

创建文件`/opt/kubernetes/server/bin/kube-controller-manager.sh`，添加以下内容

```shell
#!/bin/sh
./kube-controller-manager \
  --cluster-cidr 172.7.0.0/16 \
  --leader-elect true \
  --log-dir /data/logs/kubernetes/kube-controller-manager \
  --master http://127.0.0.1:8080 \
  --service-account-private-key-file ./certs/ca-key.pem \
  --service-cluster-ip-range 192.168.0.0/16 \
  --root-ca-file ./certs/ca.pem \
  --v 2
```

添加执行权限与创建日志目录

```shell
# 添加可执行权限
chmod +x kube-controller-manager.sh
# 创建日志目录
mkdir -p /data/logs/kubernetes/kube-controller-manager
```

创建supervisor脚本启动管理文件`/etc/supervisord.d/kube-controller-manager.ini`，添加以下内容

```shell
[program:kube-controller-manager-21]
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

创建scheluder启动脚本文件`/opt/kubernetes/server/bin/kube-scheduler.sh`文件，添加以下内容

```shell
#!/bin/sh
./kube-scheduler \
  --leader-elect \
  --log-dir /data/logs/kubernetes/kube-scheduler \
  --master http://127.0.0.1:8080 \
  --v 2
```

添加脚本执行权限与创建日志目录

```shell
# 添加脚本的可执行权限
chmod +x kube-scheduler.sh
# 创建日志目录
mkdir -p /data/logs/kubernetes/kube-scheduler
```

创建进程管理配置文件`/etc/supervisord.d/kube-scheduler.ini`文件，添加以下内容

```ini
[program:kube-scheduler-21]
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
ln -s /opt/kubernetes/server/bin/kubectl /usr/bin/kubectl
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
