# 准备签发证书

我们把 `kb200` 这台主机作为运维主机，所以接下来很多操作都在这台主机上执行，我们先登录到这台主机，准备证书签发操作


## 1. 安装cfssl工具

cfssl 工具的下载地址为：[https://github.com/cloudflare/cfssl/releases](https://github.com/cloudflare/cfssl/releases)


我们下载安装`cfssl`、`cfssl-json`、`cfssl-certingo`，如下指令


```shell
# cfssl
wget https://github.com/cloudflare/cfssl/releases/download/1.2.0/cfssl_linux-amd64 -o /usr/local/bin/cfssl
# cfssljson
wget https://github.com/cloudflare/cfssl/releases/download/1.2.0/cfssljson_linux-amd64 -o /usr/local/bin/cfssl-json
# cfssl-certinfo
wget https://github.com/cloudflare/cfssl/releases/download/1.2.0/cfssl-certinfo_linux-amd64 -o /usr/local/bin/cfssl-certinfo
# 赋予执行权限
chmod a+x /usr/local/bin/cfssl*
```



## 2. 创建证书

```shell
# 创建证书目录
mkdir -p /opt/certs
# 进入目录
cd /opt/certs
```


（1）创建签证机构

创建`/opt/certs/ca-csr.json`文件，此文件定义签证机构（CA）的相关信息，填入以下内容

```json
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Guangzhou",
            "ST": "Guangdong",
            "O": "od",
            "OU": "ops"
        }
    ],
    "ca": {
    	"expiry": "175200h"
    }
}
```

name的相关字段：

`CN`: Common Name，一般使用域名
`C`: Country，国家
`ST`: Sate，州、省
`L`: Locality，地区、城市
`O`: Organization name，组织名称、公司名称
`OU`: Organization Unit Name，组织单位名称、公司部门


ca的expiry字段代表有效时间，175200h代表20年

（2）创建证书公钥和私钥文件

使用 `cfssl` 以下命令使用配置文件中，创建一个签证机构，并使用`cfssl-json`生成承载式证书

```shell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```

生成以下三个文件

`ca.pem`: ca证书
`ca-key.pem`: ca证书私钥，即ssl证书
`ca.csr`: 










