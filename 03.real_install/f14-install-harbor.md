# 扩展:安装harbor

harbor是docker镜像管理软件，我们可以使用harbor搭建自己的镜像服务。在编写此文档的时候，harbor的最新版本是`2.6.0`。我们将在`199-debian`这台主机上搭建harbor服务，在本文，我们一起探讨harbor服务的搭建过程。

## 一、下载harbor

harbor的官方下载地址是：[https://github.com/goharbor/harbor/releases](https://github.com/goharbor/harbor/releases)

```bash
# 下载harbor
wget https://github.com/goharbor/harbor/releases/download/v2.6.0/harbor-offline-installer-v2.6.0.tgz
# 解压harbor到/opt/目录
tar -xzvf harbor-offline-installer-v2.6.0.tgz -C /opt/
```

此时，harbor解压所有解压内容在 `/opt/harbor`目录中，首先需要拷贝一份harbor目录下的配置文件`cp /opt/harbor/harbor.yml.tmpl /opt/harbor/harbor.yml`，修改内容如下

```yaml
# 修改主机名
hostname: harbor.host.com
# 修改http服务，因为我们将来还在这台主机运行nginx，所以harbor将harbor端口改为非80端口
http:
    port: 180
# 修改管理员的默认密码
harbor_admin_password: Harbor123456
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

```bash
mkdir -p /data/harbor/logs
```

## 二、安装docker-compose

harbor依赖于docker-compose做单机编排，所以需要安装docker-compose，docker-compose的二进制安装包下载地址是：[https://github.com/docker/compose/releases/](https://github.com/docker/compose/releases/)

```bash
# 将二进制文件保存到本地
curl -L https://github.com/docker/compose/releases/download/v2.11.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
# 赋予二进制文件执行权限
chmod +x /usr/local/bin/docker-compose
```

## 三、安装harbor

安装好系统里有docker与docker-compose之后，就可以执harbor的安装命令了，执行安装脚本如下

```bash
bash /opt/harbor/install.sh
```

执行完成安装脚本之后，可以使用`docker-compose ps`查看容器是否正常运行。如果现实如下相似内容，代表harbor已经正常运行

```bash
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

## 四、安装nginx

### 4.1 安装与配置

nginx的源代码的下载地址是：[https://nginx.org/en/download.html](https://nginx.org/en/download.html)

在编写文档的时候，nginx的最新稳定版本是`1.22.0`，下载nginx源代码，并编译安装，命令如下：

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

编译nginx时，可能会提示你缺少相关的依赖库，根据提示进行安装即可。

我们在nginx的配置文件`/usr/local/nginx-1.22.0/conf/nginx.conf`中添加以下`server`节点

```shell
server {
    listen  80;
    server_name  harbor.k8s.com;
    client_max_body_size  1000m;
    location / {
        proxy_pass http://127.0.0.1:180;
    }
}
```

配置好之后，使用以下命令启动nginx

```bash
# 创建nginx软连接
ln -s /usr/local/nginx-1.22.0 /usr/local/nginx
ln -s /usr/local/nginx/sbin/nginx /usr/local/bin/nginx
# 检测配置文件是否正确
nginx -t
# 启动nginx服务
nginx
```

### 4.2 开启IP转发功能

如果我们的IP没有开启转发功能，实际上harbor对于外网的服务可能是不正常的， 使用 `sysctl net.ipv4.ip_forward` 命令查看 IP是否已开启了转发功能，如果返回以下内容，代表没有正常开启

```bash
net.ipv4.ip_forward = 0
```

如果没有开启，使用以下步骤进行开启

第一步， 编辑`/etc/sysctl.conf`文件，添加以下内容：

```bash
net.ipv4.ip_forward = 1 # 这个配置改成1，0是没打开
```

第二步，重启网络，如果已正常开启了IP转发了，则不需要重启网络了

```bash-script
/etc/init.d/networking restart
```

## 五、添加域名解析

在上面的步骤中，我们发现访问`harbor.k8s.com`域名的时候并不会解析到kb200这台主机，我们需要回到`199-debian`这台域名解析服务器添加上`harbor.k8s.com`这个域名。在kb11这台这几上打开`/etc/bind/k8s.com.zone`文件，修改为以下内容

```shell
$ORIGIN k8s.com.
$TTL 600 ; 10 minutes
@ IN SOA dns.k8s.com. dnsadmin.k8s.com. (
    2022012003 ; serial 往后添加一个序列号
    10800      ; refresh (3 hours)
    900        ; retry (15 minutes)
    604800     ; expire (1 week)
    86400      ; minium (1 day)
    )
    NS dns.k8s.com.

$TTL 60    ;   1 minute
dns    A      192.168.9.199
; 添加harbor的解析
harbor    A   192.168.9.199
```

使用`systemctl restart named`重启域名解析服务，并使用`dig -t A harbor.k8s.com +short`，如果正常返回`192.168.14.200`则代表解析成功。此时，harbor已经安装成功了，如果我们的电脑本身解析 `harbor.k8s.com` 到kb200这台主机的话，此时我们能正常访问harbor的web服务。

## 六、推送镜像

harhor服务安装成功后，我们尝试将镜像推送到harbor中，使用`docker login harbor.k8s.com`命令登录到harbor。账号是`admin`，密码是配置文件中的`Harbor123456`。

接下来进行镜像推送操作

```bash
# 从docker官方镜像站拉取nginx:alpine镜像
docker pull nginx:alpine
# 将nginx:alpine镜像打为自建的harbor服务的镜像
docker tag nginx:alpine harbor.k8s.com/library/nginx:alpine
# 推送镜像
docker push harbor.k8s.com/library/nginx:alpine
```

如果没有任何报错，代表harbor服务正常运行。
