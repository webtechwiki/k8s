# 使用二进制安装包的方式安装docker


本文参考[docker官方文档](https://docs.docker.com/engine/install/binaries/)


## 1. 安装docker

### 1.1. 包管理工具安装


使用官方脚本自动安装，执行如下命令

```shell
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```


### 1.2. 二进制安装

docker二进制安装包的下载地址为：[https://download.docker.com/linux/static/stable/](https://download.docker.com/linux/static/stable/)

需要注意的是，我们现在装的 docker 版本必须对应上即将安装的 k8s 版本，理论上越新的k8s版本支持的docker 版本越多，但要注意的是，如果选择太新的docker，即使是最新的k8s版本也可能来还来不急做适配。这里我们选择一个相对稳定的版本 `19.03.15`。


我们要在k8s工作节点（`kb21`和`kb22`这两台主机）上安装docker，同时还需要在运维主机上安装docker（kb200），以下是安装的过程

```shell
# 下载二进制安装包
wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.15.tgz
# 解压安装包
tar -zxvf docker-19.03.15.tgz
# 将解压包里的内容移动系统的可执行文件目录中
mv docker/* /usr/local/bin/
```


## 2. 配置docker


创建配置文件 `/etc/docker/daemon.json`，设置阿里云的加速镜像地址，如下内容

```json
{
  "storage-driver": "overlay2",
  "insecure-registries": ["harbor.od.com"],
  "registry-mirrors": ["https://g6ogy192.mirror.aliyuncs.com"],
  "bip": "172.7.21.1/24",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "live-restore": true
}
```

跟多的配置项可以参考官方文档：[https://docs.docker.com/engine/reference/commandline/dockerd/#options](https://docs.docker.com/engine/reference/commandline/dockerd/#options)


## 3. 启动与验证docker

使用后台运行的方式启动`dockerd`服务

```shell
# 启动docker服务
dockerd &
# 运行一个nginx容器
docker run --name nginx -p 80:80 -d nginx:alpine
```

使用`curl http://127.0.0.1`命令验证nginx容器是否正常运行，如果输出以下内容，代表docker已经正常工作

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

