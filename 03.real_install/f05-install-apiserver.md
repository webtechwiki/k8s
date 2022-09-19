# 安装控制节点的apiserver

搭建好etcd数据库集群之后，我们就可以安装apiserver组件了，在撰写这篇文档的时候，我们选择`1.15.2`的k8s版本，在所有主机上安装apiserver，以下是具体的安装过程

## 一、签发ssl证书

我们先登录上`199-debian`主机，去签发ssl证书，用于k8s集群与etcd数据库的通信使用。

```shell
# 创建证书目录
mkdir -p /opt/certs/kubernetes
# 进入证书目录
cd /opt/certs/kubernetes
```

### 1.1 创建客户端证书

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
        "O": "k8s",
        "OU": "ops"
    }]
}
```

生成证书命令如下

```shell
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=client client-csr.json | cfssljson -bare client
```

执行命令后将会产生三个文件，分别是`client.csr`,`client-key.pem`,`client.pem`，生成的证书用于连接etcd服务通信，apiserver当作ectd的客户端。

### 1.2 创建服务器证书

接下来，我们还要为apiserver创建ssl证书，创建`apiserver-csr.json`文件，添加以下内容

```json
{
    "CN": "apiserver",
    "hosts": [
        "192.168.0.1",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local",
        "192.168.9.199",
        "192.168.9.192",
        "192.168.9.160"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [{
        "C": "CN",
        "ST": "Guangdong",
        "L": "Guangzhou",
        "O": "k8s",
        "OU": "ops"
    }]
}
```

我们把可能添加到集群的主机IP都加到了`hosts`节点，再使用以下命令创建ssl证书

```shell
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=server apiserver-csr.json | cfssljson -bare apiserver
```

执行命令之后生成三个文件，分别是`apiserver.csr`、`apiserver-key.pem`、`apiserver.pem`，我们将在启动apiserver服务的时候将加载证书

> 注意：实际上，我们用的证书信息文件和上篇文章etcd本身的证书信息文件是一样的。只是在部署etcd中我们生成的证书是端对端的证书，而在在本文中生成了另外两套证书。其中k8s作为ectd客户端的时候使用的客户端证书，另外是k8s作为服务器的时候使用的服务器证书。

## 二、安装apiserver

在三台主机上，执行安装操作

```shell
# 下载指定的服务器二进制包
wget https://dl.k8s.io/v1.24.1/kubernetes-server-linux-amd64.tar.gz
# 解压安装包
tar -zxvf kubernetes-server-linux-amd64.tar.gz
# 将安装包移到/opt目录下并根据版本重命名
mv kubernetes /opt/kubernetes-v1.24.1
# 创建软连接
ln -s /opt/kubernetes-v1.24.1/ /opt/kubernetes
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

## 三、启动apiser服务器

在apiserver二进制文件目录创建`/opt/kubernetes/server/bin/kube-apiserver.sh`启动脚本文件，写入以下内容

```shell
#!/bin/bash
./kube-apiserver \
    --v=2 \
    --allow-privileged=true  \
    --service-account-issuer=https://kubernetes.default.svc.cluster.local \
    --authorization-mode=Node,RBAC \
    --bind-address=0.0.0.0 \
    --secure-port=6443  \
    --advertise-address=192.168.9.199 \
    --client-ca-file=/opt/certs/ca.pem \
    --requestheader-client-ca-file=/opt/certs/ca.pem \
    --proxy-client-cert-file=/opt/certs/apiserver/apiserver-front-proxy-client.pem  \
    --proxy-client-key-file=/opt/certs/apiserver/apiserver-front-proxy-client-key.pem  \
    --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
    --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
    --etcd-cafile=/opt/certs/ca.pem \
    --etcd-certfile=/opt/certs/apiserver/apiserver.pem \
    --etcd-keyfile=/opt/certs/apiserver/apiserver-key.pem \
    --etcd-servers=https://192.168.9.199:2379,https://192.168.9.192:2379,https://192.168.9.160:2379 \
    --service-account-key-file=/opt/certs/sa.pub \
    --service-account-signing-key-file=/opt/certs/sa.key \
    --service-cluster-ip-range=10.96.0.0/12 \
    --service-node-port-range=30000-32767 \
    --enable-bootstrap-token-auth=true  \
    --kubelet-client-certificate=/opt/certs/apiserver/apiserver.pem \
    --kubelet-client-key=/opt/certs/apiserver/apiserver-key.pem \
    --tls-cert-file=/opt/certs/apiserver/apiserver.pem \
    --tls-private-key-file=/opt/certs/apiserver/apiserver-key.pem \
    --requestheader-allowed-names=aggregator  \
    --requestheader-group-headers=X-Remote-Group  \
    --requestheader-extra-headers-prefix=X-Remote-Extra-  \
    --requestheader-username-headers=X-Remote-User \
    --enable-aggregator-routing=true
```

参数说明

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
chmod +x kube-apiserver.sh
```

创建日志目录

```shell
mkdir -p /data/logs/kubernetes/kube-apiserver
```

创建supervisor进程配置文件`/etc/supervisor/conf.d/kube-apiserver.conf`

```ini
[program:kube-apiserver-160]
directory=/opt/kubernetes/server/bin
command=/opt/kubernetes/server/bin/kube-apiserver.sh
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

![20220917122956](./img/06-01.png)

此时，还可以使用`netstat -luntp | grep kube-api`命令查看网络服务的端口是否正常，如果正常，将返回如下内容

![20220917123050](./img/06-02.png)
