# 在 QEMU 上运行 hvisor

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

## 二、编译安装 QEMU 7.2.12

```
# 安装编译所需的依赖包
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
              gawk build-essential bison flex texinfo gperf libtool patchutils bc \
              zlib1g-dev libexpat-dev pkg-config  libglib2.0-dev libpixman-1-dev libsdl2-dev \
              git tmux python3 python3-pip ninja-build
# 下载源码
wget https://download.qemu.org/qemu-7.2.12.tar.xz
# 解压
tar xvJf qemu-7.2.12.tar.xz
cd qemu-7.2.12
#生成设置文件
./configure --enable-kvm --enable-slirp --enable-debug --target-list=aarch64-softmmu,x86_64-softmmu
#编译
make -j$(nproc)
```

之后编辑 `~/.bashrc` 文件，在文件的末尾加入几行：

```
# 请注意，qemu-7.2.12 的父目录可以随着你的实际安装位置灵活调整
export PATH=$PATH:/path/to/qemu-7.2.12/build
```

随后即可在当前终端 `source ~/.bashrc` 更新系统路径，或者直接重启一个新的终端。此时可以确认 qemu 版本：

```
qemu-system-aarch64 --version   #查看版本
```

> 注意，上述依赖包可能不全，例如：
>
> - 出现 `ERROR: pkg-config binary 'pkg-config' not found` 时，可以安装 `pkg-config` 包；
> - 出现 `ERROR: glib-2.48 gthread-2.0 is required to compile QEMU` 时，可以安装 `libglib2.0-dev` 包；
> - 出现 `ERROR: pixman >= 0.21.8 not present` 时，可以安装 `libpixman-1-dev` 包。

