# 安装kube-proxy

## 1. 签发ssl证书

我们登录到证书服务器，去签发证书

创建证书目录`/opt/certs/kubeproxy`，


创建证书请求文件`/opt/certs/kubeproxy/kube-proxy-csr.json`，添加如下内容

```json
{
	"CN": "system:kube-proxy",
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


创建证书

```shell
cfssl gencert -ca=../ca.pem -ca-key=../ca-key.pem -config=../ca-config.json -profile=client kube-proxy-csr.json | cfssljson -bare kube-proxy-client
```


## 2. 安装kube-proxy


签发好证书之后，我们回到`kb21`和`kb22`两个节点，进行安装操作，具体过程如下


下载证书文件
```shell
scp root@HDSS7-200.host.com:/opt/certs/kubeproxy/kube-proxy-client.pem ./
scp root@HDSS7-200.host.com:/opt/certs/kubeproxy/kube-proxy-client-key.pem ./
```


使用命令`cd /opt/kubernetes/server/bin/conf`回到集群配置文件目录，执行以下操作

设置集群

```shell
kubectl config set-cluster myk8s \
    --certificate-authority=/opt/kubernetes/server/bin/certs/ca.pem \
    --embed-certs=true \
    --server=https://192.168.14.10:7443 \
    --kubeconfig=kube-proxy.kubeconfig
```


设置客户端证书


```shell
kubectl config set-credentials kube-proxy \
    --client-certificate=/opt/kubernetes/server/bin/certs/kube-proxy-client.pem \
    --client-key=/opt/kubernetes/server/bin/certs/kube-proxy-client-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig
```


设置上下文

```shell
kubectl config set-context myk8s-context \
    --cluster=myk8s \
    --user=kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig
```




使用上下文


以上配置只需在一台主机进行，另一台主机从创建的主机拷贝即可，最后两台主机都执行以下命令

```shell
kubectl config use-context myk8s-context --kubeconfig=kube-proxy.kubeconfig
```



## 3. 启动服务

先创建一个脚本文件`ipvs.sh`，添加以下内容

```shell
#!/bin/bash
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
for i in $(ls $ipvs_mods_dir|grep -o "^[^.]*")
do
	/sbin/modinfo -F filename $i &>/dev/null
	if [ $? -eq 0 ]; then
		/sbin/modprobe $i
	fi
done
```

该脚本的作用是加载所有与ipvs相关的模块

我们可以使用以下命令查看当前ip_vs相关的模块，可以发现在执行脚本之前是没有返回内容的
```shell
lsmod | grep ip_vs
```

在两台主机执行以上脚本之后，我们创建`kube-proxy`的启动脚本文件`/opt/kubernetes/server/bin/kube-proxy.sh`

```shell
#!/bin/bash
./kube-proxy \
  --cluster-cidr 172.7.0.0/16 \
  --hostname-override kb21 \
  --proxy-mode=ipvs \
  --ipvs-scheduler=nq \
  --kubeconfig ./conf/kube-proxy.kubeconfig
```

添加可执行权限

```shell
chmod +x kube-proxy.sh
```

创建supervisor的配置文件`/etc/supervisord.d/kube-proxy.ini`文件，添加以下内容

```shell
[program:kube-proxy-21]
directory=/opt/kubernetes/server/bin
command=/opt/kubernetes/server/bin/kube-proxy.sh
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
stdout_logfile=/data/logs/kubernetes/kube-proxy/kube-proxy.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_event_enabled=false
```

创建日志目录与更新supervisor

```shell
mkdir -p /data/logs/kubernetes/kube-proxy/
supervisorctl update
```


我们还可以安装`ipvsadm`用于管理ipvs，如下命令

```shell
yum install -y ipvsadm
```




## 4. 集群的验证

我们创建一个DaemonSet类型的资源，添加`nginx-ds.yaml`文件，添加以下内容

```shell
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ds
spec:
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

执行资源创建命令

```shell
kubectl create -f nginx-ds.yaml
```

使用以下命令验证pod是否正常运行

```shell
kubectl get pod -o wide
```




