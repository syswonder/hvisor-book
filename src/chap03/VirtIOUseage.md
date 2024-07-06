# VirtIO设备的使用

目前hvisor支持三种Virtio设备：Virtio block，Virtio net和Virtio Console，以MMIO方式呈现给Root Linux以外的虚拟机。Virtio设备源码仓库位于[hvisor-tool](https://github.com/syswonder/hvisor-tool)，随命令行工具一起编译和使用。通过命令行工具创建Virtio设备后，Virtio设备会成为Root Linux上的一个守护进程，其日志信息会输出到nohup.out文件中。

## 创建和启动Virtio设备

通过命令行创建Virtio设备前，需执行`insmod hvisor.ko`加载内核模块。

### Virtio blk设备

在Root Linux控制台执行以下示例指令，可创建一个Virtio blk设备：

```shell
nohup ./hvisor virtio start \
	--device blk,addr=0xa003c00,len=0x200,irq=78,zone_id=1,img=rootfs2.ext4 &
```

其中`--device blk`表示创建一个Virtio磁盘设备，供id为`zone_id`的虚拟机使用。该虚拟机会通过一片MMIO区域与该设备交互，这片MMIO区域的起始地址为`addr`，长度为`len`，设备中断号为`irq`，对应的磁盘镜像路径为`img`。

> 使用Virtio设备的虚拟机需要在设备树中增加该Virtio mmio节点的相关信息。

### Virtio net设备

#### 创建网络拓扑

使用Virtio net设备前，需要在root Linux中创建一个网络拓扑图，以便Virtio net设备通过Tap设备和网桥设备连通真实网卡。在root Linux中执行以下指令：

```shell
mount -t proc proc /proc
mount -t sysfs sysfs /sys
ip link set eth0 up
dhclient eth0
brctl addbr br0
brctl addif br0 eth0
ifconfig eth0 0
dhclient br0
ip tuntap add dev tap0 mode tap
brctl addif br0 tap0
ip link set dev tap0 up
```

便可创建`tap0设备<-->网桥设备<-->真实网卡`的网络拓扑。

#### 启动Virtio net

在Root Linux控制台执行以下示例指令，可创建一个Virtio net设备：

```shell
nohup ./hvisor virtio start \
	--device net,addr=0xa003600,len=0x200,irq=75,zone_id=1,tap=tap0 &
```

`--device net`表示创建一个Virtio网络设备，供id为`zone_id`的虚拟机使用。该虚拟机会通过一片MMIO区域与该设备交互，这片MMIO区域的起始地址为`addr`，长度为`len`，设备中断号为`irq`，并连接到名为`tap`的Tap设备。

### Virtio console设备

在Root Linux控制台执行以下示例指令，可创建一个Virtio console设备：

```shell
nohup ./hvisor virtio start \
	--device console,addr=0xa003800,len=0x200,irq=76,zone_id=1 &
```

`--device console`表示创建一个Virtio控制台，供id为`zone_id`的虚拟机使用。该虚拟机会通过一片MMIO区域与该设备交互，这片MMIO区域的起始地址为`addr`，长度为`len`，设备中断号为`irq`。

执行`cat nohup.out | grep "char device"`，可观察到输出`char device redirected to /dev/pts/xx`。在Root Linux上执行：

```
screen /dev/pts/xx
```

即可进入该虚拟控制台，与该虚拟机进行交互。按下快捷键`Ctrl +a d`，即可返回Root Linux终端。执行`screen -r [session_id]`，即可重新进入虚拟控制台。

### 创建多个Virtio设备

执行以下命令，可同时创建Virtio blk、net、console设备，所有设备均位于一个守护进程。

```shell
nohup ./hvisor virtio start \
	--device blk,addr=0xa003c00,len=0x200,irq=78,zone_id=1,img=rootfs2.ext4 \
	--device net,addr=0xa003600,len=0x200,irq=75,zone_id=1,tap=tap0 \
	--device console,addr=0xa003800,len=0x200,irq=76,zone_id=1 &
```

## 关闭Virtio设备

执行该命令即可关闭Virtio守护进程及所有创建的设备：

```
pkill hvisor
```

