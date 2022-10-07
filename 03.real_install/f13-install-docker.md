# 扩展:使用二进制安装包安装docker

本文参考[docker官方文档](https://docs.docker.com/engine/install/binaries/)，新版本的kubernetes并不推荐使用docker作为集群的运行时环境，我们也不把docker作为集群的运行时环境。这里安装docker只是备用，或者用于其他用途。

## 一、安装docker

我们要在所有准备的节点上安装docker，以下是安装的过程

docker二进制安装包的下载地址为：[https://download.docker.com/linux/static/stable/](https://download.docker.com/linux/static/stable/)

需要注意的是，如果使用docker最为k8s的运行时，安装的docker版本必须对应上安装的k8s版本。理论上越新的k8s版本支持的docker版本越多，但要注意的是，如果选择太新的docker，即使是最新的k8s版本也可能来还来不急做适配。这里我们选择一个相对新的版本`20.10.14`。

```shell
# 下载二进制安装包
wget https://download.docker.com/linux/static/stable/x86_64/docker-20.10.14.tgz
# 解压安装包
tar -zxvf docker-20.10.14.tgz
# 赋予所有二进制文件可之行权限
chmod +x docker/*
# 将解压包里的内容移动系统的可执行文件目录中
mv docker/* /usr/bin
```

## 二、配置docker

创建配置文件 `/etc/docker/daemon.json`，设置非https的镜像地址，设置加速镜像地址等，根据自己的情况来配置，我配置的内容如下

```json
{
  "storage-driver": "overlay2",
  "insecure-registries": ["harbor.k8s.com"],
  "registry-mirrors": ["https://g6ogy192.mirror.aliyuncs.com"],
  "bip": "172.7.199.1/24",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "live-restore": true
}
```

需要注意的是，`bip`的配置在不同主机的配置需要不一样，这是生成的容器的IP网段。

跟多的配置项可以参考官方文档：[https://docs.docker.com/engine/reference/commandline/dockerd/#options](https://docs.docker.com/engine/reference/commandline/dockerd/#options)

## 三、配置systemd服务管理

### 3.1 自动挂载cgroup

```bash
# 安装
apt install -y cgroupfs-mount
# 挂在
cgroupfs-mount
# 开机自启动
systemctl enable cgroupfs-mount
```

### 3.2 关于containerd

实际上docker运行的时候会依赖containerd，不过我们在前面已经安装了containerd（路径在/usr/local/bin/containerd），docker二进制安装包包含的containerd（路径在/usr/bin/containerd）就不用管了。

### 3.3 docker服务配置

添加`docker`的服务配置文件`/usr/lib/systemd/system/docker.service`，写入如下内容

```ini
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP 
LimitNOFILE=infinity
LimitNPROC=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

启动与`docker`并设置为开机自启

```bash
systemctl daemon-reload
systemctl start docker.service
systemctl enable docker.service
```

## 四、验证docker

运行一个nginx容器，如下命令

```shell
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
