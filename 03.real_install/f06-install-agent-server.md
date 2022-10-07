# 搭建L4反向代理服务

我们需要在`199-debian`和`192-debian`上安装nginx作为反向代理服务，然后使用keepalived保证高可用性

## 一、安装nginx

`199-debian`在安装harbor时已经安装过，需要继续在`192-debian`上安装

```bash
# 下载代码
wget https://nginx.org/download/nginx-1.22.0.tar.gz
# 解压文件
tar -zxvf nginx-1.22.0.tar.gz
# 进入源码目录
cd nginx-1.22.0
# 配置编译参数，--prefix参数指定安装目录
./configure \
--prefix=/usr/local/nginx-1.22.0 \
--with-stream \
--with-http_stub_status_module \
--with-http_ssl_module --with-http_v2_module \
--error-log-path=/data/log/nginx/error.log \
--http-log-path=/data/log/nginx/access.log
# 编译并安装
make && make install
```

## 二、配置nginx

安装完成之后，我们需要两台机的nginx的配置文件`/usr/local/nginx/conf/nginx.conf`的`http`节点旁边添加四层反向代码规则，将7443端口的流量使用负载均衡的方式转发到3台主机的6443端口上

```shell
# 设置代理规则
stream {
    upstream kube-apiserver {
        server 192.168.9.199:6443  max_fails=3  fail_timeout=30s;
        server 192.168.9.192:6443  max_fails=3  fail_timeout=30s;
        server 192.168.9.160:6443  max_fails=3  fail_timeout=30s;
    }
    server {
        listen  7443;
        proxy_connect_timeout  2s;
        proxy_timeout  900s;
        proxy_pass kube-apiserver;
    }
}
```

在两台主机上配置好规则之后，通过`nginx -t`命令检查配置结果，如果输出以下内容代表配置正确

```shell
[root@199-debian vagrant]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

配置成功之后，启动nginx，如下指令

```bash
# 设置链接
ln -s /usr/local/nginx-1.22.0 /usr/local/nginx
ln -s /usr/local/nginx/sbin/nginx /usr/local/bin/nginx
# 启动ginx，199主机使用 nginx -s reload重新加载配置即可
nginx
```

## 三、安装keepalived

我们将使用keepalived实现代理服务器的高可用，以下是安装过程

```shell
apt install keepalived -y
```

在两台主机的创建`/etc/keepalived/check_port.sh`脚本文件，添加以下内容

```shell
#!/bin/bash
CHK_PORT=$1
if [ -n "$CHK_PORT" ]; then
    PORT_PROCESS=`ss -lnt|grep $CHK_PORT|wc -l`
    if [ $PORT_PROCESS -eq 0 ]; then
        echo "Port $CHK_PORT Is Not Used, End"
        exit 1
    fi
else
    echo "Check Port Cant Be Empty!"
fi
```

添加执行权限

```shell
chmod +x /etc/keepalived/check_port.sh
```

以上的操作就准备好keepalived的基础环境了，接下来我们使用`199-debian`这台主机作为主节点，使用`192-debian`作为重节点，进行以下配置

`199-debian`作为主节点，修改`/etc/keepalived/keepalived.conf`配置文件如下

```bash
! Configuration File for keepalived

global_defs {
   router_id 192.168.9.199
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_port.sh 7443"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface enp1s0
    virtual_router_id 251
    priority 100
    advert_int 1
    mcast_src_ip 192.168.9.199
    nopreempt

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.9.190
    }
}
```

`192-debian`作为从节点，修改`/etc/keepalived/keepalived.conf`配置文件如下

```bash
! Configuration File for keepalived

global_defs {
   router_id 192.168.9.192
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_port.sh 7443"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface enp3s0
    virtual_router_id 251
    priority 90
    advert_int 1
    mcast_src_ip 192.168.9.192
    nopreempt

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.9.190
    }
}
```

启动服务

```shell
# 重启服务
systemctl restart keepalived
# 设置服务为开机自启
systemctl enable keepalived
```

需要注意的是，`interface`参数对应的是真实的主机网卡名称，`virtual_router_id`参数需要在同一个虚拟IP的前提下，设置需一致。

## 四、验证

通过`ping 192.169.9.190`的方式，如果有正常返回，代表keepalived运行正常。
