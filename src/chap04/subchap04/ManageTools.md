# hvisor管理工具

hvisor通过一个管理虚拟机Root Linux来管理整个系统。Root Linux通过一套管理工具为用户提供启动和关闭虚拟机、启动和关闭Virtio守护进程的服务。管理工具中，包含一个命令行工具和内核模块。其中命令行工具用于解析并执行用户输入的命令，内核模块用于命令行工具、Virtio守护进程与Hypervisor之间的通信。管理工具的仓库地址为：[hvisor-tool](https://github.com/syswonder/hvisor-tool)。

## 启动虚拟机

用户输入以下命令，可以在Root Linux上为hvisor创建一个新的虚拟机。

```
./hvisor zone start [vm_name].json
```

命令行工具首先会解析`[vm_name].json`文件内容，将虚拟机配置写入`zone_config`结构体。并根据文件中指定的镜像和dtb文件，将其内容通过`read`函数读入临时内存。为了将镜像和dtb文件加载到指定的物理内存地址，`hvisor.ko`内核模块提供`hvisor_map`函数，可以将一片物理内存区域映射到用户态虚拟地址空间。

当命令行工具对`/dev/hvisor`执行`mmap`函数时，内核会调用`hvisor_map`函数，以实现用户虚拟内存到指定物理内存的映射。之后通过内存拷贝函数，即可将镜像和dtb文件的内容从临时内存移动到用户指定的物理内存区域。

加载好镜像后，命令行工具对`/dev/hvisor`调用`ioctl`，指定操作码为`HVISOR_ZONE_START`，之后内核模块会通过Hypercall通知Hypervisor，并传入`zone_config`结构体对象的地址，通知Hypervisor启动虚拟机。

## 关闭虚拟机

用户输入命令：

```
./hvisor shutdown -id [vm_id]
```

即可关闭ID为`vm_id`的虚拟机。该命令会对`/dev/hvisor`调用`ioctl`，指定操作码为`HVISOR_ZONE_SHUTDOWN`，之后内核模块会通过Hypercall通知Hypervisor，传入`vm_id`，通知Hypervisor关闭虚拟机。

## 启动Virtio守护进程

用户输入命令：

```
nohup ./hvisor virtio start [virtio_cfg.json] &
```

即可根据`virtio_cfg.json`中规定的Virtio设备信息，创建Virtio设备，并初始化相关的数据结构。目前支持三种Virtio设备的创建，包括Virtio-net、Virtio-block、Virtio-console设备。

由于命令行参数中包含`nohup`和`&`，该命令会以守护进程的形式存在，守护进程的所有输出被重定向到`nohup.out`。守护进程的输出包含六个等级，从低到高分别是`LOG_TRACE`, `LOG_DEBUG`, `LOG_INFO`,`LOG_WARN`,`LOG_ERROR`,`LOG_FATAL`。编译命令行工具时可指定LOG级别，例如LOG为`LOG_INFO`时，等于或高于`LOG_INFO`的输出将被记录到日志文件，而`log_trace`和`log_debug`将不会输出。

Virtio设备创建后，Virtio守护进程会轮询请求提交队列，获取其他虚拟机的Virtio请求。长时间没有请求时，会自动进入休眠状态。

## 关闭Virtio守护进程

用户输入命令：

```
pkill hvisor
```

即可关闭Virtio守护进程。Virtio守护进程在启动时，会注册`SIGTERM`信号的信号处理函数`virtio_close`。当执行`pkill hvisor`时，会向名为`hvisor`的进程发送信号`SIGTERM`，此时守护进程会执行`virtio_close`，回收资源，关闭各个子线程，最后推出
