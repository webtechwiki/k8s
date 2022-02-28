# 安装控制节点的apiserver

搭建好etcd数据库之后，我们就可以安装apiserver组件了，在撰写这篇文档的时候，k8s最新的稳定版本是`1.23.4`，我们选择较新与稳定的`1.20.15`在`kb21`和`kb22`这两台主机安装，以下是具体的安装过程


## 1. 安装apiserver

```shell
# 下载指定的服务器二进制包
wget https://dl.k8s.io/v1.20.15/kubernetes-server-linux-amd64.tar.gz
# 解压安装包
tar -zxvf kubernetes-server-linux-amd64.tar.gz
# 将安装包移到/opt目录下并根据版本重命名
mv kubernetes /opt/kubernetes-v1.20.15
# 创建软连接
ln -s /opt/kubernetes-v1.20.15/ /opt/kubernetes
```


在k8s二进制安装目录里包含了k8s源码包，还包含k8s核心组件的docker镜像，因为我们的核心服务不运行在容器里，所以可以删除掉，操作过程如下

```shell
# 进入k8s目录
cd /opt/kubernetes
# 删除源代码
rm kubernetes-src.tar.gz
# 删除二进制文件目录下以tar作为名称后缀的docker镜像包
rm -rf server/bin/*.tar
```



## 2. 签发ssl证书

我们先登录上`kb200`主机，去签发ssl证书，用于k8s集群与etcd数据库的通信使用。

```shell
# 创建证书目录
mkdir -p /opt/certs/kubernetes
# 进入证书目录
cd /opt/certs/kubernetes
```


定义证书信息，创建`ca-config.json`文件，定义证书的基本信息，写入如下内容

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


创建用于证书签名请求文件（csr）的json配置文件`client-csr.json`，写入如下内容

