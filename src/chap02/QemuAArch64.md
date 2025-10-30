# 在 QEMU 上运行 hvisor

## 一、安装交叉编译器 aarch64-none-linux-gnu-10.3

网址：[https://developer.arm.com/downloads/-/gnu-a](https://developer.arm.com/downloads/-/gnu-a)。

工具选择：AArch64 GNU/Linux target（aarch64-none-linux-gnu）。

下载链接：[https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz?rev=1cb9c51b94f54940bdcccd791451cec3&hash=B380A59EA3DC5FDC0448CA6472BF6B512706F8EC](https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz?rev=1cb9c51b94f54940bdcccd791451cec3&hash=B380A59EA3DC5FDC0448CA6472BF6B512706F8EC)。

```bash
# 下载交叉编译器并解压
wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
tar -xvf gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz

# 查看解压后的可执行文件
ls gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/
```

安装完成，记住路径，例如 `/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-`，之后都会使用这个路径。

## 二、编译安装 QEMU 9.0.1

**注意，QEMU 需要从 7.2.12 换成 9.0.1，以正常使用 PCI 虚拟化。**

```bash
# 安装编译所需的依赖包
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
    gawk build-essential bison flex texinfo gperf libtool patchutils bc \
    zlib1g-dev libexpat-dev pkg-config libglib2.0-dev libpixman-1-dev libsdl2-dev \
    git tmux python3 python3-pip ninja-build

# 下载源码并解压
wget https://download.qemu.org/qemu-9.0.1.tar.xz
tar -xvf qemu-9.0.1.tar.xz

cd qemu-9.0.1
# 生成配置并编译
./configure --enable-kvm --enable-slirp --enable-debug --target-list=aarch64-softmmu,x86_64-softmmu
make -j$(nproc)
```

之后编辑 `~/.bashrc` 文件，在文件的末尾加入几行：

```bash
# 请注意，qemu-9.0.1 的父目录可以随着你的实际安装位置灵活调整。另外需要把其放在 $PATH 变量开头。
export PATH="/path/to/qemu-9.0.1/build:$PATH"
```

随后即可在当前终端 `source ~/.bashrc` 更新系统路径，或者直接重启一个新的终端。此时可以确认 qemu 版本，如果显示为 qemu-9.0.1，则表示安装成功：

```bash
qemu-system-aarch64 --version   # 查看版本
```

> 注意，上述依赖包可能不全，例如：
>
> - 出现 `ERROR: pkg-config binary 'pkg-config' not found` 时，可以安装 `pkg-config` 包；
> - 出现 `ERROR: glib-2.48 gthread-2.0 is required to compile QEMU` 时，可以安装 `libglib2.0-dev` 包；
> - 出现 `ERROR: pixman >= 0.21.8 not present` 时，可以安装 `libpixman-1-dev` 包。

> 若生成设置文件时遇到报错 `ERROR: Dependency "slirp" not found, tried pkgconfig`：
>
> 下载 [https://gitlab.freedesktop.org/slirp/libslirp](https://gitlab.freedesktop.org/slirp/libslirp) 包，并按 README 安装即可。

## 三、编译 Linux Kernel 5.4

**注意，在最后编译 Linux Kernel 前，需修改默认生成的配置文件。需要启用 `CONFIG_BLK_DEV_RAM`，以启用 RAM 块设备支持；需要启用 `CONFIG_IPV6` 和 `CONFIG_BRIDGE`，以支持在 root linux 中创建网桥和 tap 设备。**

交叉编译 Linux Kernel 5.4 生成 root linux 的镜像，用于在 hvisor 中启动 root linux。

```bash
# CROSS_COMPILE 路径需要根据第一步安装交叉编译器的路径进行更改
CROSS_COMPILE_PATH="<路径>/bin"

# 下载 linux 5.4 源码
git clone https://github.com/torvalds/linux -b v5.4 --depth=1
cd linux
git checkout v5.4

# 生成默认的编译配置
CROSS_COMPILE_PREFIX=${CROSS_COMPILE_PATH}/aarch64-none-linux-gnu-
make ARCH=arm64 CROSS_COMPILE=${CROSS_COMPILE_PREFIX} defconfig

# 启用 CONFIG_BLK_DEV_RAM，以启用 RAM 块设备支持
./scripts/config --enable CONFIG_BLK_DEV_RAM
# 启用 CONFIG_IPV6 和 CONFIG_BRIDGE，以支持在 root linux 中创建网桥和 tap 设备
./scripts/config --enable CONFIG_IPV6
./scripts/config --enable CONFIG_BRIDGE

# 编译
make ARCH=arm64 CROSS_COMPILE=${CROSS_COMPILE_PREFIX} Image -j$(nproc)
```

> 如果编译 linux 时报错：
>
> ```
> /usr/bin/ld: scripts/dtc/dtc-parser.tab.o:(.bss+0x20): multiple definition of `yylloc'; scripts/dtc/dtc-lexer.lex.o:(.bss+0x0): first defined here
> ```
>
> 则修改 linux 文件夹下 `scripts/dtc/dtc-lexer.lex.c`，在 `YYLTYPE yylloc;` 前增加 `extern`。再次编译，发现会报错 `openssl/bio.h: No such file or directory`，此时执行 `sudo apt install libssl-dev`。

> 编译过程中会出现：
> ```
> RAM block device support (BLK_DEV_RAM) [Y/n/m/?] y
>   Default number of RAM disks (BLK_DEV_RAM_COUNT) [16] (NEW)
>   Default RAM disk size (kbytes) (BLK_DEV_RAM_SIZE) [4096] (NEW)
> ```
> 即配置具体参数，直接回车采用默认值即可。

编译完毕，内核文件位于 `arch/arm64/boot/Image`。记住整个 linux 文件夹所在的路径，例如 `/home/korwylee/lgw/hypervisor/linux`，在第七步我们还会用到这个路径。

## 四、基于 Ubuntu 22.04 arm64 base 构建文件系统

> 本部分的内容可以省略，直接下载该现成的磁盘镜像使用即可。[https://blog.syswonder.org/#/2024/20240415_Virtio_devices_tutorial](https://blog.syswonder.org/#/2024/20240415_Virtio_devices_tutorial)。

我们使用 Ubuntu 22.04 来构建根文件系统。

> **Ubuntu 20.04** 也可以，但是运行时会报 glibc 版本低的错误，可参考 [ARM64-qemu-jailhouse](https://blog.syswonder.org/#/2023/20230421_ARM64-QEMU-jailhouse) 评论区中的解决办法。

```bash
# QEMU 路径，需要根据第二步安装时的路径进行更改
QEMU_PATH="<路径>/build/qemu-system-aarch64"

# 下载 ubuntu base
wget http://cdimage.ubuntu.com/ubuntu-base/releases/22.04/release/ubuntu-base-22.04.5-base-arm64.tar.gz

# 创建 rootfs，用于挂载后续的 rootfs1.img
mkdir -p rootfs

# 创建一个 1 GiB 大小的 rootfs1.img，可以通过修改 count 修改 img 大小
dd if=/dev/zero of=rootfs1.img bs=1M count=1024 oflag=direct
# 格式化为 ext4 文件系统
mkfs.ext4 rootfs1.img

# 挂载 rootfs1.img
sudo mount -t ext4 rootfs1.img rootfs/
# 将 ubuntu.tar.gz 的内容解压到 rootfs
sudo tar -xzf ubuntu-base-22.04.5-base-arm64.tar.gz -C rootfs/

# 让 rootfs 绑定和获取物理机的一些信息和硬件
sudo cp "${QEMU_PATH}" rootfs/usr/bin/
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
sudo mount -t proc /proc rootfs/proc
sudo mount -t sysfs /sys rootfs/sys
sudo mount -o bind /dev rootfs/dev
sudo mount -o bind /dev/pts rootfs/dev/pts

# 将文件系统切换到 rootfs
sudo chroot rootfs  # 执行该指令可能会报错，请参考下面的解决办法
# 在 rootfs 中安装必要的软件包
apt-get update
apt-get install git sudo vim bash-completion \
    kmod net-tools iputils-ping resolvconf ntpdate screen
apt-get clean

# 以下由 # 圈住的内容可做可不做
###################
adduser arm64
adduser arm64 sudo
echo "kernel-5_4" > /etc/hostname
echo "127.0.0.1 localhost" > /etc/hosts
echo "127.0.0.1 kernel-5_4" >> /etc/hosts
dpkg-reconfigure resolvconf
dpkg-reconfigure tzdata
###################

# 退出 rootfs
exit

# 卸载 rootfs
sudo umount rootfs/proc
sudo umount rootfs/sys
sudo umount rootfs/dev/pts
sudo umount rootfs/dev
sudo umount rootfs
```

> 此时可以顺便创建后续要用到的 `rootfs2.img`，其大小应适当减少，以便放入 `rootfs1.img` 中。
> ```bash
> # QEMU 路径，需要根据第二步安装时的路径进行更改
> QEMU_PATH="<路径>/build/qemu-system-aarch64"
>
> # 创建 rootfs2.img，其大小适当减少到 256 MiB
> dd if=/dev/zero of=rootfs2.img bs=1M count=256 oflag=direct
> mkfs.ext4 rootfs2.img
> sudo mount -t ext4 rootfs2.img rootfs/
> sudo tar -xzf ubuntu-base-22.04.5-base-arm64.tar.gz -C rootfs/
> sudo cp "${QEMU_PATH}" rootfs/usr/bin/
> sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
> sudo mount -t proc /proc rootfs/proc
> sudo mount -t sysfs /sys rootfs/sys
> sudo mount -o bind /dev rootfs/dev
> sudo mount -o bind /dev/pts rootfs/dev/pts
> sudo umount rootfs/proc
> sudo umount rootfs/sys
> sudo umount rootfs/dev/pts
> sudo umount rootfs/dev
> sudo umount rootfs
> ```

最后卸载挂载，完成根文件系统的制作。

> 执行 `sudo chroot rootfs` 时，如果报错 `chroot: failed to run command '/bin/bash': Exec format error`，可以执行指令：
>
> ```bash
> sudo apt-get install qemu-user-static
> sudo update-binfmts --enable qemu-aarch64
> ```

## 五、Rust 环境配置

请参考 [Rust 语言圣经](https://course.rs/first-try/intro.html)。

## 六、编译和运行 hvisor

首先将 [hvisor 代码仓库](https://github.com/syswonder/hvisor) 拉到本地，并切换到 dev 分支。

运行前需要准备好 hvisor 的平台配置文件，具体准备工作包括根文件系统、Linux 内核镜像以及编译对应的设备树文件，现 hvisor 各平台配置文件在仓库的 `platform/<架构>/<平台名>/` 路径下，例如本教程将采用的配置文件位于 `platform/aarch64/qemu-gicv3/` 路径下。

```bash
# 复制根文件系统 rootfs1.ext4
ROOTFS1_PATH="<路径>/rootfs1.img"
mkdir -p platform/aarch64/qemu-gicv3/image/virtdisk
cp "${ROOTFS1_PATH}" platform/aarch64/qemu-gicv3/image/virtdisk/rootfs1.ext4
```

```bash
# 复制 Linux 内核镜像
KERNEL_PATH="<路径>/Image"
mkdir -p platform/aarch64/qemu-gicv3/image/kernel
cp "${KERNEL_PATH}" platform/aarch64/qemu-gicv3/image/kernel/Image
```

```bash
# 编译设备树
make BID=aarch64/qemu-gicv3 dtb
```

> 其实建议采用硬链接的方式，以便减轻磁盘空间的占用和方便根文件系统修改时同步。

之后在 hvisor 目录下，执行相应命令即可启动 hvisor。

```bash
make BID=aarch64/qemu-gicv3 LOG=info run
```

执行后会进入 uboot 启动界面，该界面下执行：

```
bootm 0x40400000 - 0x40000000
```

该启动命令会从物理地址 `0x40400000` 启动 hvisor，`0x40000000` 本质上已无用，但因历史原因仍然保留。hvisor 启动时，会自动启动 root linux（用于管理的 Linux），并进入 root linux 的 shell 界面，root linux 即为 zone0，承担管理工作。

> 提示缺少 `dtc` 时，可以执行指令：
>
> ```bash
> sudo apt install device-tree-compiler
> ```

## 七、使用 hvisor-tool 启动 zone1-linux

首先完成最新版本的 hvisor-tool 的编译。具体请参考 [hvisor-tool](https://github.com/syswonder/hvisor-tool) 的 README。

```bash
# Linux 源代码路径，需要根据第三步安装时的路径进行更改
LINUX_PATH="<路径>/linux"

git clone https://github.com/syswonder/hvisor-tool.git
cd hvisor-tool
make all ARCH=arm64 LOG=LOG_INFO KDIR="${LINUX_PATH}"
```

> 请务必保证 hvisor 中的 root linux 镜像是由编译 hvisor-tool 时参数选项中的 Linux 源代码目录编译产生。

> 请务必保证 hvisor-tool 编译时采用的 linux header 版本与 root linux 的 linux header 版本一致，否则 hvisor-tool 的 driver 可能会无法加载。
> 可以通过使用与第三步中的 root linux 相同的交叉编译工具链进行编译，即使用第一步的交叉编译器路径进行配置。

编译完成后，需要将 hvisor-tool 的可执行文件 `tools/hvisor` 和内核模块 `driver/hvisor.ko` 复制到 root linux 的根文件系统中启动 zone1 linux 的目录，例如 `/root`，再同时将 zone1 的根文件系统、内核镜像、以及编译后的设备树放在同一目录。

具体的文件名需要与 hvisor-tool 配置文件（来自 hvisor 的 `platform/aarch64/qemu-gicv3/configs/zone1-linux-virtio.json` 和 `platform/aarch64/qemu-gicv3/configs/zone1-linux.json`）的内容保持一致。

按照 hvisor 提供的配置文件，可执行命令如下。

```bash
# 回到创建的 root linux 根文件系统时的目录
LINUX_PATH="<路径>/linux"
HVISOR_PATH="<路径>/hvisor"
HVISOR_TOOL_PATH="<路径>/hvisor-tool"

# 挂载
sudo mount -t ext4 rootfs1.img rootfs/

# 复制 hvisor-tool 的 driver/hvisor.ko 和 tools/hvisor
sudo cp "${HVISOR_TOOL_PATH}/driver/hvisor.ko" rootfs/root/
sudo cp "${HVISOR_TOOL_PATH}/tools/hvisor" rootfs/root/

# 复制 hvisor-tool 的配置文件到 root 路径下
sudo cp "${HVISOR_PATH}/platform/aarch64/qemu-gicv3/configs/zone1-linux-virtio.json" \
    rootfs/root/zone1-linux-virtio.json
sudo cp "${HVISOR_PATH}/platform/aarch64/qemu-gicv3/configs/zone1-linux.json" \
    rootfs/root/zone1-linux.json

# 复制 zone1 linux 的根文件系统、内核镜像、以及编译后的设备树
sudo cp rootfs2.img \
    rootfs/root/rootfs2.ext4
sudo cp "${LINUX_PATH}/arch/arm64/boot/Image" \
    rootfs/root/Image
sudo cp "${HVISOR_PATH}/platform/aarch64/qemu-gicv3/image/dts/zone1-linux.dtb" \
    rootfs/root/zone1-linux.dtb

# 卸载
sudo umount rootfs

# 如果之前是复制的 rootfs1.img，则还需重新复制一份，命令如下
# 切换到 hvisor 目录
# ROOTFS1_PATH="<路径>/rootfs1.img"
# mkdir -p platform/aarch64/qemu-gicv3/image/virtdisk
# cp "${ROOTFS1_PATH}" platform/aarch64/qemu-gicv3/image/virtdisk/rootfs1.ext4
```

> 如果遇到 `rootfs1.ext4` 容量不够，则可以参考 [img 扩容](https://blog.syswonder.org/#/2023/20230421_ARM64-QEMU-jailhouse?id=_2-img%e6%89%a9%e5%ae%b9) 为 `rootfs1.ext4` 扩容。

之后在 QEMU 上即可通过 root linux 启动 zone1-linux。具体命令如下。

```bash
# 启动 QEMU
make BID=aarch64/qemu-gicv3 LOG=info run
```

```
# 启动 root linux
bootm 0x40400000 - 0x40000000
```

```bash
cd root
insmod hvisor.ko
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts

rm nohup.out

# 启动 zone1-linux 的 virtio
nohup ./hvisor virtio start zone1-linux-virtio.json &
# 启动 zone1-linux
./hvisor zone start zone1-linux.json && \
cat nohup.out | grep "char device" && \
script /dev/null
```

> 启动 zone1-linux 的详细步骤参看 hvisor-tool 的 README 以及 [启动示例](https://github.com/syswonder/hvisor-tool/tree/main/examples/qemu-aarch64/with_virtio_blk_console/README.md)。

> 如果显示 virtio 出现 WARNING 或者 ERROR，可以查看 `nohup.out` 查看详细信息，或者使用 `dmesg` 命令查看内核日志。
