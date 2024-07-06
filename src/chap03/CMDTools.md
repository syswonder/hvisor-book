# 命令行工具

命令行工具是附属于hvisor的管理工具，用于在管理虚拟机Root Linux上创建和关闭其他虚拟机，并负责启动Virtio守护进程，提供Virtio设备模拟。仓库地址位于[hvisor-tool](https://github.com/syswonder/hvisor-tool)。

## 如何编译

命令行工具目前支持两种体系结构：arm64和riscv，且需要配合一个内核模块才能使用。在x86主机上通过交叉编译可针对不同体系结构进行编译

* arm64编译

在hvisor-tool目录下执行以下命令，即可得到面向arm64体系结构的命令行工具hvisor以及内核模块hvisorl.ko。

```
make all ARCH=arm64 KDIR=xxx
```

其中KDIR为Root Linux源码路径，用于内核模块的编译。

* riscv编译

编译面向riscv体系结构的命令行工具和内核模块：

```
make all ARCH=riscv KDIR=xxx
```

## 对虚拟机进行管理

### 加载内核模块

使用命令行工具前，需要加载内核模块，便于用户态程序与Hyperviosr进行交互：

```
insmod hvisor.ko
```

卸载内核模块的操作为：

```
rmmod hvisor.ko
```

其中hvisor.ko位于hvisor-tool/driver目录下。

### 启动一个虚拟机

在Root Linux上可通过以下命令，创建一个id为1的虚拟机。该命令会将虚拟机的操作系统镜像文件`Image`加载到真实物理地址`xxxa`处，将虚拟机的设备树文件`linux2.dtb`加载到真实物理地址`xxxb`处，并进行启动。

```
./hvisor zone start --kernel Image,addr=xxxa --dtb linux2.dtb,addr=xxxb --id 1
```

### 关闭一个虚拟机

关闭id为1的虚拟机：

```
./hvisor zone shutdown -id 1
```