# 安装etcd

我们将使用`kb12`、`kb21`、`kb22`这三台主机作为控制节点，首先在控制节点上搭建ectd集群。

## 1. 准备ssl证书

登录到`kb200`证书签发服务器。在本系列的第三篇中，我们已经进行了根证书的签发。现在我们需要基于证书服务器的根证书为etcd服务创建ssl证书。以下 是具体的签发过程


**创建证书请求文件**

创建证书保存目录

```shell
# 在根证书目录下创建专门存放etcd证书的目录
mkdir -p /opt/certs/etcd
# 进入etcd证书目录
cd /opt/certs/etcd
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
            "server auth",
            "client auth"
        ]
      },
      "client": {
         "expiry": "175200h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      },
      "peer": {
         "expiry": "175200h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "peer auth"
        ]
      }
    }
  }
}
```

上面的文件定义了证书的基本信息，其中`profiles`里可以包含多个档案对象，我们定了三个，在这里我们把`server`定义为服务器专用，`client`是客户端专用，`peer`是服务器与客户端通用，接下来我们将基于`peer`属性创建证书。


创建证书请求文件`etcd-csr.json`，写入以下内容

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

