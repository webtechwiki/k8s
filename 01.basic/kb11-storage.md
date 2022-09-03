# 存储

## 1. 概述

> 容器的生命周期可能很短，会被频繁地创建和销毁。那么容器销毁时，保存在容器中的数据也会被清除。这种结果对用户来说，在某种情况下是不乐意的，为了持久化保存容器的数据，kubernates引入volume的概念。

Volume是Pod中能够被多个容器访问的共享目录，它被定义在Pod上，然后被一个Pod里的多个容器挂载到具体的文件目录下，kubernates通过Volume实现同一个Pod中不同容器之间的数据共享以及数据持久化存储。Volume的生命周期不与容器的生命周期相关，当容器终止或者重启时，Volume中的数据也不会丢失。kubernates的Volume支持多种类型，比较常见的有以下几个：

* 简单存储：EmptyDir、HostPath、NFS
* 高级存储：PV、PVC
* 配置存储：ConfigMap、Secret

在下文中我们将介绍k8s中的简单存储功能


## 2. EmptyDir
EmptyDir是在Pod被分配到Node时创建的，它的初始化内容为空，并且无需指定宿主机上对应的目录文件，因为kubernetes会自动分配一个目录，到Pod销毁时，EmptyDir中的数据也会被永久删除。Empty用途如下：
* 临时空间，例如用于某些应用程序运行时所需的临时目录，且无需永久保留
* 一个容器需要从另一个容器中获取数据的目录（多容器共享）


接下来，通过一个容器之间文件共享的案例来使用一下EmptyDir。在一个Pod中准备两个容器，nginx和busybox，然后声明一个Volume分别挂载到两个容器的目录中，再让nginx容器负责向Volume中写入日志，busybox中通过命令将日志内容读取到控制台。创建一个`volume-empty.yaml`资源清单文件，添加以下内容
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-emptydir
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"] # 初始命令，动态读取文件中内容
    volumeMounts: # 将logs-volume挂载到busybox容器中，对应目录为logs
    - name: logs-volume
      mountPath: /logs
  volumes: # 声明volume，name为logs-volume，类型为emptyDir
  - name: logs-volume
    emptyDir: {}
```

创建成功之后，使用以下命令查看日志
```shell
kubectl logs -f volume-emptydir -n dev -c busybox
```
当访问nginx时，将看到相应 的输出数据

## 3. HostPath
EmptyDir中数据不会被持久化，它会随着Pod的结束而销毁，如果想简单的将数据持久化到主机中，可以选择HostPath。
HostPath就是将Node主机中一个实际目录挂载在Pod中，以提供容器使用，这样的设计就可以保证Pod销毁了，但是数据依旧可以存在Node主机上。创建一个`volume-hostpath.yaml`资源清单文件，添加以下内容：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-hostpath
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"]
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
  volumes:
  - name: logs-volume
    hostPath:
      path: /root/logs
      type: DirectoryOrCreate # 目录存在就使用，不存在先创建后使用
```

hostPath类型type类型
* **DirectoryOrCreate**：目录存在就使用，不存在就先创建，后使用
* **Directory**：目录必须存在
* **FileOrCreate**：文件存在就使用，不存在就先创建后使用
* **File**：文件必须存在
* **Socket**：unix套接字必须存在
* **CharDevice**：字符设备必须存在
* **BlockDevice**：块设备必须存在

使用以下命令创建

```shell
kubectl create -f volume-hostpath.yaml
```

## 4. NFS
HostPath可以解决数据持久化的问题，但是一旦Node节点故障了，Pod如果转到了别的节点，又会出问题了，此时需要准备单独的网络存储系统，比如常见的NFS、CIFS。
NFS是一个网络文件存储系统，可以搭建一台NFS服务器，然后将Pod中的存储直接连接到NFS系统上，这样的话，无论Pod在节点上怎么转移，只要Node跟NFS对接没问题，数据就可以成功访问。
（1）首先准备NFS的服务器，这里为了简单，直接在master节点作NFS服务器
```shell
# 安装nfs服务
apt install nfs-kernel-server
# 准备一个共享目录
mkdir /var/data/nfs -pv
# 将共享目录读写权限暴露给集群中的所有主机
vim /etc/exports
```
在multipass虚拟环境中，集群都在一个网段内，添加以下内容
```shell
# 以读写和非root权限暴露
/var/data/nfs 192.168.64.0/24(rw,no_root_squash)
```
重启nfs服务
```shell
# 启动
systemctl start nfs-kernel-server
```
（2）在node节点安装nfs客户端
```shell
apt install nfs-common
```

创建NFS存储卷，创建`volume-nfs.yaml`文件，添加以下内容
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-hostpath
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"]
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
  volumes:
  - name: logs-volume
    nfs:
      server: 192.168.64.8 # nfs服务器
      path: /var/data/nfs # 共享文件路径
```
当我们访问pod中的nginx之后，将看到master节点的`/var/data/nfs`目录下产生了相关日志文件


