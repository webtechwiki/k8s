# k8s二进制安装环境准备

## 一、物理机器准备

### 1.1 集群主机概述

准备基础的设备如下，所有的机器都是x86的64位架构

|      IP       |            操作系统            |               配置                |        备注        |
| ------------- | ------------------------------ | --------------------------------- | ------------------ |
| 192.168.9.199 | Debian GNU/Linux 11 (bullseye) | 内存:12G + SSD硬盘:256G + CPU:6核 | 标记为: 199-debian |
| 192.168.9.192 | Debian GNU/Linux 11 (bullseye) | 内存:8G + SSD硬盘:256G + CPU:6核  | 标记为: 192-debian |
| 192.168.9.160 | Debian GNU/Linux 11 (bullseye) | 内存:8G + SSD硬盘:256G + CPU:6核  | 标记为: 190-debian |

由于`199-debian`这台主机资源相对充足，所以我们使用这台机充当了更多的角色

- 192.168.9.199: etcd服务器、控制节点、Proxy的L4、L7代理，同时作为运维主机：签发证书服务器、dns服务器、Docker的私有仓库、k8s资源配置清单仓库、提供共享存储（NFS）

- 192.168.9.192: etcd服务器、控制节点、Proxy的L4、L7代理、工作节点

- 192.168.9.192: etcd服务器、控制节点、工作节点

### 1.2 二进制组件版本

|       组件        |   版本   |                                                    下载链接                                                     |
| ----------------- | -------- | --------------------------------------------------------------------------------------------------------------- |
| kubernetes-server | v1.24.1  | <https://dl.k8s.io/v1.24.1/kubernetes-server-linux-amd64.tar.gz>                                                |
| etcd              | v3.5.4   | <https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz>                       |
| cni-plugins       | v1.1.1   | <https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz>    |
| crictl            | v1.24.2  | <https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.24.2/crictl-v1.24.2-linux-amd64.tar.gz>      |
| containerd        | 1.6.4    | <https://github.com/containerd/containerd/releases/download/v1.6.4/cri-containerd-cni-1.6.4-linux-amd64.tar.gz> |
| docker-ce         | 20.10.14 | <https://download.docker.com/linux/static/stable/x86_64/docker-20.10.14.tgz>                                    |
| nerdctl           | 20.10.14 | <https://github.com/containerd/nerdctl/releases/download/v0.20.0/nerdctl-0.20.0-linux-amd64.tar.gz>             |

我们先把所有的安装包下载到三台服务器，得到如下文件列表

```bash
kubernetes-server-linux-amd64.tar.gz
etcd-v3.5.4-linux-amd64.tar.gz
cri-containerd-cni-1.6.4-linux-amd64.tar.gz
cni-plugins-linux-amd64-v1.1.1.tgz
crictl-v1.24.2-linux-amd64.tar.gz
nerdctl-0.20.0-linux-amd64.tar.gz
docker-20.10.14.tgz
```

## 二、设置统一的DNS

### 2.1 安装DNS服务

我们把 `199-debian` 这台主机作为域名服务器。接下来先安装bind9软件

```bash
apt install -y bind9
```

安装好bind之后，修改bind的配置文件`/etc/named.conf`，修改的配置参数如下

```bash
# 修改服务的IP
listen-on port 53 { 192.168.9.199; };
# 允许任意主机使用解析服务
allow-query     { any; };
# 添加 forwarders 键， 值：网关地址
forwarders      { 192.168.9.254; };
# 开启dns转发
allow-query-cache       { any; };
# 使用递归的方式
recursion yes;
#关闭sec
dnssec-enable no;
dnssec-validation no;
```

### 2.2 配置域名解析

在这里，我们不赘述域名解析以及相关软件的相关知识，如果你在阅读本文有困难时，建议先去了解 **dsn服务器** 以及 **bind9软件** 的相关知识。集群中的所有主机都设置我们自己搭建的DNS服务，以下是具体操作过程：

修改区域配置文件 `/etc/bind/named.conf.local`，我们将`k8s.com`作为集群的业务域名，后续使用该域名访问集群的相关服务，添加如下正向解析区域

