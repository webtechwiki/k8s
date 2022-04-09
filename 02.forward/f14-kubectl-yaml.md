# 使用yaml资源管理清单管理k8s

我们把使用`资源清单`管理k8s的方法叫做声明式资源管理方法


## 1. 声明式管理方法

**查看资源配置清单**

查看正在运行的名称为`nginx-dp`的service的资源配置清单

```shell
kubectl get svc nginx-dp -o yaml -n kube-public
```


**解释资源配置清单**


查看service的资源配置清单解释

```shell
kubectl explain service
```


**应用资源配置清单**

在对资源配置订单做好定义之后，使用一下命令应用资源配置清单

```shell
kubectl apply -f nginx-ds-svc.yaml
```