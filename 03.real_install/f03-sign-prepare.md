# 证书签发 环境准备

我们把 `199-debian` 这台主机作为运维主机，所以接下来很多操作都在这台主机上执行，我们先登录到这台主机，准备证书签发操作

## 一、安装cfssl工具

cfssl 工具的下载地址为：[https://github.com/cloudflare/cfssl/releases](https://github.com/cloudflare/cfssl/releases)

我们下载安装`cfssl`、`cfssl-json`、`cfssl-certingo`，如下指令

```bash
# cfssl
wget https://github.com/cloudflare/cfssl/releases/download/1.2.0/cfssl_linux-amd64 -o /usr/local/bin/cfssl
# cfssljson
wget https://github.com/cloudflare/cfssl/releases/download/1.2.0/cfssljson_linux-amd64 -o /usr/local/bin/cfssljson
# cfssl-certinfo
wget https://github.com/cloudflare/cfssl/releases/download/1.2.0/cfssl-certinfo_linux-amd64 -o /usr/local/bin/cfssl-certinfo
# 赋予执行权限
chmod a+x /usr/local/bin/cfssl*
```

## 二、创建根证书

### 2.1 创建证书目录

```bash
# 创建证书目录
mkdir -p /opt/certs
# 进入目录
cd /opt/certs
```

### 2.2 定义自签证机构（CA）的根证书信息

创建`/opt/certs/ca-csr.json`证书信息，此文件定义签证机构（CA）的相关信息，定义好之后用于生成根证书，填入以下内容

```json
{
    "CN": "CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Guangzhou",
            "ST": "Guangdong",
            "O": "k8s",
            "OU": "CA"
        }
    ],
    "ca": {
        "expiry": "175200h"
    }
}
```

证书根字段说明

- `CN`：证书名称
- `key`：定义证书类型，algo为加密类型，size为加密长度
- `ca.expiry`：证书有效时间

names条目字段说明

- `CN`: Common Name，一般使用域名
- `C`: Country Code，申请单位所属国家，只能是两个字母的国家码。例如，中国只能是CN。
- `ST`: State or Province，省份名称或自治区名称
- `L`: Locality，城市或自治州名
- `O`: Organization name，组织名称、公司名称
- `OU`: Organization Unit Name，组织单位名称、公司部门

ca的expiry字段代表有效时间，175200h代表20年

### 2.3 生成自签证机构的根证书

申请数字证书之前，必须先生成证书的密钥文件和CSR文件。CSR文件是你的公钥证书原始文件，包含了你的服务器信息和你的单位信息，需要提交给CA认证中心进行审核。

使用 `cfssl` 命令生成证书，但生成的证书内容只会输入在控制台中，所以需要使用`cfssl-json`生成承载式证书写入到文件中，如下命令

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

生成以下三个文件

`ca.csr`:  证书签名申请（Certificate Signing Request）文件
`ca.pem`: ca公钥证书
`ca-key.pem`: ca私钥证书

生成的三个文件也就是根证书包含的内容。在后续，我们给各个服务颁发证书的时候，都基于CA根证书来颁发。

### 2.4 定义证书颁发的配置信息

我们先在`/opt/certs/`创建`ca-config.json`文件，定义证书的配置信息，写入如下内容

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

上面的文件定义了证书的基本信息，其中`profiles`里可以包含多个档案对象,分别用于三种场景，如下：

`server`：服务器去连接客户端时，需要的证书；
`client`：客户端去连接服务器时，需要的证书；
`peer`：服务器与客户端交换数据，都需要的证书。

在后续的证书颁发，都给予该份配置文件，使用自建签证机构CA来颁发证书，后续申请证书的时候，我们大多使用peer。

## 三、颁发各个服务所需的证书

### 3.1 etcd服务器

定义证书信息

```bash
mkdir -p /opt/certs/etcd
cd /opt/certs/etcd
cat > /opt/certs/etcd/etcd-peer-csr.json <<EOF
{
    "CN": "etcd",
    "hosts": [
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
        "OU": "etcd"
    }]
}
EOF
```

生成证书

```bash
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=peer etcd-peer-csr.json | cfssljson -bare etcd-peer
```

### 3.2 kube-apiserver

#### 3.2.1 服务器与客户端证书

定义证书信息

```bash
mkdir -p /opt/certs/apiserver
cd /opt/certs/apiserver
cat > /opt/certs/apiserver/apiserver-csr.json <<EOF
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
        "OU": "apiserver"
    }]
}
EOF
```

生成证书

```bash
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=peer apiserver-csr.json | cfssljson -bare apiserver
```

证书用于连接etcd服务器、连接keubelet使用的SSL证书，也用于本身服务的证书。

#### 3.2.2 apiserver聚合证书

定义证书信息

```bash
cat > /opt/certs/apiserver/apiserver-front-proxy-client-csr.json <<EOF
{
  "CN": "apiserver-front-proxy-client",
  "key": {
     "algo": "rsa",
     "size": 2048
  }
}
EOF
```

生成证书

```bash
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=peer apiserver-front-proxy-client-csr.json | cfssljson -bare apiserver-front-proxy-client
```

### 3.3 kube-controller-manager

定义证书信息

```bash
mkdir -p /opt/certs/controller-manager/
cd /opt/certs/controller-manager/
cat > /opt/certs/controller-manager/controller-manager-csr.json <<EOF
{
    "CN": "apiserver",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [{
        "C": "CN",
        "ST": "Guangdong",
        "L": "Guangzhou",
        "O": "k8s",
        "OU": "controller-manager"
    }]
}
EOF
```

生成证书

```bash
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=peer controller-manager-csr.json | cfssljson -bare controller-manager
```

### 3.4 kube-proxy

定义证书信息

```bash
mkdir -p /opt/certs/proxy/
cd /opt/certs/proxy/
cat > /opt/certs/proxy/proxy-csr.json <<EOF
{
    "CN": "apiserver",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [{
        "C": "CN",
        "ST": "Guangdong",
        "L": "Guangzhou",
        "O": "k8s",
        "OU": "proxy"
    }]
}
EOF
```

生成证书

```bash
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=peer proxy-csr.json | cfssljson -bare proxy
```

### 3.5 kube-scheduler

定义证书信息

```bash
mkdir -p /opt/certs/scheduler/
cd /opt/certs/scheduler/
cat > /opt/certs/scheduler/scheduler-csr.json <<EOF
{
    "CN": "apiserver",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [{
        "C": "CN",
        "ST": "Guangdong",
        "L": "Guangzhou",
        "O": "k8s",
        "OU": "scheduler"
    }]
}
EOF
```

生成证书

```bash
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=peer scheduler-csr.json | cfssljson -bare scheduler
```

### 3.6 创建ServiceAccount Key

```bash
openssl genrsa -out /opt/certs/sa.key 2048
openssl rsa -in /opt/certs/sa.key -pubout -out /opt/certs/sa.pub
```