```bash
zone "k8s.com" {
    type master;
    file "k8s.com.zone";
};
```

编辑区域数据文件`/etc/bind/k8s.com.zone`，改为如下内容

```ini
$ORIGIN k8s.com.
$TTL 600 ; 10 minutes
@ IN SOA dns.k8s.com. dnsadmin.k8s.com. (
    2022012002 ; serial
    10800      ; refresh (3 hours)
    900        ; retry (15 minutes)
    604800     ; expire (1 week)
    86400      ; minium (1 day)
    ) 
    NS dns.k8s.com.

$TTL 60    ;   1 minute
dns             A      192.168.9.199
199-debian      A      192.168.9.199
192-debian      A      192.168.9.192
160-debian      A      192.168.9.160
```

修改`/etc/bind/named.conf.options`，找到`forwarders`关键字取消该“配置块”的注释，并设置上游dns服务器，我的局域网真实的dns是`192.168.9.253`，修改位如下内容

```bash
forwarders {
    192.168.9.253;
};
```

启动dns服务

```bash
# 启动dns 服务
systemctl start named
# 将dns 服务设置为开机自启
systemctl enable named
```

通过查询53端口服务，判定dns服务器是否正常运行

```bash
netstat -luntp | grep 53
```

可以看到53端口服务正常运行，如下图所示

![20220916182709](img/20220916182709.png)

通过以下命令检测`199-debian`这条dns配置是否能正常解析

```bash
dig -t A 199-debian.k8s.com @192.168.9.199 +short
```

如果看到正常返回IP地址，则代表DNS服务正常工作。这时候，修改三台主机的DNS为我们自建的DNS。在每一台主机修改 `/etc/network/interfaces` 网卡配置文件中的`dns-nameservers`项，追加自建DNS并设置在第一个，如下

```ini
# 追加自建DNS：192.168.9.199
dns-nameservers 192.168.9.199 192.168.9.253 192.168.9.252
```

再重启网络服务

```bash
/etc/init.d/networking restart
```

重启网络之后，要检查`/etc/resolv.conf`是否已经加上我们追加的DNS，并检查自定义的域名解析是否生效。

## 三、机器初始化配置

### 3.1 安装ipvsadm

ipvsadm用于管理IPVS，在所有主机上操作

```bash
# 安装
apt install ipvsadm ipset sysstat conntrack -y
# 写入配置
cat >> /etc/modules-load.d/ipvs.conf <<EOF 
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF
# 重新加载IPVS系统模块
systemctl restart systemd-modules-load.service
```

查看已载入系统的模块

```bash
lsmod | grep -e ip_vs -e nf_conntrack
```

返回如下类似内容代表正确安装

```bash
root@debian:/home/debian# lsmod | grep -e ip_vs -e nf_conntrack
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 184320  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack_netlink    57344  0
nf_conntrack          176128  6 xt_conntrack,nf_nat,xt_nat,nf_conntrack_netlink,xt_MASQUERADE,ip_vs
nf_defrag_ipv6         24576  2 nf_conntrack,ip_vs
nf_defrag_ipv4         16384  1 nf_conntrack
nfnetlink              20480  5 nft_compat,nf_conntrack_netlink,nf_tables,ip_set
libcrc32c              16384  5 nf_conntrack,nf_nat,nf_tables,raid456,ip_v
```

### 3.2 修改内核参数

在所有主机上操作

```bash
# 写入内核配置文件
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# 让配置立即生效
sysctl -p /etc/sysctl.d/k8s.conf
```

### 3.3 设置主机名

`199-debian`设置主机名如下命令

```bash
hostnamectl set-hostname 199-debian
```

`192-debian`设置主机名如下命令

```bash
hostnamectl set-hostname 192-debian
```

`160-debian`设置主机名如下命令

```bash
hostnamectl set-hostname 160-debian
```

所有主机追加静态解析，如下

```bash
cat >> /etc/hosts <<EOF
192.168.9.199   199-debian
192.168.9.192   192-debian
192.168.9.160   160-debian
EOF
```
