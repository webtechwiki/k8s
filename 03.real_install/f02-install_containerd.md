# 安装containerd作为runtime

## 一、加载containerd所需的模块

在所有节点上加载`overlay`和`br_netfilter`，如下命令

```bash
modprobe overlay
modprobe br_netfilter
```

在所有机器上执行下面的命令，目的是系统重启时模块能自动加载

```bash
cat > /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```

## 二、安装containerd

CRI 是一个插件接口，它使 kubelet 能够使用各种容器运行时，无需重新编译集群组件。容器运行时接口（CRI）是 kubelet 和容器运行时之间通信的主要协议。我们我们安装containerd是带有`cri`的，直接解压到根目录即可完成安装，如下命令

```bash
# 解压
tar -C / -xzvf cri-containerd-cni-1.6.4-linux-amd64.tar.gz
```

解压完成，containerd 包含的各个组件将存在系统对应目录中，启动服务文件也会自动安装到`/etc/systemd/system/containerd.service`。

## 三、创建containerd的配置文件

创建默认配置文件

```bash
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

启用SystemdCgroup

```bash
# 修改
sed -i "s#SystemdCgroup\ \=\ false#SystemdCgroup\ \=\ true#g" /etc/containerd/config.toml
# 查看验证
cat /etc/containerd/config.toml | grep SystemdCgroup
```

修改容器基础镜像地址为阿里云

```bash
# 修改
sed -i "s#k8s.gcr.io#registry.aliyuncs.com/google_containers#g" /etc/containerd/config.toml
# 查看验证
cat /etc/containerd/config.toml | grep sandbox_image
```

## 四、启动containerd并设置开机自启

```bash
systemctl daemon-reload
systemctl enable --now containerd
```

## 五、配置crictl客户端连接的运行时位置

crictl 是 CRI 兼容的容器运行时命令行接口即cri的客户端，我们可以使用它来检查和调试 Kubernetes 节点上的容器运行时和应用程序。需要在三台主机上安装，以下是安装过程

```bash
# 解压
tar -zxvf crictl-v1.24.2-linux-amd64.tar.gz -C /usr/bin/

# 生成配置文件
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

# 测试
systemctl restart  containerd
crictl info
```

## 六、安装nerdctl

Nerdctl是一个相对较新的containerd命令行客户端。nerdctl的目标是用户友好和docker兼容。在某种程度上，nerdctl + containerd可以无缝地替代docker + dockerd。nerdctl的目标是促进试验Docker中没有的最前沿的容器特性。需要在三台主机上安装，直接解压到对应目录即可完成安装

```bash
tar -zxvf nerdctl-0.20.0-linux-amd64.tar.gz -C /usr/bin/ nerdctl
```

## 七、安装nerdctl所需要的cni插件

在三台主机上安装，解压到对应目录即可完成安装，如下命令

```bash
tar -zxvf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin/
```

修改`/etc/profile`，在第如下两行指令，执行如下命令

```bash
# 追加配置
cat >> /etc/profile <<EOF
source <(nerdctl completion bash)
export CONTAINERD_NAMESPACE=k8s.io
EOF

# 让配置立即生效
source /etc/profile
```
