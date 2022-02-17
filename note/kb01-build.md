# 学习kubernetes，从快速搭建k8s集群开始

> 我们将使用multipass来创建多台Ubuntu Server 18.04虚拟环境。如果你不熟悉multipass的基本操作，可以参考另一篇文章：[https://blog.jkdev.cn/index.php/archives/327/](https://blog.jkdev.cn/index.php/archives/327/)。

本次我们将部署一个主节点（master1）和两个工作节点（worker1、worker2）的集群。为了节省电脑资源，master1、worker1、worker2每个节点分配2个cpu、2G内存、10G硬盘。这是k8s要求的最低配置，但这些配置完全足够我们用以学习，在root用户下执行相关操作。

本文主要内容是k8s集群搭建过程，并不涉及k8s的深入知识，你可能对文章的一些专业词语感到陌生，但没有关系，在后续的文章中将逐渐介绍。



## 一、准备环境

### 1. 创建Ubuntu Server 18.04虚拟机
分别创建2核cpu、10G硬盘、2G内存，名为master1、worker1、worker2三台虚拟机

```shell
multipass launch -c 2 -d 10G -m 2G -n master1 18.04
multipass launch -c 2 -d 10G -m 2G -n worker1 18.04
multipass launch -c 2 -d 10G -m 2G -n worker2 18.04
```

使用 `multipass list` 查询创建的虚拟机列表，如下

```shell
pan@pandeMacBook-Pro ~ % multipass list
Name                    State             IPv4             Image
master1                 Running           192.168.64.8     Ubuntu 18.04 LTS
worker1                 Running           192.168.64.11    Ubuntu 18.04 LTS
worker2                 Running           192.168.64.12    Ubuntu 18.04 LTS
```

创建虚拟机之后，我们需要对虚拟机进行初始化操作，以下2、3的操作需要在所有主机上进行。

### 2. 修改root用户密码

为了方便使用root用户，我们对每一台虚拟机的root账户进行密码修改操作。以master1为例，如下指令

```shell
# 进入主机
multipass shell master1
# 修改root密码，这里我都改为123456
sudo passwd root
# 修改密码之后，直接使用su命令切换 root用户
su
```

### 3. 关闭防火墙和iptables
根据官方文档，防火墙和iptables可能会影响到k8s集群，所以我们需要关闭掉
```shell
# 关闭防火墙
ufw disable
# 关闭iptables
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -F
```



## 二、安装docker与kubeadm

以下1、2、3的操作需要在所有主机上进行。

### 1. 安装与配置docker
安装docker
```shell
# 安装 docker 的阿里云镜像 GPG 证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# 写入 docker 的阿里云镜像软件源
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# 更新软件库
apt-get -y update
# 安装 docker
apt-get -y install docker-ce=5:19.03.15~3-0~ubuntu-bionic
# 固定版本
apt-mark hold docker-ce
```
设置docker阿里云加速镜像仓库
```shell
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://g6ogy192.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"] 
}
EOF
systemctl daemon-reload
systemctl restart docker
```

### 2. 安装kubeadm, kubelet, kubectl
```shell
# 安装 k8s 的阿里云镜像 GPG 证书
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
# 添加 k8s 的阿里云镜像源
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

> 需要注意的是：安装的 docker 版本要和 k8s 版本对应，以上安装的 docker 19.03 和 k8s 1.18.0 是存在对应关系的，否则 k8s 无法正常使用，你可以通过查阅官方文档的方式查看更多的对应关系。

### 3. 集群镜像准备
使用```kubeadm config images list```命令查看当前集群基础服务所需要的镜像，镜像版本会根据当前的系统环境而定，返回如下内容
```shell
k8s.gcr.io/kube-apiserver:v1.18.20
k8s.gcr.io/kube-controller-manager:v1.18.20
k8s.gcr.io/kube-scheduler:v1.18.20
k8s.gcr.io/kube-proxy:v1.18.20
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.7
```
我们使用docker拉取镜像，但是由于国内正常访问不到```k8s.cgr.io```，可以替换阿里加速镜像地址：```registry.aliyuncs.com/google_containers```，执行如下命令
```shell
docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.18.20
docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.18.20
docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.18.20
docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.18.20
docker pull registry.aliyuncs.com/google_containers/pause:3.2
docker pull registry.aliyuncs.com/google_containers/etcd:3.4.3-0
docker pull registry.aliyuncs.com/google_containers/coredns:1.6.7
```

接下来给镜像重命名，使其和原kubeadm需要的镜像名称一致
```shell
docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.18.20 k8s.gcr.io/kube-apiserver:v1.18.20
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.18.20 k8s.gcr.io/kube-controller-manager:v1.18.20
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.18.20 k8s.gcr.io/kube-scheduler:v1.18.20
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.18.20 k8s.gcr.io/kube-proxy:v1.18.20
docker tag registry.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2
docker tag registry.aliyuncs.com/google_containers/etcd:3.4.3-0 k8s.gcr.io/etcd:3.4.3-0
docker tag registry.aliyuncs.com/google_containers/coredns:1.6.7 k8s.gcr.io/coredns:1.6.7
```

再删除掉从阿里云下载的镜像

```shell
docker rmi registry.aliyuncs.com/google_containers/kube-apiserver:v1.18.20
docker rmi registry.aliyuncs.com/google_containers/kube-controller-manager:v1.18.20
docker rmi registry.aliyuncs.com/google_containers/kube-scheduler:v1.18.20
docker rmi registry.aliyuncs.com/google_containers/kube-proxy:v1.18.20
docker rmi registry.aliyuncs.com/google_containers/pause:3.2
docker rmi registry.aliyuncs.com/google_containers/etcd:3.4.3-0
docker rmi registry.aliyuncs.com/google_containers/coredns:1.6.7
```



## 三、k8s集群初始化

### 1. 初始化master节点

在master节点上执行初始化命令

```shell
kubeadm init --apiserver-advertise-address=192.168.64.8 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16  --ignore-preflight-errors=Swap
```

参数说明

* ```--apiserver-advertise-address```：指定apiserver的服务IP，如果系统包含多个IP，必须选择外网IP
* ```--service-cidr```：k8s中service网络使用的网段
* ```--pod-network-cidr```：k8s中pod网络使用的网段
* ```--ignore-preflight-errors```：忽略swap报错

初始化结果如下

```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.64.8:6443 --token mzolyd.fgbta1hw9s9yml55 \
    --discovery-token-ca-cert-hash sha256:21ffa3a184bb6ed36306b483723c37169753f9913e645dc4f88bb12afcebc9dd
```


根据以上的成功初始化提示信息，我们可以知道接下来要完成三个操作，一是拷贝配置文件；二是部署pod的网络插件；三是将工作节点加入到集群中。下文的 2、3、4 中说明操作过程。

### 2. 拷贝配置文件

根据初始化结果提示，我们执行以下命令，拷贝k8s的相关配置文件到用户的家目录，才能使master1上当前登录的用户正常操作集群

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3. 配置集群网络

在拷贝好k8s的配置文件到用户家目录之后，我们需要安装网络插件。本次我们使用flannel作为集群的网络插件，将flannel配置文件从github下载保存到master1，文件地址为：[https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml](https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml)，在master1上将文件命名为kube-flannel.yml，后执行以下命令


需要注意的是，如果我们的虚拟机包含多个IP，我们需要指定真实IP的网卡。如我们在vagrant 里创建的虚拟机，第二个网卡才是真实IP，修改flannel资源清单文件，在 `- --kube-subnet-mgr` 后面加一行参数`- --iface=网卡名`，如下：
```YML
# -------------
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
# -------------
```


```shell
kubectl apply -f kube-flannel.yml
```

如看到如下信息，即网络插件安装成功

```shell
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```


### 4. 初始化集群工作节点
初始化集群管理节点master1之后，此时我们可以将工作节点worker1和wroker2加入到集群中。将工作节点的初始化命令拷贝到worker1和worker2，使集群的工作节点（worker1、worker2）和管理节点（master1）关联起来，如下命令

```shell
kubeadm join 192.168.64.8:6443 --token mzolyd.fgbta1hw9s9yml55 \
    --discovery-token-ca-cert-hash sha256:21ffa3a184bb6ed36306b483723c37169753f9913e645dc4f88bb12afcebc9dd
```

在工作节点执行初始化命令之后，我们在管理节点使用```kubectl get pods --all-namespaces```查看结果，如下结果

```shell
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

需要注意的是，如果你的列表中显示的所有pod并不是处于Running状态，你需要等待一段时间。而且，你在安装集群的过程中，最好处于一个优质的网络环境。如果你电脑配置允许，你还可以使用上文的涉及到的操作步骤，添加更多的工作节点。



如果你在搭建过程中出现问题，或者因为其他原因想重置k8s集群，使用使用以下的步骤：
```shell
# 驱离工作节点的 pod
kubectl drain worker1 --delete-local-data --force --ignore-daemonsets
kubectl drain worker2 --delete-local-data --force --ignore-daemonsets
# 删除工作节点
kubectl delete node worker1
kubectl delete node worker2
# 重置集群
kubeadm reset
```


至此，集群搭建完毕。从下一篇文章开始我们将继续介绍k8s基础知识

> 本文原创首发自wx订阅号：**极客开发者up**，禁止转载

