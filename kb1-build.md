# 学习kubernetes，从快速搭建k8s集群开始

> 本系列文章，我们将在Ubuntu Server 18.04上搭建k8s环境进行入门学习。为了使用原生的Ubuntu Server 18.04，我们将使用multipass来创建多台Ubuntu Server 18.04虚拟环境。也就是说，如果你想完整参考本系列博客学习，你电脑上应当安装并能正常运行multipass，如果你想了解multipass基本操作，可以参考我写的另一篇博客：[https://blog.jkdev.cn/index.php/archives/327/](https://blog.jkdev.cn/index.php/archives/327/)。
>
> 本文演示k8s集群搭建步骤，并不涉及k8s基础知识，你可能对文章的一些专业词语感到默认，但没有关系，我们在后面会循序渐进地介绍k8s知识。

本次我们将部署一个主节点（master1）和两个工作节点（worker1、worker2）的集群。为了节省电脑资源，master1、worker1、worker2每个节点分配2个cpu、2G内存、10G硬盘。这是k8s要求的最低配置，但这些配置完全足够我们用以学习。相关操作都会在root用户之下。



## 一、准备环境

### 1. 创建Ubuntu Server 18.04虚拟机
分别创建2核cpu、10G硬盘、2G内存，名为master1、worker1、worker2三台虚拟机
```shell
multipass launch -c 1 -d 10G -m 1G -n master1 18.04
multipass launch -c 1 -d 10G -m 1G -n worker1 18.04
multipass launch -c 1 -d 10G -m 1G -n worker2 18.04
```
使用```multipass list```查询创建的虚拟机列表，如下
```shell
pan@pandeMacBook-Pro ~ % multipass list
Name                    State             IPv4             Image
master1                 Running           192.168.64.8     Ubuntu 18.04 LTS
worker1                 Running           192.168.64.11    Ubuntu 18.04 LTS
worker2                 Running           192.168.64.12    Ubuntu 18.04 LTS
````
创建虚拟机之后，我们需要对虚拟机进行初始化操作，以下的操作需要在所有主机上进行。

### 2. 修改root用户密码
为了方便使用root用户，我们对每一台虚拟机进行密码修改操作。以master1为例，如下代码
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

下面的很多安装过程，我们大多使用阿里云镜像进行安装。

### 1. 安装与配置docker
安装docker
```shell
# 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# 写入软件源信息
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# 更新软件库
apt-get -y update
# 安装程序
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
# 下载 gpg 密钥
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
# 添加 k8s 镜像源
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

### 3. 集群镜像准备
使用```kubeadm config images list```命令查看当前集群所需要的镜像，镜像版本会根据kubeadm版本而定，返回如下内容
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
kubeadm init --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16  --ignore-preflight-errors=Swap
```

参数说明

* ```--service-cidr```：k8s中scv网络的网络段
* ```--pod-network-cidr```：k8s中pod使用的网络段
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

### 2. 配置集群网络

根据初始化结果提示，我们需要安装网络插件。本次我们使用flannel作为集群的网络插件，将flannel配置文件从互联网保存到master1，文件地址为：[https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml](https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml)，在master1上将文件命名为kube-flannel.yml，后执行以下命令

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
### 3. 让Linux普通用户能操作集群

根据初始化结果提示，为了让master1上的Linux普通用户正常操作集群，我们输入exit按回车切换回普通用户后执行以下命令

```shell
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 4. 初始化集群工作节点
同时，初始化集群管理节点master1之后，我们需要将工作节点worker1和wroker2加入到集群中。将工作节点的初始化命令拷贝到worker1和worker2，使集群的工作节点（worker1、worker2）和工作节点（master1）关联起来，如下命令

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

需要注意的是，如果你的列表中显示的所有pod并不是处于Running状态，你需要等待一段时间。而且，你在安装集群的过程中，最好处于一个优质的网络环境。

至此，集群搭建完毕。从下一篇文章开始我们将继续介绍k8s基础知识



> 本文原创首发自wx订阅号：**极客开发者up**，禁止转载