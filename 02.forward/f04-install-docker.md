# 使用二进制安装包的方式安装docker


本文参考[docker官方文档](https://docs.docker.com/engine/install/binaries/)


## 1. 安装docker

docker二进制安装包的下载地址为：[https://download.docker.com/linux/static/stable/](https://download.docker.com/linux/static/stable/)

需要注意的是，我们现在装的 docker 版本必须对应上即将安装的 k8s 版本，理论上越新的k8s版本支持的docker 版本越多，但要注意的是，如果选择太新的docker，即使是最新的k8s版本也可能来还来不急做适配。这里我们选择一个相对稳定的版本 `19.03.15`。


我们要在k8s工作节点（`kb21`和`kb22`这两台主机）上安装docker，以下是安装的过程

```shell
# 下载二进制安装包
wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.15.tgz
```


## 2. 配置docker


## 3. 启动与验证docker


