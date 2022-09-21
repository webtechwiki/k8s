# 安装控制节点的apiserver

搭建好etcd数据库集群之后，我们就可以安装apiserver组件了，在所有主机上安装apiserver，以下是具体的安装过程。

## 一、创建kubelet授权用户

因为后面要配置kubelet的bootstrap认证，即kubelet启动时自动创建CSR请求，这里需要在apiserver上开启token的认证。所以先在master上生成一个随机值作为token。下面在一台主机操作即可

```shell
# 创建证书目录
openssl rand -hex 10
```

假设生成的token为`88c916f382dc619a6bca`，把这个token写入到一个文件里，这里写入到 `/etc/kubernetes/bb.csv`，如下

```bash
cat > /etc/kubernetes/bb.csv <<EOF
6440328e1b3a1f4873dc,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF
```

这里第二列定义了一个用户名kubelet-bootstrap，后面在配置kubelet时会为此用户授权。创建好该文件后，同步到其他主机。

## 二、安装apiserver

在三台主机上，执行安装操作

```shell
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

## 三、启动apiser服务

### 3.1 创建启动脚本

在apiserver二进制文件目录创建`/opt/kubernetes/server/bin/kube-apiserver.sh`启动脚本文件，写入以下内容

```bash
#!/bin/bash
./kube-apiserver \
    --v=2 \
    --logtostderr=true \
    --allow-privileged=true \
    --bind-address="192.168.9.160" \
    --secure-port="6443" \
    --token-auth-file="/etc/kubernetes/bb.csv" \
    --advertise-address="192.168.9.160" \
    --service-cluster-ip-range="10.96.0.0/16" \
    --service-node-port-range="30000-60000" \
    --etcd-servers="https://192.168.9.160:2379" \
    --etcd-cafile="/etc/kubernetes/pki/ca.pem" \
    --etcd-certfile="/etc/kubernetes/pki/etcd.pem" \
    --etcd-keyfile="/etc/kubernetes/pki/etcd-key.pem" \
    --client-ca-file="/etc/kubernetes/pki/ca.pem" \
    --tls-cert-file="/etc/kubernetes/pki/apiserver.pem" \
    --tls-private-key-file="/etc/kubernetes/pki/apiserver-key.pem" \
    --kubelet-client-certificate="/etc/kubernetes/pki/apiserver.pem" \
    --kubelet-client-key="/etc/kubernetes/pki/apiserver-key.pem" \
    --service-account-key-file="/etc/kubernetes/pki/ca-key.pem" \
    --service-account-signing-key-file="/etc/kubernetes/pki/ca-key.pem" \
    --service-account-issuer="https://kubernetes.default.svc.cluster.local" \
    --kubelet-preferred-address-types="InternalIP,ExternalIP,Hostname" \
    --enable-admission-plugins="NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota" \
    --authorization-mode="Node,RBAC" \
    --enable-bootstrap-token-auth=true
    #--requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
    #--proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
    #--proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
    #--requestheader-allowed-names=aggregator  \
    #--requestheader-group-headers=X-Remote-Group  \
    #--requestheader-extra-headers-prefix=X-Remote-Extra-  \
    #--requestheader-username-headers=X-Remote-User
```

赋予执行权限

```shell
chmod +x /opt/kubernetes/server/bin/kube-apiserver.sh
```

上面注释的部分是配置聚合层的，本环境里没有启用聚合层所以这些选项被注释了，如果配置了聚合层的话，则需要把#取消。相关参数说明

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
`--requestheader-allowed-names`：允许访问的客户端 common names 列表，通过 header 中 –requestheader-username-headers 参数指定的字段获取。客户端 common names 的名称需要在 client-ca-file 中进行设置，将其设置为空值时，表示任意客户端都可访问。

以上是我们在启动apiserver的时候常用的参数，apiserver具有很多参数，很多参数也有默认值，可以`./kube-apiserver --hep`命令查看更多的帮助。

### 3.2 使用supervisor运行

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
stdout_logfile=/data/logs/supervisor/apiserver.stdout.log
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
