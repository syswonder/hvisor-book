# QEMU模拟ARM64内核

## 一、安装交叉编译器 aarch64-none-linux-gnu- 10.3

网址：[https://developer.arm.com/downloads/-/gnu-a](https://developer.arm.com/downloads/-/gnu-a) 

工具选择：AArch64 GNU/Linux target (aarch64-none-linux-gnu) 

下载链接：[https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz?rev=1cb9c51b94f54940bdcccd791451cec3&hash=B380A59EA3DC5FDC0448CA6472BF6B512706F8EC](https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz?rev=1cb9c51b94f54940bdcccd791451cec3&hash=B380A59EA3DC5FDC0448CA6472BF6B512706F8EC) 

```bash
wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
tar xvf gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
ls gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/
```

安装完成，记住路径，例如在：/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-，之后都会使用这个路径。

## 二、编译安装QEMU 7.0

```
# 安装编译所需的依赖包
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
              gawk build-essential bison flex texinfo gperf libtool patchutils bc \
              zlib1g-dev libexpat-dev pkg-config  libglib2.0-dev libpixman-1-dev libsdl2-dev \
              git tmux python3 python3-pip ninja-build
# 下载源码
wget https://download.qemu.org/qemu-7.0.0.tar.xz 
# 解压
tar xvJf qemu-7.0.0.tar.xz   
cd qemu-7.0.0
#生成设置文件
./configure --enable-kvm --enable-slirp --enable-debug --target-list=aarch64-softmmu,x86_64-softmmu  
#编译
make -j$(nproc)   
```

之后编辑 `~/.bashrc` 文件，在文件的末尾加入几行：

```
# 请注意，qemu-7.0.0 的父目录可以随着你的实际安装位置灵活调整
export PATH=$PATH:/path/to/qemu-7.0.0/build
```

随后即可在当前终端 `source ~/.bashrc` 更新系统路径，或者直接重启一个新的终端。此时可以确认qemu版本：

```
qemu-system-aarch64 --version   #查看版本
```

> 注意，上述依赖包可能不全，例如：
>
> - 出现 `ERROR: pkg-config binary 'pkg-config' not found` 时，可以安装 `pkg-config` 包；
> - 出现 `ERROR: glib-2.48 gthread-2.0 is required to compile QEMU` 时，可以安装 `libglib2.0-dev` 包；
> - 出现 `ERROR: pixman >= 0.21.8 not present` 时，可以安装 `libpixman-1-dev` 包。

## 三、编译Kernel 5.4

```bash
git clone https://github.com/torvalds/linux -b v5.4 --depth=1
cd linux
git checkout v5.4
make ARCH=arm64 CROSS_COMPILE=/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- defconfig
```

为了能使linux产生一个/dev/ram0，便于后续使用，此时需要编辑本地.config文件，添加一行：

```
CONFIG_BLK_DEV_RAM=y
```

之后进行编译：

```shell
make ARCH=arm64 CROSS_COMPILE=/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- Image -j$(nproc)
```

> 如果编译linux时报错：
>
> ```
> /usr/bin/ld: scripts/dtc/dtc-parser.tab.o:(.bss+0x20): multiple definition of `yylloc'; scripts/dtc/dtc-lexer.lex.o:(.bss+0x0): first defined here
> ```
>
> 则修改linux文件夹下`scripts/dtc/dtc-lexer.lex.c`，在`YYLTYPE yylloc;`前增加`extern`。再次编译，发现会报错：openssl/bio.h: No such file or directory ，此时执行`sudo apt install libssl-dev`

编译完毕，内核文件位于：arch/arm64/boot/Image。记住整个linux文件夹所在的路径，例如：home/korwylee/lgw/hypervisor/linux。

## 四、基于ubuntu 20.04 arm64 base构建文件系统

busybox制作的文件系统过于简单（如没有apt工具），因此，我们需要使用更丰富的ubuntu文件系统来制作linux的根文件系统。注意，ubuntu22.04也可以。

下载：[ubuntu-base-20.04.5-base-arm64.tar.gz](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz)  

链接：[http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz)

```bash
wget http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz

mkdir rootfs
# 创建一个ubuntu.img
dd if=/dev/zero of=ubuntu-20.04-rootfs_ext4.img bs=1M count=4096 oflag=direct
mkfs.ext4 ubuntu-20.04-rootfs_ext4.img
# 将ubuntu.tar.gz放入已经挂载到rootfs上的ubuntu.img中
sudo mount -t ext4 ubuntu-20.04-rootfs_ext4.img rootfs/
sudo tar -xzf ubuntu-base-20.04.5-base-arm64.tar.gz -C rootfs/

# 让rootfs绑定和获取物理机的一些信息和硬件
# qemu-path为你的qemu路径
sudo cp qemu-path/build/qemu-system-aarch64 rootfs/usr/bin/ 
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
sudo mount -t proc /proc rootfs/proc
sudo mount -t sysfs /sys rootfs/sys
sudo mount -o bind /dev rootfs/dev
sudo mount -o bind /dev/pts rootfs/dev/pts

# 执行该指令可能会报错，请参考下面的解决办法
sudo chroot rootfs 

apt-get update
apt-get install git sudo vim bash-completion -y
apt-get install net-tools ethtool ifupdown network-manager iputils-ping -y
apt-get install rsyslog resolvconf udev -y

# 如果上面软件包没有安装，至少要安装下面的包
apt-get install systemd -y

apt-get install build-essential git wget flex bison libssl-dev bc libncurses-dev kmod -y

adduser arm64
adduser arm64 sudo
echo "kernel-5_4" >/etc/hostname
echo "127.0.0.1 localhost" >/etc/hosts
echo "127.0.0.1 kernel-5_4">>/etc/hosts
dpkg-reconfigure resolvconf
dpkg-reconfigure tzdata
exit

sudo umount rootfs/proc
sudo umount rootfs/sys
sudo umount rootfs/dev/pts
sudo umount rootfs/dev
sudo umount rootfs
```

最后卸载挂载，完成根文件系统的制作。

> 执行`sudo chroot .`时，如果报错`chroot: failed to run command ‘/bin/bash’: Exec format error`，可以执行指令：
>
> ```
> sudo apt-get install qemu-user-static
> sudo update-binfmts --enable qemu-aarch64
> ```

## 五、编译jailhouse

```bash
sudo mount ubuntu-20.04-rootfs_ext4.img rootfs/
# 进入rootfs中的某个文件夹
```

编译rootfs:

```bash
# for x86
git clone https://github.com/siemens/jailhouse.git
cd jailhouse 
make

# for aarch64
git clone https://github.com/siemens/jailhouse.git
cd jailhouse 
# hvisor参考的是v0.10版本，因此如果希望对照hvisor，请切换到v0.10分支
git checkout v0.10
# 如果希望用编译出的jailhouse运行hvisor，则需要执行下一条指令打patch；否则，请不要作此操作
patch -f -p1 < hvisor/hvisor.patch #见后面的说明，其中有仓库地址
# KDIR为之前编译的linux文件夹
make ARCH=arm64 CROSS_COMPILE=/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- KDIR=/home/korwylee/lgw/hypervisor/linux

sudo make ARCH=arm64 CROSS_COMPILE=/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- KDIR=/home/korwylee/lgw/hypervisor/linux DESTDIR=/home/korwylee/lgw/hypervisor/linux install 
```

说明：编译jailhouse一定要指定KDIR，说明sysroot目标，才可以编译成功，并且需要提前编译一个linux作为sysroot，否则默认从本机linux源码目录中去找相应的库代码。

> hvisor位于https://github.com/syswonder/hvisor

## 六、启动QEMU

```bash
qemu-system-aarch64 \
	-machine virt,gic_version=3 \
	-machine virtualization=true \
	-cpu cortex-a57 \
	-machine type=virt \
	-nographic \
	-smp 16 \
	-m 1024 \
	-kernel ./linux/arch/arm64/boot/Image \
	-append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
	-drive if=none,file=ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
	-device virtio-blk-device,drive=hd0 \
	-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
	-device virtio-net-device,netdev=net0,mac=52:55:00:d1:55:01
```

其中kernel为之前编译好的linux镜像。

## 七、运行jailhouse

```shell
cd ~/jailhouse
sudo mkdir -p /lib/firmware
sudo cp hypervisor/jailhouse.bin /lib/firmware/
sudo insmod driver/jailhouse.ko
sudo ./tools/jailhouse enable configs/arm64/qemu-arm64.cell
# 关闭jailhouse则执行下面的命令
sudo ./tools/jailhouse disable
```

启动一个gic-demo：

```shell
sudo ./tools/jailhouse cell create configs/arm64/qemu-arm64-gic-demo.cell
sudo ./tools/jailhouse cell load gic-demo inmates/demos/arm64/gic-demo.bin
sudo ./tools/jailhouse cell start gic-demo
sudo ./tools/jailhouse cell destroy gic-demo
```

常见问题：

> 1. 如果执行jailhouse报错说glibc版本低，则在host上chroot到rootfs中换源，增加ubuntu 22.04的清华源。步骤如下：
>
> 执行命令
>
> ```
> sudo vi /etc/apt/sources.list
> ```
>
> 替换文件内容为：
>
> ```
> # 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
> deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal main restricted universe multiverse
> # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal main restricted universe multiverse
> deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-updates main restricted universe multiverse
> # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-updates main restricted universe multiverse
> deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-backports main restricted universe multiverse
> # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-backports main restricted universe multiverse
> 
> # deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-security main restricted universe multiverse
> # # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-security main restricted universe multiverse
> 
> deb http://ports.ubuntu.com/ubuntu-ports/ focal-security main restricted universe multiverse
> # deb-src http://ports.ubuntu.com/ubuntu-ports/ focal-security main restricted universe multiverse
> 
> # 预发布软件源，不建议启用
> # deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-proposed main restricted universe multiverse
> # # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-proposed main restricted universe multiverse
> deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ jammy main
> ```
>
> 替换后执行：
>
> ```
> sudo apt update 
> sudo apt install libc6
> ```
>
> 2. 如果运行jailhouse时报ext4 error文件系统的错，则可以在host上执行：
>
> ```
> e2fsck -f ubuntu-20.04-rootfs_ext4.img
> ```

## 八、启动一个non-root-linux on qemu

non-root-linux启动时，需要挂载文件系统，下面的教程分为内存文件系统和磁盘文件系统。

### 8.1 内存文件系统

#### 8.1.1编译 busybox 1.36.0 （文件系统）

```bash
wget https://busybox.net/downloads/busybox-1.36.0.tar.bz2
tar -jxvf busybox-1.36.0.tar.bz2

cd busybox-1.36.0
sudo apt-get install libncurses5-dev 
make menuconfig
```

在弹出窗口中，依次进入并设定：

* Settings
  * [*] Build static binary (no shared libs)
  * Cross compiler prefix设置为：`/交叉编译器路径/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-`

效果为：

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310131545422.png" alt="Untitled"  />

```bash
# 编译安装
make -j$(nproc) && make install
```

编译完毕后busybox生成在_install目录。

#### 8.1.2 为文件系统创建console

```bash
cd _install
mkdir dev
cd dev
sudo mknod console c 5 1
sudo mknod null c 1 3
sudo mknod tty1 c 4 1
sudo mknod tty2 c 4 2
sudo mknod tty3 c 4 3
sudo mknod tty4 c 4 4
cd ..
mkdir -p etc/init.d/ 
cd etc/init.d; touch rcS
chmod +x rcS
vi rcS

#回到_install目录
cd ../..
# 压缩成cpio.gz文件系统
find . -print0 | cpio --null -ov --format=newc | gzip -9 > initramfs.cpio.gz
```

之后将initramfs.cpio.gz通过挂载根文件系统，传入到guest linux中。然后启动QEMU，并enable jailhouse。

> 如果报错说找不到python，则执行以下命令:
>
> ```
> sudo ln -s /usr/bin/python3 /usr/bin/python
> ```

之后执行：

```bash
cd tools/
sudo ./jailhouse cell linux ../configs/arm64/qemu-arm64-linux-demo.cell ../linux-Image -c "console=ttyAMA0,115200 root=/dev/ram0 rdinit=/linuxrc" -d ../configs/arm64/dts/inmate-qemu-arm64.dtb -i ../initramfs.cpio.gz
```

其中linux-Image是之前编译的linux镜像。

### 8.2 磁盘文件系统

制作新的ubuntu镜像：

> 为了省事，这里只做了一个简单的镜像，更完整的镜像文件参考第四章。

```bash
dd if=/dev/zero of=second-ubuntu-20.04-rootfs_ext4.img bs=1M count=256 oflag=direct
mkfs.ext4 second-ubuntu-20.04-rootfs_ext4.img
sudo mount -t ext4 second-ubuntu-20.04-rootfs_ext4.img rootfs/
sudo tar -xzf ubuntu-base-20.04.5-base-arm64.tar.gz -C rootfs/
sudo cp qemu-path/build/qemu-system-aarch64 rootfs/usr/bin/
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
```

要让non-root-linux启动一个磁盘文件系统，那么它应该能通过设备树在qemu模拟的virtio-blk设备中找到对应的磁盘。

为了找到qemu模拟的virtio-blk设备，需要导出qemu模拟的设备树信息：

```bash
sudo qemu-system-aarch64 \
    -machine virt,gic_version=3 \
    -machine virtualization=true \
    -cpu cortex-a57 \
    -machine type=virt \
    -nographic \
    -smp 16 \
    -m 1024 \
    -kernel ./linux/arch/arm64/boot/Image \
    -append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
    -drive if=none,file=second-ubuntu-20.04-rootfs_ext4.img,id=hd1,format=raw \
    -device virtio-blk-device,drive=hd1 \
    -drive if=none,file=ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
    -device virtio-blk-device,drive=hd0 \
    -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
    -device virtio-net-device,netdev=net0,mac=52:55:00:d1:55:01 \
    -machine dumpdtb=qemu-virt.dtb
```

其中machine dumpdtb就将qemu设备树导出到dtb文件中，之后需要将其转换为可读的dts文件，在shell中执行：

```bash
dtc -I dtb -O dts -o qemu-virt.dts qemu-virt.dtb
```

qemu指定的virtio设备在设备树中是倒着来的，由于我们指定second-ubuntu-20.04-rootfs_ext4.img为non-root-linux的磁盘，它是qemu启动参数中的第一个设备，故找到最后一个virtio-mmio区域：

```bash
virtio_mmio@a003e00 {
    dma-coherent;
    interrupts = <0x00 0x2f 0x01>;
    reg = <0x00 0xa003e00 0x00 0x200>;
    compatible = "virtio,mmio";
};
```

这说明，该磁盘所在的mmio区域从0xa003e00开始，大小为0x200，用到了中断为SPI的第0x2f号中断，即32+47=79号中断。将其写入到inmate-qemu-arm64.dts，同时为non-root-linux的cell配置文件增加相应的mem region：

```c
		/*disk*/ {
			.phys_start = 0x0a003e00,
			.virt_start = 0x0a003e00,
			.size = 0x0200,
			.flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
				JAILHOUSE_MEM_IO | JAILHOUSE_MEM_IO_32 | JAILHOUSE_MEM_IO_8 | JAILHOUSE_MEM_IO_16 | JAILHOUSE_MEM_IO_64,
		},
```

由于non-root-linux在probe virtio-blk时，用到了79号中断，因此修改cell配置文件：

```c
	.irqchips = {
		/* GIC */ {
			.address = 0x08000000,
			.pin_base = 32, // pin_base 表示从第几号中断开始，pin_bitmap的类型为u32[4]，
			.pin_bitmap = { // 每一个元素表示32个中断，其中位设为1的中断，root cell会取消拥有该中断
				(1 << (33 - 32)),
				(1 << 15),  // 79号中断
				0,
				(1 << (140 - 128))
			},
		},
	},
```

之后编译jailhouse，然后启动qemu：

```bash
sudo qemu-system-aarch64 \
    -machine virt,gic_version=3 \
    -machine virtualization=true \
    -cpu cortex-a57 \
    -machine type=virt \
    -nographic \
    -smp 16 \
    -m 1024 \
    -kernel ./linux/arch/arm64/boot/Image \
    -append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
    -drive if=none,file=second-ubuntu-20.04-rootfs_ext4.img,id=hd1,format=raw \
    -device virtio-blk-device,drive=hd1 \
    -drive if=none,file=ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
    -device virtio-blk-device,drive=hd0 \
    -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
    -device virtio-net-device,netdev=net0,mac=52:55:00:d1:55:01
```

之后启动non-root-linux:

```bash
sudo insmod driver/jailhouse.ko
sudo ./tools/jailhouse enable configs/arm64/qemu-arm64.cell
cd tools/
sudo ./jailhouse cell linux ../configs/arm64/qemu-arm64-linux-demo.cell ../linux-Image -c "console=ttyAMA0,115200 root=/dev/vda rw" -d ../configs/arm64/dts/inmate-qemu-arm64.dtb
```

## 附录

### 1. 配置qemu网络，与外部通信

```bash
+-----------------------------------------------------------------+
|  Host                                                           |
| +---------------------+                                         |
| |                     |                                         |
| | br0:                |                                         |
| |   192.168.0.32/24 +-----+                                   |
| |                     |     |                                   |
| +----+----------------+     |       +-------------------------+ |
|      |                      |       |  Guest                  | |
|      |                      |       | +---------------------+ | |
| +----+----------------+  +--+---+   | |                     | | |
| |                     |  |      |   | | eth0:               | | |
| | eth1:               |  | tap0 |   | |   192.168.0.33/24 | | |
| |   192.168.0.175/24   |  |      +-----+                     | | |
| |                     |  |      |   | +---------------------+ | |
| +---------------------+  +------+   +-------------------------+ |
+-----------------------------------------------------------------+
```

网络连接图

配置网桥 `br0`

```bash
sudo ip link add name br0 type bridge
sudo ip link set dev br0 down
sudo ip addr flush dev br0
sudo ip addr add 192.168.0.32/24 dev br0
sudo ip link set dev br0 up
```

配置 tap 设备 `tap0`

```bash
sudo ip tuntap add name tap0 mode tap
sudo ip link set dev tap0 up
```

将宿主机网络接口 `eth0`和 `tap0`接入网桥 `br0`

```bash
sudo ip link set eth1 master br0
sudo ip link set tap0 master br0
```

然后qemu启动虚拟机，在虚拟机内

```bash
sudo ifconfig eth0 up
sudo ifconfig eth0 192.168.0.33
```

现在可以实现guest ping通host

#### 优化网络，让其可以与外部通信 (暂不可信)

以上步骤完成后虚拟机可与宿主机所在网络的其他设备互连（包括宿主机），也可以通过指定的网关连接互联网，但是此时宿主机无法连接互联网，解决方法如下:

删除 `eth0`接口的默认网关：

```bash
sudo ip route del default dev eth1
```

为 `br0`添加默认网关：

```bash
sudo ip route add default via 192.168.0.1 dev br0
```

### 2. img扩容

当rootfs chroot空间不足时，需要扩容，按照以下步骤进行无损扩容：

```bash
# 首先取消挂载img
umount ./rootfs

## bug: umount: /root/rootfs: target is busy.
## 解决：umount ./rootfs -l 强行卸载（慎用）

dd if=/dev/zero of=add.img bs=1M count=4096 # 新建4G空间
cat add.img >> ubuntu-20.04-rootfs_ext4.img
e2fsck -f ubuntu-20.04-rootfs_ext4.img
resize2fs ubuntu-20.04-rootfs_ext4.img

mount ubuntu-20.04-rootfs_ext4.img rootfs
`
