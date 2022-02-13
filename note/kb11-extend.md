# 扩展与结语


## 1. 扩展内容

在本系列的教程中，我们使用multipass搭建k8s的基础环境，我们还可以使用**vagrant**虚拟机管理工具来搭建k8s的基础环境。定义`Vagrantfile`，添加如下内容

```shell
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.disksize.size = '10GB'
  # master1
  config.vm.define "master1" do |master1|
    master1.vm.network "public_network", ip: "192.168.33.10"
    master1.vm.hostname = "master1"
    # 将宿主机../data目录挂载到虚拟机/vagrant_dev目录
    master1.vm.synced_folder "../data", "/vagrant_data"
    # 指定核心数和内存
    config.vm.provider "virtualbox" do |v|
      v.memory = 2048
      v.cpus = 2
    end
  end
  # worker1
  config.vm.define "worker1" do |worker1|
    worker1.vm.network "public_network", ip: "192.168.33.11"
    worker1.vm.hostname = "worker1"
    # 将宿主机../data目录挂载到虚拟机/vagrant_pro目录
    worker1.vm.synced_folder "../data", "/vagrant_data"
    # 指定核心数和内存
    config.vm.provider "virtualbox" do |v|
      v.memory = 2048
      v.cpus = 2
    end
  end
  # worker2
  config.vm.define "worker2" do |worker2|
    worker2.vm.network "public_network", ip: "192.168.33.12"
    worker2.vm.hostname = "worker2"
    # 将宿主机../data目录挂载到虚拟机/vagrant_pro目录
    worker2.vm.synced_folder "../data", "/vagrant_data"
    # 指定核心数和内存
    config.vm.provider "virtualbox" do |v|
      v.memory = 2048
      v.cpus = 2
    end
  end
end
```

使用`vagrant status`命令查看当前虚拟机的状态，可以看到如下内容

```shell
pan@pan-PC ~/Work/vagrant/kubernetes$ vagrant status                                                                                                          
Current machine states:

master1                   running (virtualbox)
worker1                   running (virtualbox)
worker2                   running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```


通过以下指令进入虚拟机中
```shell
# 进入master1主机
vagrant ssh master1
```


## 2. 结语


至此，k8s的基础知识就介绍这里，后续我将有机会产出k8s相关的深入知识，欢迎持续关注！

- 作者微信订阅号：极客开发者up

- 作者博客：[https://blog.jkdev.cn](https://blog.jkdev.cn)

