# 安装主控制节点控制器和调度器


## 1. 安装controller-manager


创建文件`/opt/kubernetes/server/bin/kube-controller-manager-startup.sh`，添加以下内容

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



> 02:52

添加执行权限与创建日志目录


```shell
# 添加可执行权限
chmod +x kube-controller-manager-startup.sh
# 创建日志目录
mkdir -p /data/logs/kubernetes/kube-controller-manager
```


创建supervisor脚本启动管理文件`/etc/supervisord.d/kube-controller-manager.ini`，添加以下内容

```shell
[program:kube-controller-manager-21]
directory=/opt/kubernetes/server/bin
command=/opt/kubernetes/server/bin/kube-controller-manager-startup.sh
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

跟新supervisor

```shell
supervisorctl update
```



> 02:52




