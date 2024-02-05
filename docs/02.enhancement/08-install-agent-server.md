# 安装L4反向代理服务


## 1. 安装nginx

我们需要在`kb11`和`kb12`上安装keepalived

```shell
# 安装nginx
yum install nginx -y
# 安装nginx的stream模块，用于L4的反向代理
yum install nginx-mod-stream -y
```

安装完成之后，我们需要在nginx的配置文件`/etc/nginx/nginx.conf`加载`stream`添加四层反向代码规则

```shell
# 设置代理规则
stream {
	upstream kube-apiserver {
		server 192.168.14.21:6443  max_fails=3  fail_timeout=30s;
		server 192.168.14.22:6443  max_fails=3  fail_timeout=30s;
	}
	server {
		listen  7443;
		proxy_connect_timeout  2s;
		proxy_timeout  900s;
		proxy_pass kube-apiserver;
	}
}
```

如果使用手动编译的方式安装`stream`模块，需要手动加载如下代码

```shell-script
# 加载stream模块
load_module /usr/lib64/nginx/modules/ngx_stream_module.so;
```

通过以上的配置可知，我们将`kb11`和`kb12`本机的7443端口反向代码到了`kb21`和`kb22`的6443端口。在两台主机上配置好规则之后，通过`nginx -t`命令检查配置结果，如果输出以下内容代表配置正确

```shell
[root@kb11 vagrant]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```


配置成功之后，启动nginx，如下指令

```shell-script
# 启动ginx
nginx
# 让nginx开机自启
systemctl enable nginx
```

## 2. 安装keepalived

我们将使用keepalived实现代理服务器的高可用，以下是安装过程

```shell
yum install keepalived -y
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


以上的操作就准备好keepalived的基础环境了，接下来我们使用`kb11`这台主机作为主节点，使用`kb12`作为重节点，进行以下配置

`kb11`主节点，修改`/etc/keepalived/keepalived.conf`配置文件如下

```shell
! Configuration File for keepalived

global_defs {
   router_id 192.168.14.11
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_port.sh 7443"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth1
    virtual_router_id 251
    priority 100
    advert_int 1
    mcast_src_ip 192.168.14.11
    nopreempt

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.14.10
    }
}
```


`kb12`从节点，修改`/etc/keepalived/keepalived.conf`配置文件如下

```shell
! Configuration File for keepalived

global_defs {
   router_id 192.168.14.12
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_port.sh 7443"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth1
    virtual_router_id 251
    priority 90
    advert_int 1
    mcast_src_ip 192.168.14.12
    nopreempt

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.14.10
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



