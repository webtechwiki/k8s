# flannel网络介绍


k8s设计了网络模型，但确将它的实现交给了网络插件，CNI网络插件主要的功能就是实现POD资源能够跨宿主机进行通信，插件的CNI网络插件如下


- Flannel
- Calico
- Canal
- Contiv
- OpenContrial
- NSX-T
- Kube-router

在本文中，我们将介绍使用Flannel插件来实现容器间的跨主机访问


## 1. 下载与安装flannel

我们需要在`kb21`和`kb22`这两台主机安装flannel。

Flannel的仓库地址是：[https://github.com/flannel-io/flannel/](https://github.com/flannel-io/flannel/)

```shell-script
# 下载flannel
wget https://github.com/flannel-io/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz
# 创建flannel安装目录
mkdir /opt/flannel-v0.11.0
# 安装
tar -zxvf flannel-v0.11.0-linux-amd64.tar.gz -C /opt/flannel-v0.11.0/
# 创建软连接
ln -s /opt/flannel-v0.11.0/ /opt/flannel
```


## 2. 配置启动参数


进入到`/opt/flannel`，创建子网配置文件`subnet.env`，添加如下内容


```shell
FLANNEL_NETWORK=172.7.0.0/16
FLANNEL_SUBNET=172.7.21.1/24
FLANNEL_MTU=1500
FLANNEL_IPMASQ=false
```

如果是kb22这台主机，`FLANNEL_SUBNET`这个值改为`172.7.22.1/24`


准备SSL证书

```shell
# 创建证书目录
mkdir certs
# 进入etcd证书目录
cd certs
# 从证书服务器下载证书
scp root@kb200.host.com:/opt/certs/ca.pem ./
scp root@kb200.host.com:/opt/certs/kubernetes/client.pem ./
scp root@kb200.host.com:/opt/certs/kubernetes/client-key.pem ./
```




创建启动脚本`/opt/flannel/flanneld.sh`，添加一下内容

```shell
#!/bin/bash
./flanneld \
  --public-ip=192.168.14.21 \
  --etcd-endpoints=https://192.168.14.12:2379,https://192.168.14.21:2379,https://192.168.14.22:2379 \
  --etcd-cafile=./certs/ca.pem \
  --etcd-certfile=./certs/client.pem \
  --etcd-keyfile=./certs/client-key.pem \
  --iface=eth1 \
  --subnet-file=./subnet.env \
  --healthz-port=2401
```

给启动脚本添加执行权限

```shell
chmod +x flanneld.sh
```

创建日志目录

```shell
mkdir -p /data/logs/flanneld
```


创建supervisor配置文件`/etc/supervisord.d/flanneld.ini`

```shell
[program:flannel-21]
directory=/opt/flannel
command=/opt/flannel/flanneld.sh
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
stdout_logfile=/data/logs/flanneld/flanneld.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_event_enabled=false
```


## 3. 操作etcd，增加host-gw

在更新supervisord之前，我们需要先使用etcd增加host-gw，如下命令

```shell
./etcdctl set /coreos.com/network/config '{"Network":"172.7.0.0/16","Backend":{"Type":"host-gw"}}'
```

这条命令表示设置flannel的网络模型为`host-gw`模型，flannel将会读取到键名`/coreos.com/network/config`对应的值。如果我们要确保etcd已经设置了对应的键值对，可以使用下名这个命令查看

```shell
./etcdctl get /coreos.com/network/config
```


## 4. 启动服务

```shell
# 更新supervisor
supervisorctl update
```





















