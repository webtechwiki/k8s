# 安装Containerd作为Runtime

## 一、安装cni插件

```bash
# 下载安装包
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz

# 创建cni插件所需目录
mkdir -p /etc/cni/net.d /opt/cni/bin
# 解压cni二进制包
tar xf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin/
```

## 二、安装continerd

### 2.1 安装containerd

```bash
wget https://github.com/containerd/containerd/releases/download/v1.6.4/cri-containerd-cni-1.6.4-linux-amd64.tar.gz

# 解压
tar -C / -xzvf cri-containerd-cni-1.6.4-linux-amd64.tar.gz
```

此时启动服务文件已经自动安装到`/etc/systemd/system/containerd.service`

### 2.2 配置Containerd所需的模块

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

加载模块

```bash
systemctl restart systemd-modules-load.service
```

### 2.3 创建Containerd的配置文件

创建默认配置文件

```bash
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
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

### 2.4 启动containerd并设置开机自启

```bash
systemctl daemon-reload
systemctl enable --now containerd
```

### 2.5 配置crictl客户端连接的运行时位置

```bash
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.24.2/crictl-v1.24.2-linux-amd64.tar.gz

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
