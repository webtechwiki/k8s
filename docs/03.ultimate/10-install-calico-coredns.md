# 安装calico和coreDNS

以下的操作，我们在`199-debian`节点去完成。

## 一、安装calico

```bash
cd /etc/kubernetes
wget https://docs.projectcalico.org/manifests/calico.yaml
```

修改calico.yaml文件，将`CALICO_IPV4POOL_CIDR`改为和kube-proxy的配置一样，如下

```ini
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
```

创建calico服务

```bash
kubectl apply -fcalico.yaml
```

执行命令之后，calico会拉去远端的镜像并运行，等到所有pod都处于`Running`状态代表服务启动完成，如下

```bash
root@199-debian:/etc/kubernetes# kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-6799f5f4b4-xqpf9   1/1     Running   0          3m34s
calico-node-9bt29                          1/1     Running   0          3m34s
calico-node-djxvc                          1/1     Running   0          3m34s
```

此时calico已经正常运行了，如果上节创建的nginx的pod还没有删除的话，先删除掉再创建，如下命令

```bash
kubectl delete -f nginx-pod.yaml
kubectl apply -f nginx-pod.yaml
```

创建新的pod之后，使用`kubectl get pod -o wide`查看pod所处的节点，此时我们在任意工作节点请求该IP，都能成功请求。

## 二、安装coredns

### 2.1 下载基础资源配置文件

下载corndns资源配置文件

```bash
# 下载
wget https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed
# 重命名
mv coredns.yaml.sed coredns.yaml
```

### 2.2 修改配置

做出以下修改：

大概第62行，找到配置文件中的`CLUSTER_DOMAIN`、`REVERSE_CIDRS`这两个变量改为集群域名，如下

```yml
kubernetes cluster.local in-addr.arpa ip6.arpa {
  fallthrough in-addr.arpa ip6.arpa
}
```

大概第66行，`UPSTREAMNAMESERVER`改为宿主机DNS配置`/etc/resolve.conf`，如下

```yaml
forward . /etc/resolve.conf {
  max_concurrent 1000
}
```

大概在186行，将`CLUSTER_DNS_IP`改为kubelet配置文件中指定的集群IP地址`10.96.0.10`，如下

```yml
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.96.0.10
```

### 2.3 启动服务

使用以下命令启动服务

```bash
kubectl apply -f coredns.yaml
```
