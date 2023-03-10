# 安装etcd

我们将使用`kb12`、`kb21`、`kb22`这三台主机作为控制节点，首先在控制节点上搭建ectd集群。

## 1. 准备ssl证书

登录到`kb200`证书签发服务器。在本系列的第三篇中，我们已经进行了根证书的签发。现在我们需要基于证书服务器的根证书为etcd服务创建ssl证书。以下 是具体的签发过程


**1.1. 定义证书信息**

创建证书保存目录

```shell
# 在根证书目录下创建专门存放etcd证书的目录
mkdir -p /opt/certs/etcd
# 进入etcd证书目录
cd /opt/certs/etcd
```

我们先在`/opt/certs/`创建`ca-config.json`文件，定义证书的基本信息，写入如下内容

```json
{
  "signing": {
    "default": {
      "expiry": "175200h"
    },
    "profiles": {
      "server": {
         "expiry": "175200h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth"
        ]
      },
      "client": {
         "expiry": "175200h",
         "usages": [
            "signing",
            "key encipherment",
            "client auth"
        ]
      },
      "peer": {
         "expiry": "175200h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
```

上面的文件定义了证书的基本信息，其中`profiles`里可以包含多个档案对象。我们定了三个节点，关键不同的配置在`usages`节点，如下：

`server`：服务器去连接客户端时，需要证书
`client`：客户端去连接服务器时，需要证书
`peer`：是服务器与客户端交换数据，都需要证书，接下来我们将基于`peer`属性创建证书。


接下来在`/opt/certs/etcd`目录创建证书请求文件`etcd-peer-csr.json`，写入以下内容

```json
{
    "CN": "etcd",
    "hosts": [
        "192.168.14.11",
        "192.168.14.12",
        "192.168.14.21",
        "192.168.14.22"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [{
        "C": "CN",
        "ST": "Guangdong",
        "L": "Guangzhou",
        "O": "od",
        "OU": "ops"
    }]
}
```

相关参数：

- `CN`：证书名称
- `hosts`：颁发的主机
- `key`：定义证书类型，algo为加密类型，size为加密长度
- `names`：证书的基本信息



**1.2. 向签证机构（CA）申请证书**

完成以上步骤，使用以下命令给客户端申请证书

```shell
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=peer etcd-peer-csr.json | cfssljson -bare etcd-peer
```

## 2. 安装ectd


我们将在`kb12`、`kb21`、`kb22`这三台主机上安装etcd，组成一个ectd集群。在撰写这篇文档的时候吗，etcd最新的版本是`3.5`，我们选择一个较新较稳定的版本`3.4.18`，具体操作如下：


先创建一个ectd用户，禁止该用户远程登录，并且没有家目录，如下命令

```shell
useradd -s /sbin/nologin -M etcd
```


安装

```shell
# 下载对应版本
wget https://github.com/etcd-io/etcd/releases/download/v3.1.18/etcd-v3.1.18-linux-amd64.tar.gz
# 解压
tar -zxvf etcd-v3.1.18-linux-amd64.tar.gz
# 将etc移到/opt目录，并修改etcd目录名
mv etcd-v3.1.18-linux-amd64/ /opt/etcd-v3.1.18
# 创建etcd软链接
ln -s /opt/etcd-v3.1.18 /opt/etcd
# 创建etcd证书目录和数据目录
mkdir -p /opt/etcd/certs /data/etcd /data/logs/etcd-server
# 进入证书目录
cd /opt/etcd/certs
# 从证书服务器下载证书
scp root@kb200.host.com:/opt/certs/ca.pem ./
scp root@kb200.host.com:/opt/certs/etcd/etcd-peer.pem ./
scp root@kb200.host.com:/opt/certs/etcd/etcd-peer-key.pem ./
```

此时，使用`ls -l`命令可查看到两张证书和一个私钥文件，如下内容

