# 使用kubeadm快速搭建k8s集群

## 一、概述

### 1.1 使用multipass搭建虚拟环境

我们可以使用`multipass`来创建多台`Ubuntu Server`虚拟环境，作为Linux集群的环境。如果你不知道`multipass`这个虚拟化工具的使用方法，可以参考另一篇文章：<https://blog.jkdev.cn/index.php/archives/326/>。

### 1.2 使用vagtrant搭建虚拟环境

除了multipass，我们还可以使用`vagtrant`来创建Linux虚拟环境，`vagrant`的基本操作可以参考这篇文章：[https://blog.jkdev.cn/index.php/archives/335/](https://blog.jkdev.cn/index.php/archives/335/)。

### 1.3 使用VirtualBox搭建虚拟环境

如果你不想用以上的两款工具，可以直接使用VirtualBox来创建你的虚拟环境，但直接使用VirtualBox操作会更加繁杂，所以更推荐使用multipass或者vagrant。

### 1.4 目标

本次我们将部署一个主（master、管理、控制）节点master1和两个工作（worker）节点worker1、worker2的集群。master1、worker1、worker2每个节点分配2个cpu、2G内存、10G硬盘。这是k8s要求的最低配置，但这些配置完全足够我们用以学习。

我们将使用multipass来搭建集群环境，下文是在虚拟环境中使用kubeadm搭建k8s集群搭建过程，在后续的文章中也会循序渐进地介绍k8s。

## 二、准备环境

### 2.1 创建Ubuntu18.04虚拟机

在撰写此文的时候，`ubuntu18.04`正处于其稳定的生命周期内，本次我们选择`ubuntu18.04`作为示例。分别创建2核cpu、10G硬盘、2G内存，名为master1、worker1、worker2三台虚拟机，如下命令

```bash
multipass launch -c 2 -d 10G -m 2G -n master1 18.04
multipass launch -c 2 -d 10G -m 2G -n worker1 18.04
multipass launch -c 2 -d 10G -m 2G -n worker2 18.04
```

使用`multipass list`列出已创建的虚拟机列表，如下

```bash
pan@pandeMacBook-Pro ~ % multipass list
Name                    State             IPv4             Image
master1                 Running           192.168.64.4     Ubuntu 18.04 LTS
worker1                 Running           192.168.64.5     Ubuntu 18.04 LTS
worker2                 Running           192.168.64.6     Ubuntu 18.04 LTS
```

创建虚拟机之后，我们需要记住IP与虚拟机的对应关系，再对虚拟机进行初始化操作，以下2.2、2.3的操作过程需要在所有主机上进行。

### 2.2 修改root用户密码

因为后面的操作都是使用`root`用户进行，下面我们先修改每一台主机的`root`用户密码，以master1为例，如下操作命令

```bash
# 进入虚拟机
multipass bash master1
# 修改root密码
sudo passwd root
# 修改密码之后，使用su命令切换root用户
su
```

### 2.3 关闭防火墙和iptables

根据官方文档，防火墙和iptables可能会影响到k8s集群，所以我们都要关闭掉，如下命令

```bash
# 关闭防火墙
ufw disable
# 关闭iptables
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -F
```

## 二、安装docker与kubeadm

以下3.1、3.2、3.3的操作过程需要在所有主机上进行。

### 3.1 安装与配置docker

安装docker

```bash
# 安装阿里云docker镜像服务的GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# 写入阿里云docker镜像服务软件源
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# 更新软件库
apt-get update
# 查看可以安装的docke-ce版本
apt-cache madison docker-ce
# 安装docker
apt-get -y install docker-ce=5:20.10.17~3-0~debian-bullseye
# 固定docker-ce版本
apt-mark hold docker-ce
```

每个人都可以使用阿里云创建自己的镜像加速仓库，比如`https://g6ogy192.mirror.aliyuncs.com`就是我的个人镜像加速地址，但是你也可以使用，设置镜像仓库地址如下命令

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

### 3.2 安装kubeadm、kubelet、kubectl

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
apt-get install -y kubelet=1.18.0-00 kubeadm=1.18.0-00 kubectl=1.18.0-00
# 固定版本
apt-mark hold kubelet kubeadm kubectl
```

> 需要注意的是：安装的`docker`版本要和`k8s`版本匹配，否则`k8s`可能出现无法驱动`docker`的情况，以上安装的`docker19.03`和`k8s1.18.0`是存在对应关系的，你可以通过查阅k8s官方文档的方式查看更多的对应关系。

### 3.3 集群镜像准备

使用`kubeadm config images list`命令查看当前集群基础服务所需要的镜像，镜像版本会根据当前的系统环境（当前安装kubeadm的版本）而定，返回如下内容

```bash
k8s.gcr.io/kube-apiserver:v1.18.20
k8s.gcr.io/kube-controller-manager:v1.18.20
k8s.gcr.io/kube-scheduler:v1.18.20
k8s.gcr.io/kube-proxy:v1.18.20
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.7
```

但是由于国内正常访问不到`k8s.cgr.io`，可以替换阿里云镜像地址：`registry.aliyuncs.com/google_containers`，执行如下命令

```bash
docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.18.20
docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.18.20
docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.18.20
docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.18.20
docker pull registry.aliyuncs.com/google_containers/pause:3.2
docker pull registry.aliyuncs.com/google_containers/etcd:3.4.3-0
docker pull registry.aliyuncs.com/google_containers/coredns:1.6.7
```

接下来给镜像打标签，得到kubeadm需要的镜像

```bash
docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.18.20 k8s.gcr.io/kube-apiserver:v1.18.20
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.18.20 k8s.gcr.io/kube-controller-manager:v1.18.20
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.18.20 k8s.gcr.io/kube-scheduler:v1.18.20
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.18.20 k8s.gcr.io/kube-proxy:v1.18.20
docker tag registry.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2
docker tag registry.aliyuncs.com/google_containers/etcd:3.4.3-0 k8s.gcr.io/etcd:3.4.3-0
docker tag registry.aliyuncs.com/google_containers/coredns:1.6.7 k8s.gcr.io/coredns:1.6.7
```

再删除掉从阿里云下载的镜像

```bash
docker rmi registry.aliyuncs.com/google_containers/kube-apiserver:v1.18.20
docker rmi registry.aliyuncs.com/google_containers/kube-controller-manager:v1.18.20
docker rmi registry.aliyuncs.com/google_containers/kube-scheduler:v1.18.20
docker rmi registry.aliyuncs.com/google_containers/kube-proxy:v1.18.20
docker rmi registry.aliyuncs.com/google_containers/pause:3.2
docker rmi registry.aliyuncs.com/google_containers/etcd:3.4.3-0
docker rmi registry.aliyuncs.com/google_containers/coredns:1.6.7
```

## 三、k8s集群初始化

### 3.1 初始化master节点

在master1节点上执行初始化命令

```bash
kubeadm init --apiserver-advertise-address=192.168.64.4 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16  --ignore-preflight-errors=Swap
```

参数说明

* `--apiserver-advertise-address`：指定apiserver的服务IP，此处填写master1的IP地址，如果系统包含多个IP，必须选择外网IP（能与其他节点进行网络通信的IP）
* `--service-cidr`：k8s中service网络使用的网段
* `--pod-network-cidr`：k8s中pod网络使用的网段
* `--ignore-preflight-errors`：忽略swap报错

初始化结果如下

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.64.4:6443 --token wkgukj.up6d3a8rng4ytyl0 \
    --discovery-token-ca-cert-hash sha256:d36346a3820781efd88ae3796a58e2fa139f411f9ddc9c1dc7c9c2add8da5cfb
```

如果出现类似以上的提示信息，代表master节点初始化成功。以上信息提示我们要完成三个操作，并给出了相关提示：一是拷贝配置文件、二是部署pod的网络插件、三是将工作节点加入到集群中。

### 3.2 拷贝配置文件

根据初始化结果提示，我们执行以下命令，拷贝k8s的相关配置文件到用户的家目录，才能使master1上当前登录的用户正常操作集群

```bash
# 创建配置用户的k8s文件目录
mkdir -p $HOME/.kube
# 拷贝配置和修改所有者，我们本身使用root用户操作，不需要加sudo前缀
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3.3 配置集群网络

在拷贝好k8s的配置文件到用户家目录之后，我们需要安装网络插件。本次我们使用`flannel`作为集群的网络插件，将`flannel`配置文件从github下载保存到master1，文件地址为：
[https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml](https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml)，在master1上将文件命名为`kube-flannel.yml`。

需要注意的是，如果我们的（虚拟）主机里存在多个IP，我们需要指定有效IP的网卡（如我们在vagrant里创建的虚拟机，第二个网卡才是有效IP，需要指定有效IP的网卡）。修改flannel资源清单文件，在`- --kube-subnet-mgr`后面加一条参数配置：`- --iface=网卡名`，如下

```YML
# ...
containers:
      - name: kube-flannel
       #image: flannelcni/flannel:v0.16.3 for ppc64le and mips64le (dockerhub limitations may apply)
        image: rancher/mirrored-flannelcni-flannel:v0.16.3
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=enp0s8
# ...
```

后执行以下命令创建k8s网络插件

```bash
kubectl apply -f kube-flannel.yml
```

如看到包含`created`的如下类似信息，证明网络插件安装成功

```bash
namespace/kube-flannel created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

### 3.4 初始化集群工作节点

初始化集群管理节点master1之后，我们可以将工作节点worker1和wroker2加入到集群中。将工作节点的初始化命令拷贝到worker1和worker2去执行，使集群的工作节点（worker1、worker2）和管理节点（master1）关联起来，如下命令

```bash
kubeadm join 192.168.64.4:6443 --token wkgukj.up6d3a8rng4ytyl0 \
    --discovery-token-ca-cert-hash sha256:d36346a3820781efd88ae3796a58e2fa139f411f9ddc9c1dc7c9c2add8da5cfb
```

在工作节点执行初始化命令之后，我们在管理节点master1使用`kubectl get pods --all-namespaces`查看结果，可以看到类似如下信息

```bash
root@master1:/home/ubuntu# kubectl get pods --all-namespaces
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
kube-system   coredns-66bff467f8-pw9br          1/1     Running   0          13m
kube-system   coredns-66bff467f8-wsj45          1/1     Running   0          13m
kube-system   etcd-master1                      1/1     Running   0          14m
kube-system   kube-apiserver-master1            1/1     Running   0          14m
kube-system   kube-controller-manager-master1   1/1     Running   0          14m
kube-system   kube-flannel-ds-c4jnh             1/1     Running   0          3m39s
kube-system   kube-flannel-ds-rg58c             1/1     Running   0          3m14s
kube-system   kube-flannel-ds-sw85v             1/1     Running   0          3m15s
kube-system   kube-proxy-ddk88                  1/1     Running   0          3m15s
kube-system   kube-proxy-dt825                  1/1     Running   0          13m
kube-system   kube-proxy-jgm4h                  1/1     Running   0          3m14s
kube-system   kube-scheduler-master1            1/1     Running   0          14m
```

如果需要查看更多的信息（如所运行的宿主机），只需在原命令中追加`-o wide`参数即可。

需要注意的是，如果你的列表中显示的所有pod并不是处于Running状态，一般是正在从网络上下载资源，还没有完全启动服务，你需要等待一段时间。你在安装集群的过程中，最好处于一个优质的网络环境。如果你电脑配置允许，你还可以使用上文说明的操作步骤，添加更多的工作节点。

### 3.5 重置k8s集群

如果你在搭建过程中出现问题，如因为网络原因安装失败等，或者因为其他原因想重置k8s集群，可以使用以下的操作步骤

```bash
# 驱离工作节点的pod
kubectl drain worker1 --delete-local-data --force --ignore-daemonsets
kubectl drain worker2 --delete-local-data --force --ignore-daemonsets
# 删除工作节点
kubectl delete node worker1
kubectl delete node worker2
# 重置集群
kubeadm reset
```

## 四、总结

以上是使用kubeadm快速搭建k8s集群的完整说明，或许你的安装会失败。原因可能是你的虚拟环境问题、也可能因为网络原因，或者是你操作错误。如果出现失败，你需要根据你的操作步骤认真分析并重试。

> 作者wx订阅号：**极客开发者**，禁止转载
