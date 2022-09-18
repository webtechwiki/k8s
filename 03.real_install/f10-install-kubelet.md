# 安装kubectl


## 1. 签发证书

kubectl作为客户端同时作为服务器，我们登录到`kb200`这台主机，下面将为`kubelet`创建ssl证书


创建证书信息文件`/opt/certs/kubelet/kubelet-scr.json`文件，添加以下内容

```json
{
	"CN": "k8s-kubelet",
	"hosts": [
		"127.0.0.1",
		"192.168.14.10",
		"192.168.14.11",
		"192.168.14.12",
		"192.168.14.21",
		"192.168.14.22",
		"192.168.14.200"
	],
	"key": {
		"algo": "rsa",
		"size": 2048
	},
	"name": [
		{
			"C": "CN",
			"ST": "Guangdong",
			"L": "Guangzhou",
			"O": "od",
			"OU": "ops"
		}
	]
}
```

使用以下命令创建证书

```shell
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=server kubelet-scr.json | cfssljson -bare kubelet
```



## 2. 创建kubelet配置文件

接下来我们回到`kb21`和`kb22`这两台主机，需要在这两台主机上安装kubelet


先从证书服务下载证书文件，如下命令

```shell
cd /opt/kubernetes/server/bin/certs
scp root@kb200.host.com:/opt/certs/kubelet/kubelet-key.pem ./
scp root@kb200.host.com:/opt/certs/kubelet/kubelet.pem ./
```


先使用`cd /opt/kubernetes/server/bin/conf`进去配置文件目录


执行以下配置


设置集群，包含apiserver的控制ip，以及ca公钥

```shell
kubectl config set-cluster myk8s \
    --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
    --embed-certs=true \
    --server=https://192.168.9.190:7443 \
    --kubeconfig=kubelet.kubeconfig
```


设置客户端证书


```shell
kubectl config set-credentials k8s-node \
    --client-certificate=/opt/kubernetes/server/bin/certs/client.pem \
    --client-key=/opt/kubernetes/server/bin/certs/client-key.pem \
    --embed-certs=true \
    --kubeconfig=kubelet.kubeconfig
```


设置上下文

```shell
kubectl config set-context myk8s-context \
    --cluster=myk8s \
    --user=k8s-node \
    --kubeconfig=kubelet.kubeconfig
```



使用上下文

```shell
kubectl config use-context myk8s-context --kubeconfig=kubelet.kubeconfig
```


以上的操作是生成配置文件，我们只需要在一台主机上操作，再复制到其他主机即可。


## 3. 创建一个集群角色绑定的资源

创建一个资源清单文件`k8s-node.yaml`，定义集群角色绑定的资源

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-node
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: k8s-node
```


该资源定义了一个用户`k8s-node`，该用户拥有运算节点的权限

```shell
kubectl create -f k8s-node.yaml
```


使用以下命令查看创建的资源
```shell
kubectl get clusterrolebinding k8s-node
```


活着查看更详细的资源信息
```shell
kubectl get clusterrolebinding k8s-node -o yaml
```


以上操作我们只需要在一个主机执行即可，因为操作记录会自动同步到etcd数据库



## 4. 配置kubelet启动脚本


接下来在`kb21`和`kb22`上启动kubelet，在让kubelet启动之前，我们需要有一个基础的pause镜像，以下是拉取命令，该镜像负责其k8s集群中pod启动之前的初始化操作

```shell
docker pull kubernetes/pause
```


创建kubelet的启动脚本文件`/opt/kubernetes/server/bin/kubelet.sh`文件，添加以下内容

```shell
#!/bin/bash
./kubelet \
  --anonymous-auth=false \
  --cgroup-driver systemd \
  --cluster-dns 192.168.0.2 \
  --node-ip 192.168.14.21 \
  --cluster-domain cluster.local \
  --runtime-cgroups=/systemd/system.slice \
  --kubelet-cgroups=/systemd/system.slice \
  --fail-swap-on="false" \
  --client-ca-file ./certs/ca.pem \
  --tls-cert-file ./certs/kubelet.pem \
  --tls-private-key-file ./certs/kubelet-key.pem \
  --hostname-override kb21 \
  --image-gc-high-threshold 20 \
  --image-gc-low-threshold 10 \
  --kubeconfig ./conf/kubelet.kubeconfig \
  --log-dir /data/logs/kubenetes/kube-kubelet \
  --pod-infra-container-image kubernetes/pause:latest \
  --root-dir /data/kubelet
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
# 创建日志目录
mkdir -p /data/logs/kubernetes/kube-kubelet
# 创建数据目录
mkdir -p /data/kubelet
```


创建supervisor进程配置文件`/etc/supervisord.d/kube-kubelet.ini`文件，添加以下内容

```shell
[program:kube-kubelet-21]
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