> 若生成设置文件时遇到报错 ERROR: Dependency "slirp" not found, tried pkgconfig：
>
> 下载 [https://gitlab.freedesktop.org/slirp/libslirp](https://gitlab.freedesktop.org/slirp/libslirp)包，并按 readme 安装即可。

## 三、编译 Linux Kernel 5.4

在编译 root linux 的镜像前, 在.config 文件中把 CONFIG_IPV6 和 CONFIG_BRIDGE 的 config 都改成 y, 以支持在 root linux 中创建网桥和 tap 设备。具体操作如下：

```
git clone https://github.com/torvalds/linux -b v5.4 --depth=1
cd linux
git checkout v5.4
# CROSS_COMPILE路径根据第一步安装交叉编译器的路径适当修改
make ARCH=arm64 CROSS_COMPILE=/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- defconfig
# 在.config中增加一行
CONFIG_BLK_DEV_RAM=y
# 修改.config的两个CONFIG参数
CONFIG_IPV6=y
CONFIG_BRIDGE=y
# 编译，CROSS_COMPILE路径根据第一步安装交叉编译器的路径适当修改
make ARCH=arm64 CROSS_COMPILE=/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- Image -j$(nproc)
```

> 如果编译 linux 时报错：
>
> ```
> /usr/bin/ld: scripts/dtc/dtc-parser.tab.o:(.bss+0x20): multiple definition of `yylloc'; scripts/dtc/dtc-lexer.lex.o:(.bss+0x0): first defined here
> ```
>
> 则修改 linux 文件夹下`scripts/dtc/dtc-lexer.lex.c`，在`YYLTYPE yylloc;`前增加`extern`。再次编译，发现会报错：openssl/bio.h: No such file or directory ，此时执行`sudo apt install libssl-dev`

编译完毕，内核文件位于：arch/arm64/boot/Image。记住整个 linux 文件夹所在的路径，例如：home/korwylee/lgw/hypervisor/linux, 在第 7 步我们还会用到这个路径。

## 四、基于 ubuntu 20.04 arm64 base 构建文件系统

> 本部分的内容可以省略，直接下载该现成的磁盘镜像使用即可。https://blog.syswonder.org/#/2024/20240415_Virtio_devices_tutorial

我们使用 ubuntu 20.04（22.04 也可以）来构建根文件系统。

下载：[ubuntu-base-20.04.5-base-arm64.tar.gz](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz)

链接：[http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz)

```bash
wget http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz

mkdir rootfs
# 创建一个1G大小的ubuntu.img，可以修改count修改img大小
dd if=/dev/zero of=ubuntu-20.04-rootfs_ext4.img bs=1M count=1024 oflag=direct
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
sudo apt-get install git sudo vim bash-completion \
		kmod net-tools iputils-ping resolvconf ntpdate

# 以下由#圈住的内容可做可不做
###################
adduser arm64
adduser arm64 sudo
echo "kernel-5_4" >/etc/hostname
echo "127.0.0.1 localhost" >/etc/hosts
echo "127.0.0.1 kernel-5_4">>/etc/hosts
dpkg-reconfigure resolvconf
dpkg-reconfigure tzdata
###################
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

## 五、Rust 环境配置

请参考：[Rust 语言圣经](https://course.rs/first-try/intro.html)

## 六、编译和运行 hvisor

首先将[hvisor 代码仓库](https://github.com/KouweiLee/hvisor)拉到本地，之后切换到 dev 分支，并在 hvisor/images/aarch64 文件夹下，将之前编译好的根文件系统、Linux 内核镜像分别放在 virtdisk、kernel 目录下，并分别重命名为 rootfs1.ext4、Image。并在 devicetree 目录下，执行`make all`。

之后，在 hvisor 目录下，执行：

```
make ARCH=aarch64 LOG=info FEATURES=platform_qemu run
```

之后会进入 uboot 启动界面，该界面下执行：

```
bootm 0x40400000 - 0x40000000
```

该启动命令会从物理地址`0x40400000`启动 hvisor，设备树的地址为`0x40000000`。hvisor 启动时，会自动启动 root linux（用于管理的 Linux），并进入 root linux 的 shell 界面，root linux 即为 zone0，承担管理工作。

> 提示缺少`dtc`时，，可以执行指令：
>
> ```
> sudo apt install device-tree-compiler
> ```

## 七、使用 hvisor-tool 启动 zone1-linux

首先完成最新版本的 hvisor-tool 的编译。具体请参考[hvisor-tool](https://github.com/syswonder/hvisor-tool)的 README（中文版本是最新版本，英文 README 可能更新不及时）。例如，若要编译面向 arm64 的命令行工具，且 Hvisor 环境中的 Linux 镜像编译来源的源码位于 `KDIR=~/linux`，则可执行

```
make all ARCH=arm64 LOG=LOG_WARN KDIR=~/linux
```

> 请务必保证 Hvisor 中的 Linux 镜像是由编译 hvisor-tool 时参数选项中的 Linux 源码目录编译产生。

编译完成后，将 driver/hvisor.ko、tools/hvisor、driver/ivc.ko（若有该文件）复制到 image/virtdisk/rootfs1.ext4 根文件系统中启动 zone1 linux 的目录（目前是/home/arm64）；再将 zone1 的内核镜像（如果是与 zone0 相同的 Linux，复制一份 image/aarch64/kernel/Image 即可）、设备树（image/aarch64/linux2.dtb）放在 rootfs1.ext4 的相同目录（/home/arm64），并重命名为 Image、linux2.dtb。

最后将 image/aarch64/virtdisk 中的 rootfs1.ext4 原地复制一份，改名为 rootfs2.etx4。

之后在 QEMU 上即可通过 root linux-zone0 启动 zone1-linux。

> 启动 zone1-linux 的详细步骤参看 hvisor-tool 的 README。以下给出一个参考（以 hvisor-tool 为准），其中的 linux2.json 即为 zone1-linux 的配置文件：

```
#在/home/arm64目录下执行：
insmod hvisor.ko
./hvisor zone start linux2.json
```
