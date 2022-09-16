# 证书签发 环境准备

我们把 `kb200` 这台主机作为运维主机，所以接下来很多操作都在这台主机上执行，我们先登录到这台主机，准备证书签发操作

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

## 二、创建证书

```bash
# 创建证书目录
mkdir -p /opt/certs
# 进入目录
cd /opt/certs
```

### 2.1 创建签证机构（CA）信息

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

names 的相关字段：

`CN`: Common Name，一般使用域名
`C`: Country Code，申请单位所属国家，只能是两个字母的国家码。例如，中国只能是CN。
`ST`: State or Province，州、省
`L`: Locality，州名或省份名称
`O`: Organization name，组织名称、公司名称
`OU`: Organization Unit Name，组织单位名称、公司部门

ca的expiry字段代表有效时间，175200h代表20年

### 2.2. 生成根证书 的 CSR 文件和秘钥文件

申请数字证书之前，必须先生成证书的密钥文件和CSR文件。CSR文件是你的公钥证书原始文件，包含了你的服务器信息和你的单位信息，需要提交给CA认证中心进行审核。

使用 `cfssl` 命令生成证书，但生成的证书内容只会输入在控制台中，所以需要使用`cfssl-json`生成承载式证书写入到文件中，如下命令

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```

生成以下三个文件

`ca.csr`:  证书签名申请（Certificate Signing Request）文件
`ca.pem`: ca公钥证书
`ca-key.pem`: ca私钥证书

我们在修改ssh配置文件，支持后续操作的主机可以下载我们当前主机的文件，使用`vim /etc/ssh/sshd_config`命令修改ssh配置文件，如下

```bash
PasswordAuthentication yes
```

修改好配置文件之后使用以下命令重启服务

```bash
systemctl restart sshd
```
