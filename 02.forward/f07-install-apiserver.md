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

生成证书，生成的证书用于与etcd数据通信

```shell
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=ca-config.json -profile=client client-csr.json | cfssljson -bare client
```


> 实际上，我们用的证书信息文件和上篇文章etcd本身的证书信息文件是一样的，只是我们使用 client 节点的配置


完成P18, 9:27

