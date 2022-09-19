# 安装etcd

我们将使用`199-debian`、`192-debian`、`160-debian`这三台主机搭建ectd集群。

## 一、创建运行用户

我们将在`199-debian`、`192-debian`、`160-debian`这三台主机上安装etcd，组成一个ectd集群。在撰写这篇文档的时候吗，etcd最新的版本是`3.5.4`，我们选择一个较新的版本`v3.5.4`，具体操作如下：

先创建一个ectd用户，禁止该用户远程登录，并且没有家目录，如下命令

```shell
useradd -s /sbin/nologin -M etcd
```

## 二、安装

```shell
# 下载对应版本
wget https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz
# 解压
tar -zxvf etcd-v3.5.4-linux-amd64.tar.gz
# 将etc移到/opt目录，并修改etcd目录名
mv etcd-v3.5.4-linux-amd64/ /opt/etcd-v3.5.4
# 创建etcd软链接
ln -s /opt/etcd-v3.5.4 /opt/etcd
# 创建etcd证书目录和数据目录
mkdir -p /opt/etcd/certs /data/etcd/etcd-server /data/logs/etcd-server
# 从证书服务器下载证书
cd /opt/
scp debian@199-debian.k8s.com:/opt/certs/ca.pem ./
```

给所有etcd用户需要读写的目录赋予权限

```shell
# 将目录权限改为etcd用户
chown -R etcd.etcd /opt/certs/etcd/ /data/etcd /data/logs/etcd-server
```

## 三、编写etcd启动脚本

我们先在etcd目录编写启动脚本`/opt/etcd/startup.sh`，如下

```shell
#!/bin/bash
./etcd --name etcd-server-199 \
  --data-dir /data/etcd/etcd-server \
  --listen-peer-urls https://192.168.9.199:2380 \
  --listen-client-urls https://192.168.9.199:2379,http://127.0.0.1:2379 \
  --quota-backend-bytes 8000000000 \
  --initial-advertise-peer-urls https://192.168.9.199:2380 \
  --advertise-client-urls https://192.168.9.199:2379,http://127.0.0.1:2379 \
  --initial-cluster etcd-server-199=https://192.168.9.199:2380,etcd-server-192=https://192.168.9.192:2380,etcd-server-160=https://192.168.9.160:2380 \
  --initial-cluster-token="etcd-token" \
  --initial-cluster-state=new \
  --cert-file ./certs/etcd-peer.pem \
  --key-file ./certs/etcd-peer-key.pem \
  --client-cert-auth \
  --trusted-ca-file ./certs/ca.pem \
  --peer-cert-file ./certs/etcd-peer.pem \
  --peer-key-file ./certs/etcd-peer-key.pem \
  --peer-client-cert-auth \
  --peer-trusted-ca-file ./certs/ca.pem
```

给启动脚本添加权限

```shell
chmod +x startup.sh
```

参数说明：

`--name`：当前etcd的唯一名称，要保证和其他节点不冲突，环境变量: ETCD_DATA_DIR
`--data-dir`：指定etcd存储数据的存储位置，环境变量: ETCD_NAME
`--listen-peer-urls`：端对端的通信url，包含主机地址和端口号，指定当前etcd和其他节点etcd通信时的服务地址和端口。假如etcd集群中包含三个etcd服务，那么三个etcd节点构成了一个高可用集群，etcd集群会自动选举一个节点作为主节点，另外两个节点作为从节点。所有的写入操作将往主节点写，所有的读操作在从节点上读，这是etcd读写的过程。上述设置的2380端口，用来监听其他etcd节点发送过来的数据，环境变量: ETCD_LISTEN_PEER_URLS
`--listen-client-urls`：指定当前etcd接收客户端指令的地址和端口，在这里的客户端我们指的是k8s集群的master节点，环境变量: ETCD_LISTEN_CLIENT_URLS
`--quota-backend-bytes`：后端的配额
`--initial-advertise-peer-urls`：指定etcd广播端口，当前etcd会将数据同步到其他节点，通过2380端口发送，环境变量: ETCD_INITIAL_ADVERTISE_PEER_URLS
`--advertise-client-urls`：给客户端通告的端口，环境变量: ETCD_ADVERTISE_CLIENT_URLS
`--initial-cluster`：定义etcd集群中所有节点的名称和IP，以及通信端口，环境变量: ETCD_INITIAL_CLUSTER
`--initial-cluster-token`：定义etcd中的token，所有节点的token必须保持一致，环境变量: ETCD_INITIAL_CLUSTER_TOKEN
`--initial-cluster-state`：定义etcd集群的状态，new代表新建集群，existing代表加入现有集群，环境变量: ETCD_INITIAL_CLUSTER_STATE
`--cert-file`：客户端服务器 TLS 证书文件的路径，环境变量: ETCD_PEER_CERT_FILE
`--key-file`：客户端服务器 TLS key 文件的路径，环境变量: ETCD_PEER_KEY_FILE
`--client-cert-auth`：开启客户端证书认证，环境变量: ETCD_PEER_CLIENT_CERT_AUTH
`--trusted-ca-file`：客户端服务器 TLS 信任证书文件的路径，环境变量: ETCD_PEER_TRUSTED_CA_FILE
`--peer-cert-file`：peer server TLS 证书文件的路径，环境变量: ETCD_PEER_CERT_FILE
`--peer-key-file`：peer server TLS key 文件的路径，环境变量: ETCD_PEER_KEY_FILE
`--peer-client-cert-auth`：开启 peer client 证书验证，环境变量: ETCD_PEER_CLIENT_CERT_AUTH
`--peer-trusted-ca-file`：peer server TLS 信任证书文件路径，环境变量: ETCD_PEER_TRUSTED_CA_FILE