```json
{
    "CN": "k8s-node",
    "hosts": [
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

生成证书命令如下

```shell
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=ca-config.json -profile=client client-csr.json | cfssljson -bare client
```

执行命令后将会产生三个文件，分别是`client.csr`,`client-key.pem`,`client.pem`，生成的证书用于连接etcd服务通信，k8s当作ectd的客户端。



接下来，我们还要为apiserver创建ssl证书，创建`apiserver-csr.json`文件，添加以下内容

```json
{
    "CN": "etcd",
    "hosts": [
        "192.168.0.1",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local",
        "192.168.14.10",
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

我们把可能添加到集群的主机IP都加到了`hosts`节点，再使用以下命令创建ssl证书

```shell
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=ca-config.json -profile=server apiserver-csr.json | cfssljson -bare apiserver
```

执行命令之后生成三个文件，分别是`apiserver.csr`、`apiserver-key.pem`、`apiserver.pem`，我们将在启动apiserver服务的时候将加载证书



> 注意：实际上，我们用的证书信息文件和上篇文章etcd本身的证书信息文件是一样的。只是在部署etcd中我们生成的证书是端对端的证书，而在在本文中生成了另外两套证书。其中k8s作为ectd客户端的时候使用的客户端证书，另外是k8s作为服务器的时候使用的服务器证书。


## 3. 启动apiser服务器

我们回到`kb21`和`kb22`这两台主机，继续执行安装操作


```shell
# 创建证书目录
mkdir -p /opt/kubernetes/server/bin/certs
# 进入证书目录
cd /opt/kubernetes/server/bin/certs
# 从证书服务器下载证书
scp root@HDSS7-200.host.com:/opt/certs/ca.pem ./
scp root@HDSS7-200.host.com:/opt/certs/ca-key.pem ./
scp root@HDSS7-200.host.com:/opt/certs/kubernetes/apiserver.pem ./
scp root@HDSS7-200.host.com:/opt/certs/kubernetes/apiserver-key.pem ./
scp root@HDSS7-200.host.com:/opt/certs/kubernetes/client.pem ./
scp root@HDSS7-200.host.com:/opt/certs/kubernetes/client-key.pem ./
# 创建apiserver启动配置文件目录
mkdir -p /opt/kubernetes/server/bin/conf
```


在apiserver二进制文件目录创建`startup.sh`启动脚本文件，写入以下内容

```shell
./kube-apiserver \
    --apiserver-count 2 \
    --audit-log-path /data/log/kubernetes/kube-apiserver/audit-log \
    --audit-policy-file ./conf/audit.yaml \
    --authorization-mode RBAC \
    --client-ca-file ./certs/ca.pem \
    --requestheader-client-ca-file ./certs/ca.pem \
    --enable-admission-plugins NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \
    --etcd-cafile ./certs/ca.pem \
    --etcd-certfile ./certs/client.pem \
    --etcd-keyfile ./certs/client-key.pem \
    --etcd-servers https://192.168.14.12:2379,https://192.168.14.21:2379,https://192.168.14.22:2379 \
    --service-account-key-file ./certs/ca-key.pem \
    --service-cluster-ip-range 192.168.0.0/16 \
    --service-node-port-range 3000-29999 \
    --kubelet-client-certificate ./certs/client.pem \
    --kubelet-client-key ./certs/client-key.pem \
    --log-dir /data/logs/kubernetes/kube-apiserver \
    --tls-cert-file ./certs/apiserver.pem \
    --tls-private-key-file ./certs/apiserver-key.pem \
    --service-account-signing-key-file ./certs/apiserver-key.pem \
    --service-account-issuer kubernetes.default.svc \
    --v 2 
```

接下在当前目录下的`conf`目录来创建一个`audit.yaml`，写入内容如下

```shell
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
```


参数说明：

`--apiserver-count`：集群中运行的apiserver的服务数量
`--audit-log-path`：api请求的日志文件目录
`--audit-policy-file`：定义审核策略配置的文件的路径
`--authorization-mode`：授权模式
`--client-ca-file`：访问apiserver时使用，客户端ca文件
`--logtostderr`：将输出记录到标准日志，而不是文件，默认是true
`--v`：日志输出级别
`--log-dir`：日志目录，如果为空，日志写在当前目录
`--audit-log-maxage`：根据文件名中编码的时间戳，保留旧审核日志文件的最大天数
`--audit-log-maxbackup`：保留旧审核日志文件的最大文件数量
`--audit-log-maxsize`：日志循环前，文件最大小的最大M数
`--bind-address`：--secure-port参数指定的端口号对应监听的IP地址，如果没有指定地址（0.0.0.0或者::），默认是 0.0.0.0，代表所有的网卡都在监听服务
`--secure-port`：https服务的端口号，默认是6443
`--advertise-address`：向集群广播的ip地址，这个ip地址必须能被集群的其他节点访问，如果不指定，将使用--bind-address，如果不指定--bind-addres，将使用默认网卡
`--allow-privileged`：是否使用超级管理员权限创建容器，默认为false
`--enable-admission-plugins`：允许使用的插件
`--enable-bootstrap-token-auth`：是否使用token的方式来自动颁发证书，如果主机节点比较多的时候，手动颁发证书可能不太现实，可以使用基于token的方式自动颁发证书。本次我们暂未使用
`--token-auth-file`：该文件用于指定api-server颁发证书的token授权。本次我们暂未使用
`--etcd-servers`：各个etcd节点的IP和端口号
`--etcd-cafile`：访问etcd时使用，ectd的ca文件
`--etcd-certfile`：访问etcd时使用，ectd的证书文件
`--etcd-keyfile`：访问etcd时使用，ectd的证书私钥文件
`--service-cluster-ip-range`：创建service时，使用的虚拟网段
`--service-node-port-range`：创建service时，服务端口使用的端口范围（默认 30000-32767）
`--service-account-key-file`：包含 PEM 编码的 x509 RSA 或 ECDSA 私钥或公钥的文件，用于验证服务帐户令牌，可以是多个。如果没有指定，使用--tls-private-key-file指定的文件
`--tls-cert-file`：访问apiserver时使用，tls证书文件
`--tls-private-key-file`：访问apiserver时使用，tls证书私钥文件
`--kubelet-client-certificate`：访问kubelet时使用，客户端证书路径
`--kubelet-client-key`：访问kubelet时使用，客户端证书私钥


以上是我们在启动apiserver的时候常用的参数，apiserver具有很多参数，很多参数也有默认值，可以`./kube-apiserver --hep`命令查看更多的帮助。


赋予启动简直执行权限

```shell
chmod +x startup.sh
```

创建日志目录

```shell
mkdir -p /data/logs/kubernetes/kube-apiserver
```


创建supervisor进程配置文件`/etc/supervisord.d/kube-apiserver.ini`

```shell
[program:kube-apiserver-21]
directory=/opt/kubernetes/server/bin
command=/opt/kubernetes/server/bin/startup.sh
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
stdout_logfile=/data/logs/kubernetes/kube-apiserver/apiserver.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_event_enabled=false
```


更新supervisor服务
```shell
supervisorctl update
```

再使用`supervisorctl status`命令查看apiserver启动状态，如果显示如下内容，代表正常服务

```shell
[root@kb21 bin]# supervisorctl status
etcd-server-21                   RUNNING   pid 997, uptime 4:56:47
kube-apiserver-21                RUNNING   pid 3654, uptime 0:00:55
```




