# 在 Qemu 上运行 hvisor

我们建议在 Ubuntu 上进行实践，以下示例均基于 Ubuntu 发行版。

若你的操作系统为 Windows，你可以使用 WSL 或者 VMware/VirtualBox 虚拟机。 

## 一、安装 Qemu
若你已经拥有合适的 Qemu 可以使用，你可以跳过这一步。

我们建议使用源码编译的 Qemu，这样可以更加灵活地进行版本控制以及修改 Qemu 源码等等。

这里以 Qemu v9.0.2 为例，你也可以选择最新的版本：
```bash
# 安装依赖
sudo apt-get install git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev ninja-build
# 获取 Qemu 源码
wget https://download.qemu.org/qemu-9.0.2.tar.xz
# 解压
tar xvJf qemu-9.0.2.tar.xz
# 进入源码目录
cd qemu-9.0.2
# 配置 riscv target
./configure --target-list=riscv64-softmmu,riscv64-linux-user 
# 编译 qemu
make -j$(nproc)
# 测试是否安装成功
./build/qemu-system-riscv64 --version
```
你可以选择将它安装到环境变量，这样你可以使用 qemu-system-riscv64，而无需显式标明路径。

一种常见的方式是将其安装到 /opt/riscv, 并配置环境变量指向它。


## 二、安装 riscv 交叉编译器
我们需要用 riscv 交叉编译器来将 Linux 与 OpenSBI 编译成二进制文件，这里选择 https://github.com/riscv-collab/riscv-gnu-toolchain 。