其他参数：ssl证书相关证书

## 四、使用supervisord来启动etcd

现在我们要安装supervisord，用于管理etcd服务

```shell
# 安装supervisor
apt install supervisor -y
# 启动supervisor
systemctl start supervisor
# 让superivisor开机自启
systemctl enable supervisor
```

添加etcd的supervisor进程维护脚本`/etc/supervisor/conf.d/etcd-server.conf`，添加以下内容

```ini
[program:etcd-server-199]
directory=/opt/etcd
command=/opt/etcd/startup.sh
numprocs=1
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=etcd
redirect_stderr=true
stdout_logfile=/data/logs/etcd-server/etcd.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_event_enabled=false
```

> 注意：在不同的主机上使用不同的服务名称，这样好辨别，如199-debian使用`etcd-server-199`，如192-debian使用`etcd-server-192`

相关参数：

`program`: 程序名称
`directory`: 脚本目录
`command`: 启动的命令
`numprocs`: 启动的进程数
`autostart`: 是否开启自动启动
`autorestart`: 是否自动重启
`startsecs`: 启动之后多少时间后判定为已起来
`startretries`: 重启次数
`exitcodes`: 退出的code
`stopsignal`: 停止信号
`stopwaitsecs`: 停止等待的时间
`redirect_stderr`: 是否重定向标准输出
`stdout_logfile`: 进程标准输出内容写入文件
`stdout_logfile_maxbytes`: stdout_logfile文件做log滚动时，单个stdout_logfile文件的最大字节数，默认50M，设置为0则认为不做log滚动方式
`stdout_logfile_backups`: stdout_logfile备份文件个数，默认为10
`stdout_capture_maxbytes`: 当进程处于stdout capture mode模式的时候，写入capture FIFO的最大字节数限制，默认为0，此时认为stdout capture mode模式关闭
`stdout_event_enabled`: 如果设置为true，在进程写入标准文件是会发起PROCESS_LOG_STDOUT

更新supervisod配置文件

```shell
supervisorctl update
```

通过`supervisorctl status`查询supervisord状态，看到如下内容，代表supervisor正常运行

```shell
root@debian:/opt/etcd# supervisorctl status
etcd-server-199                  RUNNING   pid 85297, uptime 0:04:38
```

此时，我们再使用`netstat -luntp | grep etcd`查看网络服务端口，看到如下信息代表etcd已经正常启动

```shell
root@debian:/opt/etcd# netstat -luntp | grep etcd
tcp        0      0 192.168.9.199:2379      0.0.0.0:*               LISTEN      85298/./etcd        
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      85298/./etcd        
tcp        0      0 192.168.9.199:2380      0.0.0.0:*               LISTEN      85298/./etcd
```

## 五、集群验证

为了方便直接调用`etcdctl`命令，我们还可以创建其软连接

```bash
ln -s /opt/etcd/etcdctl /usr/local/bin/etcdctl
```

我们在任意节点使用etcdctl命令检查集群状态，需要注意的是，要确切指定证书的位置

```shell
etcdctl --cacert="/opt/certs/ca.pem" --cert="/opt/certs/etcd/etcd-peer.pem" --key="/opt/certs/etcd/etcd-peer-key.pem" --endpoints="https://192.168.9.199:2379,https://192.168.9.192:2379,https://192.168.9.160:2379" endpoint status --write-out=table
```

如果看到如下输出，代表 ectd 集群搭建成功

```bash
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.9.199:2379 | c8815bb4b21730b3 |   3.5.4 |  311 kB |      true |      false |         3 |      37628 |              37628 |        |
| https://192.168.9.192:2379 | f30299e8a0b43b4d |   3.5.4 |  311 kB |     false |      false |         3 |      37628 |              37628 |        |
| https://192.168.9.160:2379 | 61c90f737ccf2682 |   3.5.4 |  311 kB |     false |      false |         3 |      37628 |              37628 |        |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

为了验证etcd集群是否正常工作，我们还可以现在`199-debian`设置一个值，如下

```bash
etcdctl set name lixiaoming
```

再通过`192-debian`和`160-debian`去读取值，如果正常取到，代表etcd集群正常工作，如下命令

```bash
etcdctl get name
```

如果需要了解`etcdctl`这个指令的更多用法，使用`--help`参数即可查看。