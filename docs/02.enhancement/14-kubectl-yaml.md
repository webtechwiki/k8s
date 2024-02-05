# kubectl陈述式资源管理方法

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


**删除资源**

使用资源配置清单删除资源的命令如下

```shell
kubectl delete -f nginx-ds-svc.yaml
```


## 2.声明式资源管理方法小结

- 声明式资源管理方法，依赖于统一资源配置清单文件对资源进行管理

- 对资源的管理，是通过事先定义在统一资源配置清单内，再通过陈述式命令应用到k8s集群里


- 资源配置清单的学习方法

 1. 多看官方（被人）写的资源配置清单，能读懂
 2. 能照着现成的资源配置清单进行修改
 3. 遇到不懂的，善于利用`kubectl explain`查询注释
 4. 初学者切记上来就无中生有，自己憋着写