建议从 [Github Release](https://github.com/riscv-collab/riscv-gnu-toolchain/releases) 处下载编译好的交叉编译器，这里推荐下载 riscv64-glibc-ubuntu-xxxx-gcc、riscv64-elf-ubuntu-xxxx-gcc 两个压缩包。

一种常见的方式是将其安装到 /opt/riscv, 并配置环境变量指向它。

> 注意：这里不推荐使用源码编译，因为你可能会遇到各种各样的问题。

## 三、编译 Linux
如果你要运行 qemu-aia platform，请选择 linux v6.10 及以上版本，低版本的 linux 中不含 aia 的驱动，会导致 linux 无法正常工作。

这里以 linux v6.10 为例：

```bash
git clone https://github.com/torvalds/linux -b v6.10 --depth=1
cd linux
git checkout v6.10
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j$(nproc)
```

## 四、制作 ubuntu 根文件系统
ubuntu 根文件系统包含 apt，可以在后续按需下载需要的软件包，相较于 busybox、buildroot 而言，功能会更加丰富。

这里给出两种方式：

### 1. 使用自动构建 Ubuntu 根文件系统脚本
参考 https://github.com/LubanCat/ubuntu 。


### 2. 自制 ubuntu-base 根文件系统

```bash
wget http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.2-base-riscv64.tar.gz
mkdir rootfs
dd if=/dev/zero of=riscv_rootfs.img bs=1M count=1024 oflag=direct
mkfs.ext4 riscv_rootfs.img
sudo mount -t ext4 riscv_rootfs.img rootfs/
sudo tar -xzf ubuntu-base-20.04.2-base-riscv64.tar.gz -C rootfs/

sudo cp /path-to-qemu/build/qemu-system-riscv64 rootfs/usr/bin/
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
sudo mount -t proc /proc rootfs/proc
sudo mount -t sysfs /sys rootfs/sys
sudo mount -o bind /dev rootfs/dev
sudo mount -o bind /dev/pts rootfs/dev/pts
sudo chroot rootfs 

# chroot 进入 rootfs 后，安装必要的软件包：
apt-get update
apt-get install git sudo vim bash-completion kmod net-tools iputils-ping resolvconf ntpdate
exit

sudo umount rootfs/proc
sudo umount rootfs/sys
sudo umount rootfs/dev/pts
sudo umount rootfs/dev
sudo umount rootfs
```

## 五、Rust 环境配置

请参考 [Rust 语言圣经](https://course.rs/first-try/intro.html)。

## 六、编译运行 hvisor

### 1. 准备 hvisor 源码和必要文件

将 [hvisor 代码仓库](https://github.com/syswonder/hvisor) 克隆到本地。

```bash
git clone https://github.com/syswonder/hvisor.git
```

在 hvisor/platform/riscv64/{BOARD}/image 文件夹下添加 Linux Image、根文件系统等等。

BOARD 为 qemu-plic, 如果要在 qemu-aia 平台上执行，则将 BOARD 为 qemu-aia。

将之前编译好的根文件系统、Linux 内核镜像分别放在 virtdisk、kernel 目录下，并分别重命名为 rootfs1.ext4、Image，它们在 Makefile 中指定了，你也可以修改 Makefile 中的内容。

### 2. 编译设备树

```bash
# 编译设备树
make BID=riscv64/{BOARD} dtb
```
### 3. 编译并运行 hvisor

在 hvisor 目录下，根据需要执行下列命令：
```bash
# 对于 qemu-plic board 执行
make run ARCH=riscv64 BOARD=qemu-plic

# 对于 qemu-aia board 执行
make run ARCH=riscv64 BOARD=qemu-aia
```

注意: Makefile 中的启动命令中包含了 -S 参数，可以看做 Qemu 虚拟机启动时的一个断点，需要 Qemu 收到 continue 才可以继续执行，这时可以方便地查看 /dev/pts/xxx, 可以把它看做是 Qemu 提供的 virtio-console 暴露给 Host 使用的虚拟串口设备。

你可以看到类似以下内容：
```
char device redirected to /dev/pts/4 (label serial3)
```
然后，同时按下 ctrl+a, 随后输入 c，回车，即可继续执行，随后会打印 OpenSBI + Hvisor + Linux 的输出信息。

为了使用上述的虚拟串口，你需要新建一个终端，然后可以通过如下命令连接它：

```bash
screen /dev/pts/xxx
```

注意：Qemu 只提供一个物理串口，当启动两个 zone 并且两个 zone 各自占用一个串口时，就需要使用到该虚拟串口设备。

### 4. 启动 non-root linux
注意：Non-root 使用设备有两种方式，设备直通和 virtio，其中 virtio 设备的后端在 root zone(linux)。

对于 hvisor，我们提供了管理程序 hvisor-tool，具体请参考 [hvisor-tool](https://github.com/syswonder/hvisor-tool) 的 README。

对于 riscv 架构来说，编译 hvisor-tool 时，建议除了上述下载的交叉工具链外，另外使用 ubuntu apt 安装 riscv64-linux-gnu-gcc。

例如，若要编译面向 riscv 架构的命令行工具，且 Hvisor 环境中的 Linux 镜像编译来源的源码位于 `~/linux`，则可执行：
```bash
make all ARCH=riscv LOG=LOG_WARN KDIR=~/linux
```

请务必保证 Hvisor 中的 Root Linux 镜像是由编译 hvisor-tool 时参数选项中的 Linux 源码目录编译产生。

编译完成后，将 output/hvisor.ko、output/hvisor 复制到 hvisor/platform/riscv64/{BOARD}/image/virtdisk/rootfs1.ext4 根文件系统中，你可以先将 rootfs1.ext4 挂载后进行拷贝。

再将 zone1 的内核镜像（如果是与 zone0 相同的 Linux 内核，则复制一份 {BOARD}/image/kernel/Image 即可）、设备树（{BOARD}/image/dts/linux2.dtb）、配置文件（{BOARD}/configs/zone1-linux.json等）拷贝到 rootfs1.ext4 根文件系统中，你可以将它们重命名为 Image、linux2.dtb、linux2.json 等(与 .json 里面的文件名匹配即可)。

除此之外，还需要为 Zone1 linux 制作一个根文件系统。可以将 {BOARD}/image/virtdisk 中的 rootfs1.ext4 复制一份，也可以重新制作根文件系统（最好改小镜像大小），并改名为 riscv_rootfs2.img（和 .json 里面的文件名匹配即可）。之后将 riscv_rootfs2.img 放入 rootfs1.ext4 根文件系统中。

对于 BOARD=qemu-plic，启动 root linux 后，你可以按照如下方式启动 non-root linux：
```bash
insmod hvisor.ko
rm nohup.out
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
nohup ./hvisor zone start zone1-linux.json && cat nohup.out | grep "char device" && script /dev/null
```
对于 BOARD=qemu-aia，启动 root linux 后，你可以按照如下方式启动 non-root linux：
```bash
insmod hvisor.ko
mount -t proc proc /proc
mount -t sysfs sysfs /sys
rm nohup.out
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
nohup ./hvisor virtio start zone1-linux-virtio.json &
./hvisor zone start zone1-linux.json && \
cat nohup.out | grep "char device" && \
script /dev/null
```
注意：它们的区别在于配置不同，你可以修改配置，以自定义使用设备直通还是 virtio，以当前 hvisor 中的默认配置为例：

- qemu-plic 采用的是直通的设备（尽管启动命令中为 virtio 设备，这里可以看做是直通，因为它由 Qemu 提供设备后端）

- qemu-aia 采用了 virtio，由 root linux 提供 virtio 设备后端（non-root 的 virtio 驱动会被拦截转发到 root linux 的 virtio 后端）
