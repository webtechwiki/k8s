# 安装kubectl

## 一、给指定用户授权

### 1.1 授权

为用户kubelet-bootstrap授权，允许kubelet tls bootstrap创建CSR请求。

```bash
kubectl create clusterrolebinding kubelet-bootstrap1 --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

把system:certificates.k8s.io:certificatesigningrequests:nodeclient授权给kubelet-bootstrap，目的是实现对CSR的自动审批。

```bash
kubectl create clusterrolebinding kubelet-bootstrap2 --clusterrole=system:certificates.k8s.io:certificatesigningrequests:nodeclient --user=kubelet-bootstrap
```

这个用户名是在配置apiserver时用到的token文件`/etc/kubernetes/bb.csv`里指定的。

### 1.2 创建授权配置文件

```bash
# 进入证书目录
cd /etc/kubernetes/pki/

# 创建集群信息
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.26.71:6443 --kubeconfig=kubelet-bootstrap.conf

# 创建用户信息
kubectl config set-credentials kubelet-bootstrap --token=6440328e1b3a1f4873dc  --kubeconfig=kubelet-bootstrap.conf

# 设置上下文
kubectl config set-context kubernetes --cluster=kubernetes --user=kubelet-bootstrap --kubeconfig=kubelet-bootstrap.conf 

# 启用上下文
kubectl config use-context kubernetes --kubeconfig=kubelet-bootstrap.conf 

# 剪切配置文件到/etc/kubernetes
mv kubelet-bootstrap.conf  /etc/kubernetes/
```

## 二、创建kubelet配置文件



## 三、配置kubelet启动脚本

接下来在`192-debian`和`160-debian`上启动kubelet，在让kubelet启动之前，我们需要有一个基础的pause镜像，以下是拉取命令，该镜像负责其k8s集群中pod启动之前的初始化操作

创建kubelet的启动脚本文件`/opt/kubernetes/server/bin/kubelet.sh`文件，添加以下内容

```shell
#!/bin/bash
./kubelet \
    --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.conf  \
    --cert-dir=/var/lib/kubelet/pki \
    --hostname-override=vms71.rhce.cc \
    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
    --config=/etc/kubernetes/kubelet-config.yaml \
    --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.7 \
    --container-runtime=remote \
    --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
    --runtime-request-timeout=15m \
    --alsologtostderr=true \
    --logtostderr=false \
    --log-dir=/var/log/kubernetes \
    --v=2
```

创建配置文件

```bash
cat > /etc/kubernetes/kubelet-config.yaml <<EOF
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250 
readOnlyPort: 10255
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
cgroupDriver: systemd
clusterDNS:
- 192.068.0.2
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
EOF
```

添加可执行权限

```shell
chmod +x kubelet.sh
```

参数说明：

`anonymous-auth=false`: 不能使用匿名登录

`cgroup-driver systemd`: 要与docker保持一致

`cluster-dns`: 集群的dns，这里先固定写，后续会提到

`node-ip`: 节点ip，和主机ip一致

`fail-swap-on="false"`: k8s运算时最好把swap关闭掉，但我们其他程序可能需要，所以添加该选项，会兼容swap存在的场景

`client-ca-file`:根证书

`tls-cert-file`:kubelet作为服务端需要的证书

`tls-private-key-file`:kubelet作为服务端需要的证书私钥

`hostname-override`: 主机名称

`kubeconfig`: kubelet配置文件路径

`log-dir`: 日志目录

`pod-infra-container-image`: 基础的pause镜像

创建数据目录和日志目录

```shell
# 创建kubelet所需要的目录
mkdir -p /data/logs/kubernetes/kube-kubelet /var/lib/kubelet /var/log/kubernetes /etc/kubernetes/manifests/
# 创建数据目录
mkdir -p /data/kubelet
```

创建supervisor进程配置文件`/etc/supervisor/conf.d/kube-kubelet.conf`文件，添加以下内容

```ini
[program:kube-kubelet-199]
directory=/opt/kubernetes/server/bin
command=/opt/kubernetes/server/bin/kubelet.sh
numprocs=1
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/kubernetes/kube-kubelet/kubelet.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_event_enabled=false
```

给脚本添加可执行权限

```shell
chmod +x kubelet.sh
```

更新supervisord，如下命令

```shell-script
supervisorctl update
```

此时，服务已经正常运行了，可以使用以下命令查看节点信息

```shell
kubectl get nodes
```

如果看到以下信息，代表安装成功

```shell
[root@kb21 bin]# kubectl get nodes
NAME   STATUS   ROLES    AGE    VERSION
kb21   Ready    <none>   5m2s   v1.15.2
kb22   Ready    <none>   5m2s   v1.15.2
```

我们还可以设置集群的标签

```shell
# 设置集群为master标签
kubectl label node kb21 node-role.kubernetes.io/master=
kubectl label node kb22 node-role.kubernetes.io/master=
# 设置集群为node标签
kubectl label node kb21 node-role.kubernetes.io/node=
kubectl label node kb22 node-role.kubernetes.io/node=
```
