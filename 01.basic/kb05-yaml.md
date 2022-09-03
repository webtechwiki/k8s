# k8s资源清单文件

## 一、概述

在k8s中，一般使用yaml格式的文件来定义符合我们预期的Pod，这样的yaml文件被称为资源清单文件

## 二、常用字段

|                               字段名                               |  类型   |                                                                                                                          说明                                                                                                                          |
| ------------------------------------------------------------------ | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| version                                                            | String  | k8s的api的版本，目前基本上是v1，可以使用`kubectl api-versions`命令查看                                                                                                                                                                                 |
| kind                                                               | String  | 指定文件定义的资源类型和角色，比如：pod                                                                                                                                                                                                                |
| metadata                                                           | Object  | 元数据对象                                                                                                                                                                                                                                             |
| metadata.name                                                      | String  | 元数据对象名称，由我们自定义。比如是资源类型是pod，我们在这里可以定义pod的名称                                                                                                                                                                         |
| metadata.namespace                                                 | String  | 元数据对象命名空间，由我们自定义                                                                                                                                                                                                                       |
| Spec                                                               | Object  | 详细定义对象                                                                                                                                                                                                                                           |
| spec.containers[]                                                  | List    | Spec对象的容器列表                                                                                                                                                                                                                                     |
| spec.containers[$index].name                                       | String  | 定义容器的名称                                                                                                                                                                                                                                         |
| spec.containers[$index].image                                      | String  | 定义容器的镜像名称                                                                                                                                                                                                                                     |
| spec.containers[$index].imagePullPolicy                            | String  | 定义镜像拉取策略，有Always（每次都尝试拉取新镜像）、Never（仅使用本地镜像）、Ifnotpresent（本地有镜像则不拉取新镜像）三个值可选                                                                                                                        |
| spec.containers[$index].command[]                                  | List    | 指定容器启动命令，因为是数组可以指定多个，不指定则使用镜像打包时的启动命令                                                                                                                                                                             |
| spec.containers[$index].args                                       | List    | 指定容器启动命令参数，因为是数组可以指定多个                                                                                                                                                                                                           |
| spec.containers[$index].workDir                                    | String  | 指定容器的工作目录                                                                                                                                                                                                                                     |
| spec.containers[$index].volumeMounts[]                             | List    | 指定容器内部的存储卷配置                                                                                                                                                                                                                               |
| spec.containers[$index].volumeMounts[$subIndex].name               | String  | 指定容器加载的存储卷的名称                                                                                                                                                                                                                             |
| spec.containers[$index].volumeMounts[$subIndex].mountPath          | String  | 指定容器加载的存储卷的路径                                                                                                                                                                                                                             |
| spec.containers[$index].volumeMounts[$subIndex].readOnly           | String  | 设置存储卷的读写模式，true或者false，默认为读写模式，可以使用true代表只读模式                                                                                                                                                                          |
| spec.containers[$index].ports[]                                    | String  | 指定容器需要用到的端口列表                                                                                                                                                                                                                             |
| spec.containers[$index].ports[$subIndex].name                      | String  | 指定的端口名称                                                                                                                                                                                                                                         |
| spec.containers[$index].ports[$subIndex].containerPort             | String  | 指定容器需要监听的端口号                                                                                                                                                                                                                               |
| spec.containers[$index].ports[$subIndex].hostPort                  | String  | 指定容器所在主机需要监听的端口号，默认跟containerPort相同，注意：设置了hostPort同一台主机无法启动该容器的相同副本，因为主机的端口不能相同，这样会冲突                                                                                                  |
| spec.containers[$index].ports[$subIndex].protocol                  | String  | 指定端口协议，支持TCP和UDP，默认值为TCP                                                                                                                                                                                                                |
| spec.containers[$index].ports[$subIndex].env[]                     | List    | 指定容器运行前需设的环境变量列表                                                                                                                                                                                                                       |
| spec.containers[$index].ports[$subIndex].env[$secSubIndex].name    | List    | 指定环境变量名称                                                                                                                                                                                                                                       |
| spec.containers[$index].ports[$subIndex].env[$secSubIndex].value   | List    | 指定环境变量值                                                                                                                                                                                                                                         |
| spec.containers[$index].ports[$subIndex].resources                 | Object  | 指定资源限制和资源请求的值（这里开始就是设置资源的上限）                                                                                                                                                                                               |
| spec.containers[$index].ports[$subIndex].resources.limits          | Object  | 指定容器运行时资源的运行上限                                                                                                                                                                                                                           |
| spec.containers[$index].ports[$subIndex].resources.limits.cpu      | String  | 指定CPU限制，单位为core数，将用于`docker run --cpu-shares`参数                                                                                                                                                                                         |
| spec.containers[$index].ports[$subIndex].resources.limits.memory   | String  | 指定MEM内存限制，单位为MiB、GiB                                                                                                                                                                                                                        |
| spec.containers[$index].ports[$subIndex].resources.requests        | Objects | 指定容器启动和调度时资源需求设置                                                                                                                                                                                                                       |
| spec.containers[$index].ports[$subIndex].resources.requests.cpu    | String  | CPU需求个数，单位为core 数，容器启动时初始化数量                                                                                                                                                                                                       |
| spec.containers[$index].ports[$subIndex].resources.requests.memory | String  | 内存需求，单位为MiB、GiB，容器启动时初始化量                                                                                                                                                                                                           |
| spec.restartPolicy                                                 | String  | 定义Pod的重启策略，Always（默认，一旦终止运行，则无论容器何时终止的，kubelet服务将重启它）、OnFailure（只有Pod以非零退出码终止时，kubelet才会重启该容器，如果容器正常结束退出码为0）、Never：Pod终止后，kubelet将退出码报告给Master，不会重启服务该Pod |
| spec.imagePullSecrets                                              | Object  | 定义Pull镜像时使用secret名称，以name: secretKey格式指定                                                                                                                                                                                                |
| spec.hostNetwork                                                   | Boolean | 定义是否使用主机网络模式，默认为 false。设置true表示使用宿主机网络，不是用docker网桥，同时设置了true将无法在同一台宿主机上启动第二个副本                                                                                                               |

## 三、示例

### 3.1 定义一个namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test
```

### 3.2 定义一个pod

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
