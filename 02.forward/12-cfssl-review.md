# cfssl证书工具概述


通过前面几节文章，我们已经使用k8s二进制安装包从零开始搭建好了k8s集群，但目前我们不是百分之百的完成我们的搭建。不过核心的组件已经安装完成并且正常了，我们将在持续完成相关知识点。在本节，我们先回顾cfssl这个证书工具工具。




## 关于cfssl工具


- `cfssl`：证书签发的主要工具
- `cfssl-json`：将cfssl生成的证书（json格式）变为文件承载式证书
- `cfssl-certinfo`：用于验证证书信息



> 关于kubeconfig文件： 这是k8s用户的配置文件，它里面含有证书的，证书过期或替换该文件


## 使用cfssl-certinfo 查看证书信息



**查看pem证书信息**

使用`cfssl-certinfo`查看`pem`文件的信息


```shell-script
cfssl-certinfo -cert apiserver.pem
```

此命令返回证书的详细信息



**查看某个域名的证书信息**

查看百度首页的证书信息，使用如下命令

```shell-script
cfssl-certinfo -domain www.baidu.com
```


**解析kubeconfig中的证书信息**


我们在安装kubelet时，创建了`kubelet.kubeconfig`配置文件，实际上这个文件是将一些配置集合到此文件中，包括相关的证书信息，因此我们可以从中反解到文件中相关的证书信息。相关步骤如下：



我们取该文件的`users.user.client-certificate-data`这个节点下字符串内容，使用`base64`解码，如下命令

```shell-script
echo $CONTENT | base64 -d
```

`$CONTENT`为实际的字符串内容。执行上面命令后，将输入证书的内容。


有证书内容输出，可以输出到特定文件中，如下指令

```shell
echo $CONTENT | base64 -d > tmp.pem
```

再使用`cfssl-certinfo`查看证书文件的包含的证书详细信息，如下命令

```shell
cfssl-certinfo -cert tmp.pem
```







