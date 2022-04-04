# 安装harbor

harbor是docker镜像管理软件，我们可以使用harbor搭建自己的镜像服务。在编写此文档的时候，harbor的最新版本是`1.10.10`。我们将在`kb200`这台主机上搭建harbor服务，在本文，我们一起探讨harbor服务的搭建过程。


## 1. 下载harbor

harbor的官方下载地址是：[https://github.com/goharbor/harbor/releases](https://github.com/goharbor/harbor/releases)


```shell
# 下载harbor
wget https://github.com/goharbor/harbor/releases/download/v1.10.10/harbor-offline-installer-v1.10.10.tgz
# 解压harbor到/opt/目录
tar -xzvf harbor-offline-installer-v1.10.10.tgz -C /opt/
```

此时，harbor解压所有解压内容在 `/opt/harbor`目录中，首先需要编辑harbor目录下的配置文件`/opt/harbor/harbor.yml`，修改内容如下

```yaml
# 修改主机名
hostname: harbor.od.com
# 修改http服务，因为我们将来还在这台主机运行nginx，所以harbor将harbor端口改为非80端口
http:
	port: 180
# 修改管理员的默认密码
harbor_admin_password: harbor123456
# 修改数据的地址
data_volume: /data/harbor
# 修改日志地址
log:
	# 日志级别
	level: info
	# 滚动数量
	rotate_count: 50
	# 滚动大小
	rotate_size: 200M
	# 日志的位置
	location: /data/harbor/logs

# 注释掉https相关配置
#https:
  # https port for harbor, default is 443
  #port: 443
  # The path of cert and key files for nginx
  #certificate: /your/certificate/path
  #private_key: /your/private/key/path
```

创建日志目录

```shell
mkdir -p /data/harbor/logs
```


## 2. 安装docker-compose

harbor依赖于docker-compose做单机编排，所以需要安装docker-compose，docker-compose的二进制安装包下载地址是：[https://github.com/docker/compose/releases/](https://github.com/docker/compose/releases/)

```shell
# 将二进制文件保存到本地
curl -L https://github.com/docker/compose/releases/download/v2.2.3/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
# 赋予二进制文件执行权限
chmod +x /usr/local/bin/docker-compose
```


## 3. 安装harbor

安装好系统里有docker与docker-compose之后，就可以执harbor的安装命令了，执行安装脚本如下

```shell
bash /opt/harbor/install.sh
```

执行完成安装脚本之后，可以使用`docker-compose ps`查看容器是否正常运行。如果现实如下相似内容，代表harbor已经正常运行


```shell
[root@kb200 harbor]# docker-compose ps
NAME                COMMAND                  SERVICE             STATUS              PORTS
harbor-core         "/harbor/harbor_core"    core                running (healthy)   
harbor-db           "/docker-entrypoint.…"   postgresql          running (healthy)   5432/tcp
harbor-jobservice   "/harbor/harbor_jobs…"   jobservice          running (healthy)   
harbor-log          "/bin/sh -c /usr/loc…"   log                 running (healthy)   127.0.0.1:1514->10514/tcp
harbor-portal       "nginx -g 'daemon of…"   portal              running (healthy)   8080/tcp
nginx               "nginx -g 'daemon of…"   proxy               running (healthy)   0.0.0.0:180->8080/tcp
redis               "redis-server /etc/r…"   redis               running (healthy)   6379/tcp
registry            "/home/harbor/entryp…"   registry            running (healthy)   5000/tcp
registryctl         "/home/harbor/start.…"   registryctl         running (healthy)
```



## 4. 安装nginx


### 4.1. 源码安装

nginx的源代码的下载地址是：[https://nginx.org/en/download.html](https://nginx.org/en/download.html)


在编写文档的时候，nginx的最新版本是`1.21.6`，下载nginx源代码，并编译安装，命令如下：

```shell
# 下载云代码
wget https://nginx.org/download/nginx-1.21.6.tar.gz
# 解压文件
tar -zxvf nginx-1.21.6.tar.gz
# 进入源码目录
cd nginx-1.21.6
# 配置编译参数，--prefix参数指定安装目录
./configure --prefix=/usr/local/nginx
# 编译并安装
make && make install
```


编译nginx时，可能会提示你缺少相关的依赖库，根据提示进行安装即可。


我们在nginx的配置文件`/usr/local/nginx/conf/nginx.conf`中添加以下`server`节点

```
server {
	listen  80;
	server_name  harbor.od.com;
	client_max_body_size  1000m;
	location / {
		proxy_pass http://127.0.0.1:180;
	}
}
```



配置好了之后，我们通过下命令启动nginx

```shell
/usr/local/nginx/sbin/nginx
```

### 4.2. yum软件包安装


**设置反向代理**

可以yum安装命令如下

```shell
yum install -y nginx
```


添加配置文件`/etc/nginx/conf.d/harbor.od.conf`，并加入以下内容

```shell
server {
	listen  80;
	server_name  harbor.od.com;
	client_max_body_size  1000m;
	location / {
		proxy_pass http://127.0.0.1:180;
	}
}
```

配置好之后，使用以下命令启动nginx

```shell
# 检测配置文件是否正确
nginx -t
# 启动nginx服务
nginx
```


**开启IP转发功能**

如果我们的IP没有开启转发功能，实际上harbor对于外网的服务可能是不正常的， 使用 `sysctl net.ipv4.ip_forward` 命令查看 IP是否已开启了转发功能，如果返回以下内容，代表没有正常开启

```shell
net.ipv4.ip_forward = 0
```


如果没有开启，使用以下步骤进行开启

第一步， 编辑`/etc/sysctl.conf`文件，添加以下内容：

```shell
net.ipv4.ip_forward = 1 # 这个配置改成1，0是没打开
```


第二步，重启网络：


```shell-script
service network restart
```


第三步，修改DNS：

当我们重启网络服务之后，虚拟机的DNS重置会vagrant的DNS，我们修改DNS配置文件`/etc/resolv.conf`，将DNS服务器修改为如下

```shell
search host.com
nameserver 192.168.14.11
```



## 5. 添加域名解析

在上面的步骤中，我们发现访问`harbor.od.com`域名的时候并不会解析到kb200这台主机，我们需要回到kb11这台域名解析服务器添加上`harbor.od.com`这个域名。在kb11这台这几上打开`/var/named/od.com.zone`文件，修改为以下内容

```
$ORIGIN od.com.
$TTL 600 ; 10 minutes
@ IN SOA dns.od.com. dnsadmin.od.com. (
    2022012003 ; serial 往后添加一个序列号
    10800      ; refresh (3 hours)
    900        ; retry (15 minutes)
    604800     ; expire (1 week)
    86400      ; minium (1 day)
    )
    NS dns.od.com.

$TTL 60    ;   1 minute
dns    A   10.4.7.11
; 添加harbor的解析
harbor    A   192.168.14.200
```

使用`systemctl restart named`重启域名解析服务，并使用`dig -t A harbor.od.com +short`，如果正常返回`192.168.14.200`则代表解析成功。此时，harbor已经安装成功了，如果我们的电脑本身解析 `harbor.od.com` 到kb200这台主机的话，此时我们能正常访问harbor的web服务。



## 6. 推送镜像

harhor服务安装成功后，我们进入已经安装好docker的`kb21`主机或者`kb22`主机，尝试将镜像推送到harbor中，使用`docker login harbor.od.com`命令登录到harbor。账号是`admin`，密码是配置文件中的`harbor123456`，如果返回类似如下内容，代表登录成功

```shell
[root@kb200 harbor]# docker login harbor.od.com
Username: admin
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```


接下来进行镜像推送操作

```shell
# 从docker官方镜像站拉取nginx:alpine镜像
docker pull nginx:alpine
# 将nginx:alpine镜像打为自建的harbor服务的镜像
docker tag nginx:alpine harbor.od.com/library/nginx:alpine
# 推送镜像
docker push harbor.od.com/library/nginx:alpine
```

如果推送镜像能正常输入以下结果，代表harbor服务正常

```shell
[root@kb200 harbor]# docker push harbor.od.com/library/nginx:alpine
INFO[2022-02-23T15:58:41.697385855Z] Attempting next endpoint for push after error: Get https://harbor.od.com/v2/: dial tcp 127.0.0.1:443: connect: connection refused 
The push refers to repository [harbor.od.com/library/nginx]
419df8b60032: Pushed 
0e835d02c1b5: Pushed 
5ee3266a70bd: Pushed 
3f87f0a06073: Pushed 
1c9c1e42aafa: Pushed 
8d3ac3489996: Pushed 
alpine: digest: sha256:544ba2bfe312bf2b13278495347bb9381ec342e630bcc8929af124f1291784bb size: 1568
```






