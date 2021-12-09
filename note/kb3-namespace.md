# namespace

> 假如需要准备两套k8s集群用于开发测试和预发布环境，但是由于项目组可用主机资源有限，没有那么多主机可用，不能满足k8s集群的要求。我们可以使用k8s集群中的命名空间即可实现开发测试和预发布环境的隔离。

本期目标
* 了解namespace 的作用
* 掌握namespace查看方法
* 掌握namespace创建方法
* 掌握namespace删除方法

## 一、相关命令

### 1. 查看命名空间
```shell
kubectl get namespace
# 或者
kubectl get ns
```
返回内容如下：
```
NAME              STATUS   AGE
default           Active   14h
kube-node-lease   Active   14h
kube-public       Active   14h
kube-system       Active   14h
```
说明
* default： 用户创建的pod默认在此命名空间
* kube-public：所有用户均可访问，包括未认证用户
* kube-node-lease：kubernetes集群节点租约状态在此使用，v1.13加入
* kube-system：kubernetes集群所有组件在此运行

根据命名空间查看pod
```
kubectl get pods --namespace kube-system
```

### 2. 创建命名空间
* 通过命令创建命名空间
```
kubectl create namespace test
```

* 通过资源清单文件创建命名空间
 创建清单文件
```shell
touch 01-create-ns.yaml
```
添加以下内容
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test2
```
应用资源
```
kubectl apply -f 01-create-ns.yaml
```

### 3.删除命名空间
通过命名删除命名空间
```shell
# 删除test1命名空间
kubectl delete namespace test1
```
通过资源清单文件删除，定义内容和创建内容一致，如2中定义了test2的namespace，执行以下命令进行删除
```shell
kubectl delete -f 01-create-ns.yaml
```