```shell
[root@kb12 certs]# ls -l
total 12
-rw-r--r-- 1 root root 1350 Feb 24 17:32 ca.pem
-rw------- 1 root root 1675 Feb 24 17:33 etcd-peer-key.pem
-rw-r--r-- 1 root root 1415 Feb 24 17:33 etcd-peer.pem
```

给所有etcd用户需要读写的目录赋予权限

```shell
# 将目录权限改为etcd用户
chown -R etcd.etcd /opt/etcd/certs /data/etcd/ /data/logs/etcd-server
```


## 3. 编写etcd启动脚本


我们先在etcd目录编写启动脚本`/opt/etcd/startup.sh`，如下

```shell
#!/bin/sh
./etcd --name etcd-server-12 \
       --data-dir /data/etcd/etcd-server \
       --listen-peer-urls https://192.168.14.12:2380 \
       --listen-client-urls https://192.168.14.12:2379,http://127.0.0.1:2379 \
       --quota-backend-bytes 8000000000 \
       --initial-advertise-peer-urls https://192.168.14.12:2380 \
       --advertise-client-urls https://192.168.14.12:2379,http://127.0.0.1:2379 \
       --initial-cluster etcd-server-12=https://192.168.14.12:2380,etcd-server-21=https://192.168.14.21:2380,etcd-server-22=https://192.168.14.22:2380 \
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


## 4. 使用supervisord来启动etcd

现在我们要安装supervisord，用于管理etcd服务

```shell
# 安装supervisor
yum install supervisor -y
# 启动supervisord
systemctl start supervisord
# 让superivisord开机自启
systemctl enable supervisord
```

添加etcd的supervisor进程维护脚本`/etc/supervisord.d/etcd-server.ini`，添加以下内容

```shell
[program:etcd-server-12]
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

> 注意：在不同的主机上使用不同的服务名称，这样好辨别，如kb12使用`etcd-server-12`，如kb21使用`etcd-server-21`

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
[root@kb12 logs]# supervisorctl status
etcd-server-12                   RUNNING   pid 4945, uptime 0:01:22
```

此时，我们再使用`netstat -luntp | grep etcd`查看网络服务端口，看到如下信息代表etcd已经正常启动

```shell
[root@kb12 logs]# netstat -luntp | grep etcd
tcp        0      0 192.168.14.12:2379      0.0.0.0:*               LISTEN      4946/./etcd         
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      4946/./etcd         
tcp        0      0 192.168.14.12:2380      0.0.0.0:*               LISTEN      4946/./etcd
```

## 5. 集群验证



启动完成之后，我们在任意节点使用etcdctl命令检查集群状态，需要注意的是，要确切指定证书的位置

```shell
./etcdctl --cacert="/opt/etcd/certs/ca.pem" --cert="/opt/etcd/certs/etcd-peer.pem" --key="/opt/etcd/certs/etcd-peer-key.pem" --endpoints="https://192.168.14.12:2379,https://192.168.14.21:2379,https://192.168.14.22:2379" endpoint health --write-out=table
```


如果看到输出的表格中`HEALTH`单元格数据都为`true`，代表 ectd 集群搭建成功

```shell
[root@kb21 etcd]# ./etcdctl --cacert="/opt/etcd/certs/ca.pem" --cert="/opt/etcd/certs/etcd-peer.pem" --key="/opt/etcd/certs/etcd-peer-key.pem" --endpoints="https://192.168.14.12:2379,https://192.168.14.21:2379,https://192.168.14.22:2379" endpoint health --write-out=table
+----------------------------+--------+-------------+-------+
|          ENDPOINT          | HEALTH |    TOOK     | ERROR |
+----------------------------+--------+-------------+-------+
| https://192.168.14.22:2379 |   true | 11.968038ms |       |
| https://192.168.14.21:2379 |   true | 11.394077ms |       |
| https://192.168.14.12:2379 |   true | 11.584124ms |       |
+----------------------------+--------+-------------+-------+
```


如果需要了解`etcdctl`这个指令的更多用法，使用`--help`参数即可查看。

