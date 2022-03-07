# 安装L4反向代理服务


## 1. 安装过程

我们需要在`kb11`和`kb12`上安装keepalived

```shell
# 安装nginx
yum install nginx -y
# 安装nginx的stream模块，用于L4的反向代理
yum install nginx-mod-stream -y
```

安装完成之后，我们需要在nginx的配置文件`/etc/nginx/nginx.conf`加载`stream`添加四层反向代码规则

```shell
# 加载stream模块
load_module /usr/lib64/nginx/modules/ngx_stream_module.so;
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

通过以上的配置可知，我们将`kb11`和`kb12`本机的7443端口反向代码到了`kb21`和`kb22`的6443端口。在两台主机上配置好规则之后，通过`nginx -t`命令检查配置结果，如果输出以下内容代表配置正确





`P19 06:10`



