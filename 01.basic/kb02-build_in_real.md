# 在物理机中使用kubeadm快速搭建k8s集群

## 一、概述

在本系列的上一篇文章，演示了在虚拟环境中使用`kubeadm`搭建集群的过程，在本文，将继续演示在在实体机中安装的过程。环境如下

|      IP       |            操作系统            |               配置               |             备注              |
| ------------- | ------------------------------ | -------------------------------- | ----------------------------- |
| 192.168.9.199 | Debian GNU/Linux 11 (bullseye) | 内存:8G + SSD硬盘:256G + CPU:6核 | 作为控制节点，标记为: master1 |
| 192.168.9.192 | Debian GNU/Linux 11 (bullseye) | 内存:8G + SSD硬盘:256G + CPU:6核 | 作为工作节点，标记为: worker1 |
| 192.168.9.160 | Debian GNU/Linux 11 (bullseye) | 内存:8G + SSD硬盘:256G + CPU:6核 | 作为工作节点，标记为: worker2 |

因为安装过程和本系列文章的“[在虚拟机中使用kubeadm快速搭建k8s集群](./kb01-build.md)”操作过程基本一样，只是在本文中我们基于物理机器，并且安装更新的版本。

## 二、环境初始化

### 2.1 关闭虚拟内存

关闭swap内存

```bash
swapoff -a
```

打开`/etc/fstab`文件，找到swap的挂载配置，注释掉即可开机自动关闭，如下

```bash
# /dev/mapper/debian--vg-swap_1 none            swap    sw              0       0
```

### 2.2 安装与配置docker

安装docker

```bash
# 安装阿里云docker镜像服务的GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/debian/gpg | sudo apt-key add -
# 写入阿里云docker镜像服务软件源
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/debian $(lsb_release -cs) stable"
# 更新软件库
apt-get update
# 查看可以安装的docke-ce版本
apt-cache madison docker-ce
# 安装docker最新版
apt-get -y install docker-ce=5:20.10.17~3-0~debian-bullseye
# 固定docker-ce版本
apt-mark hold docker-ce
```

设置镜像下载地址

```bash
# 创建配置目录
mkdir -p /etc/docker
# 写入配置文件
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://g6ogy192.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"] 
}
EOF
# 重新加载服务的配置文件
systemctl daemon-reload
# 重启docker服务
systemctl restart docker
```

`https://g6ogy192.mirror.aliyuncs.com`为我的阿里云镜像个人加速地址。

### 2.3 安装kubeadm、kubelet、kubectl

```bash
# 安装阿里云k8s镜像服务的GPG证书
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
# 添加阿里云k8s镜像源
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
# 更新软件库
apt-get update
# 安装程序
apt-get install -y kubelet=1.20.15-00 kubeadm=1.20.15-00 kubectl=1.20.15-00
# 固定版本
apt-mark hold kubelet kubeadm kubectl
```

## 三、集群镜像准备

使用`kubeadm config images list`命令查看当前集群基础服务所需要的镜像，镜像版本会根据当前的系统环境（当前安装kubeadm的版本）而定，返回如下内容

```bash
k8s.gcr.io/kube-apiserver:v1.20.15
k8s.gcr.io/kube-controller-manager:v1.20.15
k8s.gcr.io/kube-scheduler:v1.20.15
k8s.gcr.io/kube-proxy:v1.20.15
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns:1.7.0
```

使用阿里云镜像地址下载镜像：`registry.aliyuncs.com/google_containers`，执行如下命令

```bash
docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.20.15
docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.20.15
docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.20.15
docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.20.15
docker pull registry.aliyuncs.com/google_containers/pause:3.2
docker pull registry.aliyuncs.com/google_containers/etcd:3.4.13-0
docker pull registry.aliyuncs.com/google_containers/coredns:1.7.0
```

接下来给镜像打标签，得到kubeadm需要的镜像

```bash
docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.20.15 k8s.gcr.io/kube-apiserver:v1.20.15
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.20.15 k8s.gcr.io/kube-controller-manager:v1.20.15
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.20.15 k8s.gcr.io/kube-scheduler:v1.20.15
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.20.15 k8s.gcr.io/kube-proxy:v1.20.15
docker tag registry.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2
docker tag registry.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0
docker tag registry.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0
```

再删除掉从阿里云下载的镜像

```bash
docker rmi registry.aliyuncs.com/google_containers/kube-apiserver:v1.20.15
docker rmi registry.aliyuncs.com/google_containers/kube-controller-manager:v1.20.15
docker rmi registry.aliyuncs.com/google_containers/kube-scheduler:v1.20.15
docker rmi registry.aliyuncs.com/google_containers/kube-proxy:v1.20.15
docker rmi registry.aliyuncs.com/google_containers/pause:3.2
docker rmi registry.aliyuncs.com/google_containers/etcd:3.4.13-0
docker rmi registry.aliyuncs.com/google_containers/coredns:1.7.0
```

## 四、集群初始化

安装过程可以参考本系列上篇文章的链接：<https://github.com/BackendDoc/kubernetes/blob/main/01.basic/kb01-build.md>。
