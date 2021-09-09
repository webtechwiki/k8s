# pod相关操作

> k8s 是不能直接运行程序的，k8s集群中最小的调度单元为pod，Pod是容器的封装。因此我们需要使用Pod来运行应用程序

本期目标
* 了解Pod概念
* 查看Pod
* 创建Pod
* Pod访问
* 删除Pod

## 一、相关操作
### 1. 查看Pod
默认查询default命名空间中的Pod
```
kubectl get pod
# 或
kubectl get pods
```
查看指定命名空间的Pod
```
kubectl get pods --namespace default
# 或
kubectl get pods -n default
```
查看所有命名空间的Pod
```
kubectl get pods --all-namespaces
```

### 2. 创建Pod
编写用于创建Pod的资源清单文件02-create-pod.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: nginx-pod
    image: nginx:latest
    imagePullPolicy: IfNotPresent
    ports:
    - name: nginxport
      containerPort: 80
```
执行应用命令，将会在默认命名空间创建Pod
```shell
kubectl apply -f 02-create-pod.yaml
```
要查看Pod在哪个节点上运行，可以使用以下命令
```shell
kubectl get pods -o wide
```
本次创建了nginx，所以可以使用访问Pod的IP进行验证。
```
curl http://10.244.1.2
```

进入Pod中
```
# 进入bash
kubectl exec -it POD名 -- bash
# 退出 bash
exit
```

### 3.  删除Pod
使用命令删除Pod
```shell
# 默认删除default命名空间下的Pod
kubectl delete pods pod1
# 或指定命名空间删除
kubectl delete pods pod1 -n default
```

使用资源清单执行删除，如在2中创建的Pod，可以使用以下命令删除
```
kubectl delete -f 02-create-pod.yaml
```