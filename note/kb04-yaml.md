# k8s声明样式资源清单（YAML）文件

目标
* 掌握yaml文件的书写格式
* 通过yaml文件实现资源清单描述方法

在k8s中，一般使用yaml格式的文件创建符合我们预期的pod，这样的yaml文件被称为资源清单

## 一、常用字段

**version**
* 字段类型：String
* 说明：k8s api的版本，目前基本上是v1，可以使用kubectl api-versions命令查看

**kind**
* 字段类型：String
* 说明：指定文件定义的资源类型和角色，比如：pod

**metadata**
* 字段类型：Object
* 说明：元数据对象，固定只就写metadata

**metadata.name**
* 字段类型：String
* 说明：元数据对象名称，由我们自定义。比如是资源类型是pod，我们在这里可以定义pod的名称

**metadata.namespace**
* 字段类型：String
* 说明：元数据对象命名空间，由我们自定义

**Spec**
* 字段类型：Object
* 说明：详细定义对象，固定值就写Spec

**spec.containers[]**
* 字段类型：List
* 说明：Spec对象的容器列表

**spec.containers[].name**
* 字段类型：String
* 说明：定义容器的名称

**spec.containers[].image**
* 字段类型：String
* 说明：定义容器的镜像名称

**spec.containers[].imagePullPolicy**
* 字段类型：String
* 说明：定义镜像拉取策略，有Always（每次都尝试拉取新镜像）、Never（仅使用本地镜像）、Ifnotpresent（本地有镜像则不拉取新镜像）三个值可选

**spec.containers[].command[]**
* 字段类型：List
* 说明：指定容器启动命令，因为是数组可以指定多个，不指定则使用镜像打包时的启动命令

**spec.containers[].args**
* 字段类型：List
* 说明：指定容器启动命令参数，因为数组可以指定多个

**spec.containers[].workDir**
* 字段类型：String
* 说明：指定容器的工作目录

**spec.containers[].volumeMounts[]**
* 字段类型：List
* 说明：指定容器内部的存储卷配置

**spec.containers[].volumeMounts[].name**
* 字段类型：String
* 说明：指定容器加载的存储卷的名称

**spec.containers[].volumeMounts[].mountPath**
* 字段类型：String
* 说明：指定容器加载的存储卷的路径

**spec.containers[].volumeMounts[].readOnly**
* 字段类型：String
* 说明：设置存储卷的读写模式，true或者false，默认为读写模式

**spec.containers[].ports[]**
* 字段类型：String
* 说明：指定容器需要用到的端口列表

**spec.containers[].ports[].name**
* 字段类型：String
* 说明：指定的端口名称

**spec.containers[].ports[].containerPort**
* 字段类型：String
* 说明：指定容器需要监听的端口号

**spec.containers[].ports[].hostPort**
* 字段类型：String
* 说明：指定容器所在主机需要监听的端口号，默认跟containerPort相同，注意：设置了hostPort同一台主机无法启动该容器的相同副本，因为主机的端口不能相同，这样会冲突

**spec.containers[].ports[].protocol**
* 字段类型：String
* 说明：指定端口协议，支持TCP和UDP，默认值为TCP

**spec.containers[].ports[].env[]**
* 字段类型：List
* 说明：指定容器运行前需设的环境变量列表

**spec.containers[].ports[].env[].name**
* 字段类型：List
* 说明：指定环境变量名称

**spec.containers[].ports[].env[].value**
* 字段类型：List
* 说明：指定环境变量值

**spec.containers[].ports[].resources**
* 字段类型：Object
* 说明：指定资源限制和资源请求的值（这里开始就是设置资源的上限）

**spec.containers[].ports[].resources.limits**
* 字段类型：Object
* 说明：指定容器运行时资源的运行上限

**spec.containers[].ports[].resources.limits.cpu**
* 字段类型：String
* 说明：指定CPU限制，单位为core数，将用于docker run -- cpu-shares参数

**spec.containers[].ports[].resources.limits.memory**
* 字段类型：String
* 说明：指定MEM内存限制，单位为MiB、GiB

**spec.containers[].ports[].resources.requests**
* 字段类型：Objects
* 说明：指定容器启动和调度时资源需求设置

**spec.containers[].ports[].resources.requests.cpu**
* 字段类型：String
* 说明：CPU需求个数，单位为core 数，容器启动时初始化数量

**spec.containers[].ports[].resources.requests.memory**
* 字段类型：String
* 说明：内存需求，单位为MiB、GiB，容器启动时初始化量

**spec.restartPolicy**
* 字段类型：String
* 说明：定义Pod的重启策略，Always（默认，一旦终止运行，则无论容器何时终止的，kubelet服务将重启它）、OnFailure（只有Pod以非零退出码终止时，kubelet才会重启该容器，如果容器正常结束退出码为0）、Never：Pod终止后，kubelet将退出码报告给Master，不会重启服务该Pod

**spec.imagePullSecrets**
* 字段类型：Object
* 说明：定义Pull镜像时使用secret名称，以name: secretKey格式指定

**spec.hostNetwork**
* 字段类型：Boolean
* 说明：定义是否使用主机网络模式，默认为 false。设置true表示使用宿主机网络，不是用docker网桥，同时设置了true将无法在同一台宿主机上启动第二个副本


## 二、举例说明
* 创建一个namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test
```

* 创建一个pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: nginx-containers
    image: nginx:latest
```

